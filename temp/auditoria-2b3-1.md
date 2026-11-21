## Especificação Técnica do Contrato da API Central de Jogos (GET /api/games/central)

> **Escopo desta etapa:** auditoria somente leitura e definição formal de contrato. Nenhum arquivo do repositório foi criado ou alterado.

---

## ENTREGÁVEL A — Diagnóstico do Estado Atual

### 1. Mapeamento das estruturas existentes no agregador Shadow (GET /api/games/shadow)

Atualmente, GET /api/games/shadow (definido em src/server/registerCollectorRoutes.ts) retorna diretamente o objeto UnifiedGamesShadowResult (definido em src/server/collectors/unifiedGamesCollector.ts) acrescido de formattedDate:

- **Envelope atual (UnifiedGamesShadowResult):**
    
    - ok: boolean, shadowMode: true, referenceDateBrt, yesterdayBrt, tomorrowBrt, generatedAt, fromCache, degraded, error?, formattedDate.
        
    - counts: { today, overnight, tomorrow, excluded, totalCanonical, reconciledAmerican, divergencesTotal }.
        
    - providerStatus: { bzzoiro, espn, sportsblaze }, cada qual com ProviderExecutionReport (ok, partial, totalCalls, succeededCalls, failedCalls, emptySlateCalls, rawItemCount, dedupedItemCount, errors[]).
        
    - sets: { today: UnifiedGameMatch[], overnight: UnifiedGameMatch[], tomorrow: UnifiedGameMatch[] }.
        
    - excluded: lista diagnóstica de partidas descartadas por partitionGamesIntoSets() com reason, startDateBrt, endDateBrt, hasReliableEndTimestamp.
        
    - reconciliationSummary: { matched, espnOnly, sportsblazeOnly, bzzoiroOnly, divergencesCount }.
        
- **Objeto atual da partida (UnifiedGameMatch):**
    
    - Estende ClassifiableMatch (src/utils/gameSets.ts) com campos planos na raiz (id, rawDate, timestampSource, status, isLive, isPostponed, completed, endDate, endTime, finishedAt, completedAt).
        
    - Identificadores na raiz: id (canonicalId), canonicalId, espnId?, sportsblazeId?, bzzoiroId?, externalIds.
        
    - Classificação e proveniência de alto nível: sport, sportFamily, league, providers, reconciled, reconciliationMode, reconciliationDeltaMinutes?, set.
        
    - Subobjetos: homeTeam, awayTeam, fixture (UnifiedFixtureGrade), liveOverlay (UnifiedLiveOverlay), heroCompat (UnifiedHeroCompat), fieldProvenance (13 chaves fixas), divergences[] e providerSnapshots ({ bzzoiro?, espn?, sportsblaze? }).
        

### 2. Campos obrigatórios, opcionais, nullables e redundantes

|   |   |
|---|---|
|Categoria|Diagnóstico no estado atual|
|**Obrigatórios na raiz/subobjetos**|id, canonicalId, sport, sportFamily, league, providers, reconciled, reconciliationMode, set, rawDate, timestampSource, status, isLive, isPostponed, completed, homeTeam, awayTeam, fixture, liveOverlay, heroCompat, fieldProvenance, divergences, providerSnapshots.|
|**Nullables intencionais**|homeTeam.score, awayTeam.score, liveOverlay.score.{home,away}, heroCompat.teamA.score, heroCompat.teamB.score (null em UPCOMING e POSTPONED); fixture.startDateBrt (string \| null); heroCompat.statsMatchId (null em partidas sportsblaze_only).|
|**Opcionais**|espnId, sportsblazeId, bzzoiroId, reconciliationDeltaMinutes, endDate, endTime, finishedAt, completedAt, fixture.{leagueId,country,leagueFlag,leagueLogo,season}, liveOverlay.{period,clock,tempo,liveState,incidents,teamStatistics,players,updated}, liveOverlay.linescores.periodScores.|
|**Redundâncias críticas (4 cópias)**|Os mesmos dados de times, placares, datas, estádio e transmissão são repetidos **4 vezes por partida**: (1) raiz (homeTeam/awayTeam/status/rawDate), (2) fixture, (3) liveOverlay, e (4) heroCompat (teamA/teamB/linescoresA/linescoresB).|

### 3. Diferenças por modalidade e provedor

- **Futebol (bzzoiroCollector.ts):**
    
    - sport: "Futebol", sportFamily: "football".
        
    - Possui metadados ricos em fixture (leagueId, country, leagueFlag, leagueLogo) e liveOverlay.incidents (homeGoals, awayGoals).
        
    - liveOverlay.linescores vem sempre com arrays vazios ([]).
        
    - period e liveState não são preenchidos (not_applicable_to_football). clock na proveniência baseia-se em m.tempo (ex.: "67'", "FIM").
        
    - Estatísticas detalhadas do jogo **não vêm na coleta diária** — são carregadas sob demanda via /api/match-stats (/api/v2/events/{id}/stats/).
        
- **Esportes Americanos (espnAmericanCollector.ts + sportsblazeAmericanCollector.ts):**
    
    - sport = sigla da liga ("NFL", "NCAAF", "MLB", "NHL", "NBA", "WNBA"), sportFamily: "american".
        
    - ESPN fornece espnLiveDetail (fonte autoritativa para period, clock, shortDetail, situation — corrigido na Etapa 2B.3.2), linescores, venue, broadcaster e logos.
        
    - SportsBlaze (disponível para NFL, MLB, NBA, WNBA) fornece periodScores, teamStatistics, players (boxscore individual) e season (year, type, week). Em jogos sportsblaze_only (fallback), venue, broadcaster e logos vêm vazios e statsMatchId é null.
        
- **Lacrosse (espnLacrosseCollector.ts):**
    
    - sport: "LACROSSE", sportFamily: "lacrosse".
        
    - Coleta leve via scoreboard ESPN: sem period, sem clock, sem liveState, sem linescores e sem boxscores de atletas. externalIds.lacrosseCollectorId preserva o prefixo lax-espn-<id>.
        

### 4. Dados internos que NÃO devem vazar na API pública

1. **providerSnapshots:** clona payloads completos dos provedores (incluindo listas inteiras de atletas e objetos crus), inflando drasticamente o JSON de resposta (centenas de KB/MB extras por requisição) e acoplando consumidores a formatos internos de terceiros.
    
2. **excluded[] (diagnóstico de partição):** útil em /api/games/shadow para auditoria da Etapa 2A/2B.3, mas ruído interno para consumidores de produção.
    
3. **liveOverlay.updated vindo de sb.updated:** no coletor SportsBlaze, updated é um timestamp do feed/cache da resposta inteira, **não** do evento individual (como já protegido por parseComparableEventTimestampMs em unifiedGamesCollector.ts:914). Não deve ser exposto como atualização de partida.
    
4. **Chaves internas / detalhes operacionais:** apiKey, URLs internas de upstream, stack traces, e shadowMode: true.
    
5. **Status em português inconsistente dentro do núcleo canônico:** heroCompat.status usa "ADIADO" enquanto o núcleo usa "POSTPONED"; no contrato canônico, o status deve ser estritamente "LIVE" | "FINAL" | "UPCOMING" | "POSTPONED".
    

### 5. Incompatibilidades e pontos de atenção nos consumidores atuais

- **/api/match-stats (server.ts:1708 e 1993):**
    
    - Para esportes americanos, o backend espera um ID numérico da ESPN (aceita sb-<league>-<espnId> removendo o prefixo e extraindo \d+ para consultar site.api.espn.com/.../summary?event=${rawEspnId}). **IDs UUID da SportsBlaze quebram essa busca.**
        
    - Para futebol, o backend remove bsd- e extrai \d+ para consultar sports.bzzoiro.com/api/v2/events/${rawBsdId}/stats/.
        
- **sportsblazeService.ts:98:** realiza (game.period || game.time || "").replace(...). Se um consumidor legado receber period numérico (ex.: 2), .replace lança TypeError. Por isso, period numérico deve ficar isolado no contrato canônico novo, sem sobrescrever contratos legados durante a transição.
    
- **Ticker.tsx:272-278 e brtDate.ts:679-684:** inferem timestampSource pelo prefixo do id (bsd- -> "bzzoiro", espn- -> "espn", sb- -> "sportsblaze"). Contudo, jogos sb-<league>-<espnId> do fluxo atual têm timestamps da ESPN. O contrato central deve expor timestampSource explícito e datas ISO com offset/UTC (Z) sem depender de heurística por prefixo.
    
- **HomeLite.tsx e /api/jogos-texto:** esperam lista enxuta com hora, status em PT ("AO VIVO" | "AGENDADO" | "ENCERRADO" | "ADIADO"), mandante, visitante, placarMandante, placarVisitante, transmissao.
    
- **MeusTimesPage.tsx e favoritesService.ts:** consomem hoje /api/football/today + endpoints de agenda de times individuais; o agregador central cobre os jogos da janela yesterday..today..tomorrow, mas **não** substitui buscas de calendário futuro de longo prazo por time específico.
    
- **HeroCommandCenter.tsx + HeroCopa.tsx:** consomem TickerItem[], teamA/teamB, linescoresA/linescoresB, homeKeyPlayers/awayKeyPlayers e statsMatchId.
    

---

## ENTREGÁVEL B — Proposta Completa do Contrato /api/games/central

### 1. Endpoint, Métodos e Escopo

- **Rota:** GET /api/games/central e HEAD /api/games/central
    
- **Métodos não permitidos:** POST, PUT, PATCH, DELETE -> HTTP 405 Method Not Allowed com header Allow: GET, HEAD.
    
- **Somente leitura e sem chamadas extras na filtragem:** toda requisição à /api/games/central lê a coleção gerada pelo agregador central (com cache de 60s por referenceDate e in-flight coalescing) e aplica filtros, projeção e ordenação **exclusivamente em memória** sobre o snapshot agregado. Nenhum filtro pode disparar chamadas adicionais aos provedores externos.
    

---

### 2. Parâmetros de Consulta (Query Parameters)

|   |   |   |   |
|---|---|---|---|
|Parâmetro|Tipo / Valores aceitos|Default|Regra e validação|
|date|YYYY-MM-DD (BRT, America/Sao_Paulo)|Data atual em BRT (getBrtTodayStr())|Validado via normalizeBrtDateIso. Qualquer formato inválido ou data inexistente (ex.: 2026-02-30, abc) retorna **HTTP 400** com error.code = "INVALID_DATE".|
|sets|Lista separada por vírgula entre: today, overnight, tomorrow (case-insensitive) ou all|today,overnight,tomorrow|all equivale a today,overnight,tomorrow. Valores desconhecidos retornam **HTTP 400** (INVALID_SETS). Conjuntos não solicitados continuam contabilizados em counts.totalAvailable ou retornam arrays omitidos/vazios? Regra: chaves não solicitadas em sets retornam array vazio [] para manter o shape tipado fixo, e meta.appliedFilters.sets informa quais foram incluídos.|
|sport|Lista separada por vírgula. Valores canônicos: football, basketball, american_football, baseball, hockey, lacrosse. **Aliases aceitos:** futebol->football, nfl/ncaaf->american_football, nba/wnba->basketball, mlb->baseball, nhl->hockey, pll->lacrosse.|Omitido (todos)|Normalizado para o domínio canônico de modalidade (football, basketball, american_football, baseball, hockey, lacrosse). Valor não reconhecido -> **HTTP 400** (INVALID_SPORT).|
|league|Lista separada por vírgula (case-insensitive). Para americanos/lacrosse: nfl, ncaaf, mlb, nhl, nba, wnba, pll. Para futebol: leagueId numérico (ex.: 9, 2, 4) ou slug/nome normalizado (ex.: brasileirao-serie-a, premier-league, champions-league).|Omitido (todas)|Suporta múltiplos valores (?league=nba,nfl). Valores que não correspondam a nenhuma liga ativa não geram erro se sintaticamente válidos (retornam listas vazias [] com ok: true); apenas strings vazias/malformadas ou > 20 itens retornam **HTTP 400** (INVALID_LEAGUE).|
|status|Lista separada por vírgula entre: LIVE, UPCOMING, FINAL, POSTPONED (case-insensitive). **Aliases:** AO VIVO->LIVE, AGENDADO->UPCOMING, ENCERRADO->FINAL, ADIADO/PP->POSTPONED.|Omitido (todos)|Valor fora do enum/aliases -> **HTTP 400** (INVALID_STATUS).|
|liveOnly|true \| false \| 1 \| 0|false|Se true/1, equivale a restringir a partida a status = "LIVE". Se combinado com ?status=FINAL (conflito lógico), retorna sets vazios [] (interseção vazia válida, 200 OK) ou se status também incluir LIVE, prevalece a interseção (LIVE). Valor inválido (ex.: liveOnly=yes) -> **HTTP 400**.|
|includeProvenance|true \| false \| 1 \| 0|false|Quando true, inclui o bloco provenance em cada partida (reconciliationMode, reconciliationDeltaMinutes, fieldProvenance e divergences). **Nunca** inclui providerSnapshots.|
|includeStats|none \| summary \| full|summary|**none**: omite stats.teamStatistics e stats.players, mantendo apenas stats.ref (se há boxscore/stats no /api/match-stats). **summary**: inclui apenas dados leves já em memória (até 5 keyPlayers por equipe quando vindos do SportsBlaze e incidents.homeGoals/awayGoals no futebol; não inclui arrays completos de todos os atletas nem teamStatistics brutos). **full**: inclui teamStatistics e lista completa de players quando já presentes no snapshot do coletor (zero fetches extras). Valor inválido -> **HTTP 400**.|
|includeCompat|true \| false \| 1 \| 0|false|Controla inclusão de compat (campos legados mínimos para transição de consumidores antigos como HeroCopa/Ticker). Quando false (padrão), mantém o payload canônico limpo.|

**Regras de combinação, limites e ordenação:**

- **Combinação lógica:** OR dentro de um mesmo parâmetro multivalorado (ex.: league=nba,nhl = NBA ou NHL) e AND entre parâmetros diferentes (ex.: sport=basketball&status=LIVE = basquete E ao vivo).
    
- **Limites de proteção:** máximo de **20 valores** por filtro lista (sport, league, status), tamanho máximo de cada token = **64 caracteres**, rejeição de caracteres de controle; violação retorna **HTTP 400**.
    
- **Ordenação determinística dentro de cada set (today, overnight, tomorrow):** LIVE -> UPCOMING -> FINAL -> POSTPONED, desempate por fixture.startTimeUtc (ou fixture.startTimeBrt) crescente e depois canonicalId alfabético.
    

---

### 3. Envelope da Resposta e Distinção de Timestamps

#### Semântica rigorosa de timestamps (requisito crítico)

1. **generatedAt (string ISO UTC):** instante em que o snapshot agregado desta resposta foi construído em memória pelo servidor (ex.: "2026-11-21T01:35:00.000Z"). Reflete o ciclo de geração do agregador (cache de 60s).
    
2. **providerUpdatedAt (Record<ProviderId, string | null>):** instante de atualização do feed/coleta reportado pelo provedor ou conhecido pelo coletor (ex.: sb.updated do SportsBlaze, ou null quando o provedor como Bzzoiro/ESPN scoreboard não fornece timestamp explícito de feed). **Não significa horário da partida nem instante do último lance de um jogo específico.**
    
3. **eventUpdatedAt (string | null, por partida em liveOverlay.eventUpdatedAt):** timestamp explícito da última mutação conhecida **daquela partida específica** (ex.: liveState.updatedAt ou espnLiveDetail.updatedAt quando disponível). **Regra inviolável:** se o provedor não fornecer timestamp por evento (como Bzzoiro ou feed agregado sem timestamp por jogo), eventUpdatedAt deve ser null — **nunca** promover generatedAt, sb.updated ou startTime para eventUpdatedAt ou completedAt.
    

#### Decisão sobre campos adicionais no envelope

- **snapshotId (INCLUÍDO):** identificador determinístico da geração em cache do agregador (ex.: "ug-2026-11-20-1795264500000"). Justificativa: permite correlação imediata entre requisições cliente, validação de ETag e diagnóstico de cache sem depender de logs internos.
    
- **cacheAgeMs (INCLUÍDO em meta.cache.ageMs):** idade em milissegundos desde generatedAt até o momento da resposta HTTP (0 quando gerado na hora). Justificativa: essencial para UI (ex.: badge de frescor no Centro de Comando) e para o Cache-Control dinâmico respeitar o TTL máximo de 60s.
    
- **dataAsOf (OMITIDO no envelope raiz, substituído por generatedAt + providerStatus[p].feedUpdatedAt):** Justificativa: um único campo dataAsOf para 3 provedores com TTLs diferentes (Bzzoiro 180s, ESPN 60s, SportsBlaze 60s/300s) é ambíguo e enganoso. Expor generatedAt no envelope e providerUpdatedAt por provedor em providerStatus é tecnicamente exato.
    
- **warnings (INCLUÍDO como string[]):** lista legível e estável de alertas não-fatais (ex.: "sportsblaze_partial_failure", "bzzoiro_empty_slate_2026-11-21"). Facilita decisão de banner discreto na UI sem precisar inspecionar arrays de erro de cada provedor.
    
- **errors (INCLUÍDO na raiz como ApiErrorItem[] apenas quando há erro fatal ou parcial relevante, vazio [] quando tudo ok):** padroniza tratamento no cliente (code, message, provider?, scope?, statusCode?).
    

#### Interface TypeScript do Envelope

Ts

```
export type CentralContractVersion = "1.0";

export interface CentralGamesResponse {
  contractVersion: CentralContractVersion;
  ok: boolean;
  degraded: boolean;
  snapshotId: string;
  generatedAt: string; // ISO-8601 UTC da construção do snapshot no agregador
  referenceDate: string; // YYYY-MM-DD em America/Sao_Paulo
  timezone: "America/Sao_Paulo";
  window: {
    yesterday: string; // YYYY-MM-DD
    today: string;     // YYYY-MM-DD
    tomorrow: string;  // YYYY-MM-DD
  };
  cache: {
    hit: boolean;
    ageMs: number;
    ttlMs: number; // 60000
  };
  filters: {
    sets: Array<"today" | "overnight" | "tomorrow">;
    sport: CanonicalSport[] | null;
    league: string[] | null;
    status: UnifiedNormalizedStatus[] | null;
    liveOnly: boolean;
    includeProvenance: boolean;
    includeStats: "none" | "summary" | "full";
    includeCompat: boolean;
  };
  counts: {
    returned: {
      today: number;
      overnight: number;
      tomorrow: number;
      total: number;
    };
    snapshotTotal: {
      today: number;
      overnight: number;
      tomorrow: number;
      total: number;
    };
    live: {
      returned: number;
      snapshotTotal: number;
    };
  };
  providerStatus: Record<ProviderId, {
    ok: boolean;
    partial: boolean;
    feedUpdatedAt: string | null;
    totalCalls: number;
    succeededCalls: number;
    failedCalls: number;
    emptySlateCalls: number;
    dedupedItemCount: number;
  }>;
  warnings: string[];
  errors: Array<{
    code: string;
    message: string;
    provider?: ProviderId;
    scope?: string;
    statusCode?: number;
  }>;
  sets: {
    today: CentralGameMatch[];
    overnight: CentralGameMatch[];
    tomorrow: CentralGameMatch[];
  };
}
```

> **Sobre os conjuntos (sets):** a definição de pertencimento a today, overnight e tomorrow obedece **estritamente** a partitionGamesIntoSets() em src/utils/gameSets.ts e src/utils/brtDate.ts, sem qualquer alteração nas regras da Etapa 2A.

---

### 4. Schema Canônico da Partida (CentralGameMatch)

O schema elimina a quadruplicação de dados de UnifiedGameMatch e separa claramente **Identidade**, **Grade Fixa (fixture)**, **Estado Dinâmico (liveOverlay)**, **Fronteira de Estatísticas (stats)**, **Proveniência opcional (provenance)** e **Compatibilidade transitória (compat)**.

Ts

```
export type CanonicalSport =
  | "football"
  | "basketball"
  | "american_football"
  | "baseball"
  | "hockey"
  | "lacrosse";

export type CanonicalSportFamily = "football" | "american" | "lacrosse";

export interface CentralTeamFixture {
  id: string | null; // ID interno do provedor quando existir (ex.: sb.homeTeam.id), ou null
  name: string;
  abbreviation: string;
  logo: string;
}

export interface CentralGameMatch {
  // 1. IDENTIDADE E CLASSIFICAÇÃO
  canonicalId: string; // Ex.: "ug:nba:espn:401800001", "ug:football:bzzoiro:888001"
  set: "TODAY" | "OVERNIGHT" | "TOMORROW";
  sport: CanonicalSport;
  sportFamily: CanonicalSportFamily;
  league: {
    code: string;        // Ex.: "NBA", "NFL", "PLL", ou slug/nome em futebol ("Premier League")
    name: string;        // Nome exibível ("NBA", "Brasileirão Série A", "Premier League")
    id: number | null;   // ID numérico da liga no Bzzoiro (futebol) ou null
    country: string | null;
    flag: string | null;
    logo: string | null;
  };
  providerIds: {
    espnId: string | null;
    sportsblazeId: string | null;
    bzzoiroId: string | null;
    lacrosseCollectorId: string | null;
  };
  providers: ProviderId[];
  reconciled: boolean;
  reconciliationMode: "matched" | "espn_only" | "sportsblaze_only" | "bzzoiro_only";

  // 2. GRADE DA PARTIDA (FIXTURE)
  fixture: {
    startTimeUtc: string | null; // ISO-8601 UTC normalizado quando parseável; null se ausente/inválido
    rawDate: string;             // String original do provedor
    timestampSource: "bzzoiro" | "espn" | "sportsblaze";
    startDateBrt: string | null; // YYYY-MM-DD em America/Sao_Paulo
    startTimeBrt: string;        // HH:mm em BRT (ou "" se indisponível)
    dateDisplay: string;         // DD/MM/YYYY em BRT
    venue: string;
    broadcaster: string;
    season: {
      year: number | null;
      type: string | null;
      week: number | null;
    } | null;
    endTime: {
      endDate: string | null;
      endTime: string | null;
      finishedAt: string | null;
      completedAt: string | null;
    };
    homeTeam: CentralTeamFixture;
    awayTeam: CentralTeamFixture;
  };

  // 3. OVERLAY AO VIVO (ESTADO DINÂMICO)
  liveOverlay: {
    status: "LIVE" | "FINAL" | "UPCOMING" | "POSTPONED";
    live: boolean;
    score: {
      home: number | null; // Sempre null em UPCOMING e POSTPONED
      away: number | null; // Sempre null em UPCOMING e POSTPONED
    };
    timeBadge: string;     // Ex.: "2:19 - 2nd", "67'", "FIM", "ADIADO", "21:30"
    period: number | null; // Inteiro 1..25 ou null (nunca string)
    clock: string | null;  // Ex.: "2:19", "14.2", ou null durante intervalo/pré-jogo/fim/futebol sem relógio
    tempo: string;         // Rótulo textual curto ("67'", "2:19 - 2nd", "Fim", "")
    liveState: {
      down?: number;
      distance?: number;
      possession?: string;
      yard_line?: number;
      timeouts?: { away?: number; home?: number };
    } | null;
    linescores: {
      header: string[];
      home: Array<string | number>;
      away: Array<string | number>;
    };
    incidents: {
      homeGoals: string[];
      awayGoals: string[];
    } | null; // Preenchido em futebol; null nas demais modalidades
    eventUpdatedAt: string | null; // Timestamp do evento se o provedor fornecer explicitamente; nunca feed `updated`
  };

  // 4. FRONTEIRA DE ESTATÍSTICAS
  stats: {
    available: boolean;          // true quando há endpoint `/api/match-stats` compatível ou boxscore embutido
    matchStatsId: string | null; // ID compatível com `/api/match-stats` (ex.: "sb-nba-401800001", "bsd-888001"); null se SportsBlaze-only
    endpoint: string | null;     // Ex.: "/api/match-stats?id=sb-nba-401800001&sport=NBA&league=NBA"
    hasTeamStats: boolean;
    hasPlayerStats: boolean;
    // Presentes conforme `includeStats`:
    // - "none": omitidos
    // - "summary": keyPlayers (até 5) quando já em memória; sem teamStatistics/players completos
    // - "full": teamStatistics e players completos quando já em memória
    keyPlayers?: {
      home: Array<{ name: string; pos?: string; stat?: string }>;
      away: Array<{ name: string; pos?: string; stat?: string }>;
    };
    teamStatistics?: {
      away?: Record<string, unknown>;
      home?: Record<string, unknown>;
    } | null;
    players?: {
      away?: SportsBlazePlayerBoxscore[];
      home?: SportsBlazePlayerBoxscore[];
    } | null;
  };

  // 5. PROVENIÊNCIA (incluído apenas quando `?includeProvenance=true`)
  provenance?: {
    reconciliationDeltaMinutes: number | null;
    rawStatusByProvider: Partial<Record<ProviderId, string>>;
    fieldProvenance: Record<ProvenanceFieldKey, FieldProvenanceEntry>;
    divergences: ReconciliationDivergence[];
  };

  // 6. COMPATIBILIDADE TRANSITÓRIA (incluído apenas quando `?includeCompat=true`)
  compat?: {
    legacyId: string;            // Ex.: "bsd-888001", "sb-nba-401800001", "lacrosse-4018..."
    tickerId: string;
    statsMatchId: string | null;
    apiId: string | null;
    heroStatus: "LIVE" | "FINAL" | "UPCOMING" | "ADIADO";
  };
}
```

#### Mapeamento entre sport do Shadow e sport canônico no Central

|   |   |   |   |
|---|---|---|---|
|Origem no unifiedGamesCollector.ts|sport canônico|sportFamily|league.code|
|"Futebol"|"football"|"football"|Nome/slug da liga (ex.: "Premier League")|
|"NBA" / "WNBA"|"basketball"|"american"|"NBA" / "WNBA"|
|"NFL" / "NCAAF"|"american_football"|"american"|"NFL" / "NCAAF"|
|"MLB"|"baseball"|"american"|"MLB"|
|"NHL"|"hockey"|"american"|"NHL"|
|"LACROSSE"|"lacrosse"|"lacrosse"|"PLL" (ou nome completo em league.name)|

---

### 5. Regras de Status HTTP, Erros e Falha Parcial vs. Total

|   |   |   |   |   |
|---|---|---|---|---|
|Cenário|HTTP Status|ok|degraded|Comportamento|
|Sucesso total (todos os provedores responderam; slate vazio é válido)|200 OK|true|false|Retorna sets preenchidos (podem ser [] se não houver jogos no dia, ex.: off-season ou 404 de boxscore futuro da SportsBlaze).|
|**Falha parcial de provedor** (ex.: Bzzoiro ou SportsBlaze falhou, mas ESPN ou outro provedor respondeu)|200 OK|true|true|Retorna jogos disponíveis dos provedores saudáveis, providerStatus[x].ok = false (ou partial = true), warnings com código legível (ex.: "provider_partial:sportsblaze") e errors com resumo seguro.|
|**Falha total dos provedores** (anyProviderSucceeded === false)|503 Service Unavailable|false|true|Não armazena resultado em cache; retorna envelope padrão com sets vazios, providerStatus detalhando falhas e errors: [{ code: "ALL_PROVIDERS_FAILED", message: "..." }].|
|**Parâmetro de query inválido** (date, sets, sport, status, liveOnly, includeStats, etc.)|400 Bad Request|false|false|Retorna JSON estruturado { contractVersion: "1.0", ok: false, error: { code, message, param } }.|
|**ETag correspondente em If-None-Match**|304 Not Modified|—|—|Sem corpo; envia ETag, Cache-Control e Vary: Accept-Encoding.|
|**Método diferente de GET/HEAD**|405 Method Not Allowed|false|false|Header Allow: GET, HEAD.|

---

## ENTREGÁVEL C — Exemplos de Payload JSON Realistas

> **Nota de fidelidade:** todos os exemplos abaixo refletem exclusivamente campos, formatos de IDs, status e regras realmente produzidos pelos coletores auditados (bzzoiroCollector, espnAmericanCollector, espnLacrosseCollector, sportsblazeAmericanCollector e unifiedGamesCollector).

### Exemplo 1 — Resposta completa com today, overnight e tomorrow (resumo de coleções)

JSON

```
{
  "contractVersion": "1.0",
  "ok": true,
  "degraded": false,
  "snapshotId": "ug-2026-11-20-1795224900000",
  "generatedAt": "2026-11-21T01:35:00.000Z",
  "referenceDate": "2026-11-20",
  "timezone": "America/Sao_Paulo",
  "window": {
    "yesterday": "2026-11-19",
    "today": "2026-11-20",
    "tomorrow": "2026-11-21"
  },
  "cache": {
    "hit": false,
    "ageMs": 0,
    "ttlMs": 60000
  },
  "filters": {
    "sets": ["today", "overnight", "tomorrow"],
    "sport": null,
    "league": null,
    "status": null,
    "liveOnly": false,
    "includeProvenance": false,
    "includeStats": "summary",
    "includeCompat": false
  },
  "counts": {
    "returned": { "today": 2, "overnight": 1, "tomorrow": 1, "total": 4 },
    "snapshotTotal": { "today": 79, "overnight": 1, "tomorrow": 303, "total": 383 },
    "live": { "returned": 2, "snapshotTotal": 14 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": true,
      "partial": false,
      "feedUpdatedAt": null,
      "totalCalls": 3,
      "succeededCalls": 3,
      "failedCalls": 0,
      "emptySlateCalls": 0,
      "dedupedItemCount": 368
    },
    "espn": {
      "ok": true,
      "partial": false,
      "feedUpdatedAt": null,
      "totalCalls": 21,
      "succeededCalls": 21,
      "failedCalls": 0,
      "emptySlateCalls": 6,
      "dedupedItemCount": 32
    },
    "sportsblaze": {
      "ok": true,
      "partial": false,
      "feedUpdatedAt": "2026-11-21T01:34:42.000Z",
      "totalCalls": 16,
      "succeededCalls": 16,
      "failedCalls": 0,
      "emptySlateCalls": 5,
      "dedupedItemCount": 24
    }
  },
  "warnings": [],
  "errors": [],
  "sets": {
    "today": [
      {
        "canonicalId": "ug:nba:espn:401810101",
        "set": "TODAY",
        "sport": "basketball",
        "sportFamily": "american",
        "league": {
          "code": "NBA",
          "name": "NBA",
          "id": null,
          "country": "USA",
          "flag": null,
          "logo": null
        },
        "providerIds": {
          "espnId": "401810101",
          "sportsblazeId": "f1a2c3d4-1111-4222-8333-444455556666",
          "bzzoiroId": null,
          "lacrosseCollectorId": null
        },
        "providers": ["espn", "sportsblaze"],
        "reconciled": true,
        "reconciliationMode": "matched",
        "fixture": {
          "startTimeUtc": "2026-11-21T00:30:00.000Z",
          "rawDate": "2026-11-21T00:30Z",
          "timestampSource": "espn",
          "startDateBrt": "2026-11-20",
          "startTimeBrt": "21:30",
          "dateDisplay": "20/11/2026",
          "venue": "TD Garden",
          "broadcaster": "NBA League Pass",
          "season": { "year": 2026, "type": "Regular Season", "week": null },
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": {
            "id": "sb-team-bos",
            "name": "Boston Celtics",
            "abbreviation": "BOS",
            "logo": "https://a.espncdn.com/i/teamlogos/nba/500/bos.png"
          },
          "awayTeam": {
            "id": "sb-team-ny",
            "name": "New York Knicks",
            "abbreviation": "NY",
            "logo": "https://a.espncdn.com/i/teamlogos/nba/500/ny.png"
          }
        },
        "liveOverlay": {
          "status": "LIVE",
          "live": true,
          "score": { "home": 54, "away": 56 },
          "timeBadge": "2:19 - 2nd",
          "period": 2,
          "clock": "2:19",
          "tempo": "2:19 - 2nd",
          "liveState": { "period": 2, "clock": "2:19" },
          "linescores": {
            "header": ["Q1", "Q2", "Q3", "Q4"],
            "home": [28, 26, "-", "-"],
            "away": [30, 26, "-", "-"]
          },
          "incidents": null,
          "eventUpdatedAt": null
        },
        "stats": {
          "available": true,
          "matchStatsId": "sb-nba-401810101",
          "endpoint": "/api/match-stats?id=sb-nba-401810101&sport=NBA&league=NBA",
          "hasTeamStats": true,
          "hasPlayerStats": true,
          "keyPlayers": {
            "home": [{ "name": "Jayson Tatum", "pos": "F", "stat": "points: 18 · rebounds: 5" }],
            "away": [{ "name": "Jalen Brunson", "pos": "G", "stat": "points: 21 · assists: 4" }]
          }
        }
      },
      {
        "canonicalId": "ug:football:bzzoiro:888001",
        "set": "TODAY",
        "sport": "football",
        "sportFamily": "football",
        "league": {
          "code": "Brasileirão Série A",
          "name": "Brasileirão Série A",
          "id": 9,
          "country": "Brasil",
          "flag": "🇧🇷",
          "logo": "https://sports.bzzoiro.com/img/league/9/"
        },
        "providerIds": {
          "espnId": null,
          "sportsblazeId": null,
          "bzzoiroId": "888001",
          "lacrosseCollectorId": null
        },
        "providers": ["bzzoiro"],
        "reconciled": false,
        "reconciliationMode": "bzzoiro_only",
        "fixture": {
          "startTimeUtc": "2026-11-21T00:00:00.000Z",
          "rawDate": "2026-11-21T00:00:00Z",
          "timestampSource": "bzzoiro",
          "startDateBrt": "2026-11-20",
          "startTimeBrt": "21:00",
          "dateDisplay": "20/11/2026",
          "venue": "Maracanã",
          "broadcaster": "Premiere, Globo",
          "season": null,
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": {
            "id": null,
            "name": "Flamengo",
            "abbreviation": "FLA",
            "logo": "https://sports.bzzoiro.com/img/team/1951/"
          },
          "awayTeam": {
            "id": null,
            "name": "Palmeiras",
            "abbreviation": "PAL",
            "logo": "https://sports.bzzoiro.com/img/team/1963/"
          }
        },
        "liveOverlay": {
          "status": "LIVE",
          "live": true,
          "score": { "home": 2, "away": 1 },
          "timeBadge": "67'",
          "period": null,
          "clock": "67'",
          "tempo": "67'",
          "liveState": null,
          "linescores": { "header": [], "home": [], "away": [] },
          "incidents": {
            "homeGoals": ["Pedro 14'", "Arrascaeta 52'"],
            "awayGoals": ["Estêvão 38'"]
          },
          "eventUpdatedAt": null
        },
        "stats": {
          "available": true,
          "matchStatsId": "bsd-888001",
          "endpoint": "/api/match-stats?id=bsd-888001&sport=Futebol&league=Brasileir%C3%A3o%20S%C3%A9rie%20A",
          "hasTeamStats": false,
          "hasPlayerStats": false
        }
      }
    ],
    "overnight": [
      {
        "canonicalId": "ug:nba:espn:401810095",
        "set": "OVERNIGHT",
        "sport": "basketball",
        "sportFamily": "american",
        "league": { "code": "NBA", "name": "NBA", "id": null, "country": "USA", "flag": null, "logo": null },
        "providerIds": {
          "espnId": "401810095",
          "sportsblazeId": null,
          "bzzoiroId": null,
          "lacrosseCollectorId": null
        },
        "providers": ["espn"],
        "reconciled": false,
        "reconciliationMode": "espn_only",
        "fixture": {
          "startTimeUtc": "2026-11-20T02:30:00.000Z",
          "rawDate": "2026-11-20T02:30Z",
          "timestampSource": "espn",
          "startDateBrt": "2026-11-19",
          "startTimeBrt": "23:30",
          "dateDisplay": "19/11/2026",
          "venue": "Crypto.com Arena",
          "broadcaster": "Disney+",
          "season": null,
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": {
            "id": null,
            "name": "Los Angeles Lakers",
            "abbreviation": "LAL",
            "logo": "https://a.espncdn.com/i/teamlogos/nba/500/lal.png"
          },
          "awayTeam": {
            "id": null,
            "name": "Denver Nuggets",
            "abbreviation": "DEN",
            "logo": "https://a.espncdn.com/i/teamlogos/nba/500/den.png"
          }
        },
        "liveOverlay": {
          "status": "LIVE",
          "live": true,
          "score": { "home": 108, "away": 106 },
          "timeBadge": "1:12 - OT",
          "period": 5,
          "clock": "1:12",
          "tempo": "1:12 - OT",
          "liveState": { "period": 5, "clock": "1:12" },
          "linescores": {
            "header": ["Q1", "Q2", "Q3", "Q4", "OT"],
            "home": [24, 27, 25, 24, 8],
            "away": [26, 22, 28, 24, 6]
          },
          "incidents": null,
          "eventUpdatedAt": null
        },
        "stats": {
          "available": true,
          "matchStatsId": "sb-nba-401810095",
          "endpoint": "/api/match-stats?id=sb-nba-401810095&sport=NBA&league=NBA",
          "hasTeamStats": false,
          "hasPlayerStats": false
        }
      }
    ],
    "tomorrow": [
      {
        "canonicalId": "ug:american_football:espn:401772900",
        "set": "TOMORROW",
        "sport": "american_football",
        "sportFamily": "american",
        "league": { "code": "NFL", "name": "NFL", "id": null, "country": "USA", "flag": null, "logo": null },
        "providerIds": {
          "espnId": "401772900",
          "sportsblazeId": "a9b8c7d6-0001-4000-9000-112233445566",
          "bzzoiroId": null,
          "lacrosseCollectorId": null
        },
        "providers": ["espn", "sportsblaze"],
        "reconciled": true,
        "reconciliationMode": "matched",
        "fixture": {
          "startTimeUtc": "2026-11-21T21:25:00.000Z",
          "rawDate": "2026-11-21T21:25Z",
          "timestampSource": "espn",
          "startDateBrt": "2026-11-21",
          "startTimeBrt": "18:25",
          "dateDisplay": "21/11/2026",
          "venue": "Arrowhead Stadium",
          "broadcaster": "ESPN 2, Disney+",
          "season": { "year": 2026, "type": "Regular Season", "week": 12 },
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": {
            "id": "sb-team-kc",
            "name": "Kansas City Chiefs",
            "abbreviation": "KC",
            "logo": "https://a.espncdn.com/i/teamlogos/nfl/500/kc.png"
          },
          "awayTeam": {
            "id": "sb-team-buf",
            "name": "Buffalo Bills",
            "abbreviation": "BUF",
            "logo": "https://a.espncdn.com/i/teamlogos/nfl/500/buf.png"
          }
        },
        "liveOverlay": {
          "status": "UPCOMING",
          "live": false,
          "score": { "home": null, "away": null },
          "timeBadge": "18:25",
          "period": null,
          "clock": null,
          "tempo": "",
          "liveState": null,
          "linescores": {
            "header": ["Q1", "Q2", "Q3", "Q4"],
            "home": ["-", "-", "-", "-"],
            "away": ["-", "-", "-", "-"]
          },
          "incidents": null,
          "eventUpdatedAt": null
        },
        "stats": {
          "available": true,
          "matchStatsId": "sb-nfl-401772900",
          "endpoint": "/api/match-stats?id=sb-nfl-401772900&sport=NFL&league=NFL",
          "hasTeamStats": false,
          "hasPlayerStats": false
        }
      }
    ]
  }
}
```

### Exemplo 2 — Resposta filtrada por modalidade (GET /api/games/central?date=2026-11-20&sets=today&sport=basketball&includeStats=none)

JSON

```
{
  "contractVersion": "1.0",
  "ok": true,
  "degraded": false,
  "snapshotId": "ug-2026-11-20-1795224900000",
  "generatedAt": "2026-11-21T01:35:00.000Z",
  "referenceDate": "2026-11-20",
  "timezone": "America/Sao_Paulo",
  "window": { "yesterday": "2026-11-19", "today": "2026-11-20", "tomorrow": "2026-11-21" },
  "cache": { "hit": true, "ageMs": 4200, "ttlMs": 60000 },
  "filters": {
    "sets": ["today"],
    "sport": ["basketball"],
    "league": null,
    "status": null,
    "liveOnly": false,
    "includeProvenance": false,
    "includeStats": "none",
    "includeCompat": false
  },
  "counts": {
    "returned": { "today": 1, "overnight": 0, "tomorrow": 0, "total": 1 },
    "snapshotTotal": { "today": 79, "overnight": 1, "tomorrow": 303, "total": 383 },
    "live": { "returned": 1, "snapshotTotal": 14 }
  },
  "providerStatus": {
    "bzzoiro": { "ok": true, "partial": false, "feedUpdatedAt": null, "totalCalls": 3, "succeededCalls": 3, "failedCalls": 0, "emptySlateCalls": 0, "dedupedItemCount": 368 },
    "espn": { "ok": true, "partial": false, "feedUpdatedAt": null, "totalCalls": 21, "succeededCalls": 21, "failedCalls": 0, "emptySlateCalls": 6, "dedupedItemCount": 32 },
    "sportsblaze": { "ok": true, "partial": false, "feedUpdatedAt": "2026-11-21T01:34:42.000Z", "totalCalls": 16, "succeededCalls": 16, "failedCalls": 0, "emptySlateCalls": 5, "dedupedItemCount": 24 }
  },
  "warnings": [],
  "errors": [],
  "sets": {
    "today": [
      {
        "canonicalId": "ug:nba:espn:401810101",
        "set": "TODAY",
        "sport": "basketball",
        "sportFamily": "american",
        "league": { "code": "NBA", "name": "NBA", "id": null, "country": "USA", "flag": null, "logo": null },
        "providerIds": { "espnId": "401810101", "sportsblazeId": "f1a2c3d4-1111-4222-8333-444455556666", "bzzoiroId": null, "lacrosseCollectorId": null },
        "providers": ["espn", "sportsblaze"],
        "reconciled": true,
        "reconciliationMode": "matched",
        "fixture": {
          "startTimeUtc": "2026-11-21T00:30:00.000Z",
          "rawDate": "2026-11-21T00:30Z",
          "timestampSource": "espn",
          "startDateBrt": "2026-11-20",
          "startTimeBrt": "21:30",
          "dateDisplay": "20/11/2026",
          "venue": "TD Garden",
          "broadcaster": "NBA League Pass",
          "season": { "year": 2026, "type": "Regular Season", "week": null },
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": { "id": "sb-team-bos", "name": "Boston Celtics", "abbreviation": "BOS", "logo": "https://a.espncdn.com/i/teamlogos/nba/500/bos.png" },
          "awayTeam": { "id": "sb-team-ny", "name": "New York Knicks", "abbreviation": "NY", "logo": "https://a.espncdn.com/i/teamlogos/nba/500/ny.png" }
        },
        "liveOverlay": {
          "status": "LIVE",
          "live": true,
          "score": { "home": 54, "away": 56 },
          "timeBadge": "2:19 - 2nd",
          "period": 2,
          "clock": "2:19",
          "tempo": "2:19 - 2nd",
          "liveState": { "period": 2, "clock": "2:19" },
          "linescores": { "header": ["Q1", "Q2", "Q3", "Q4"], "home": [28, 26, "-", "-"], "away": [30, 26, "-", "-"] },
          "incidents": null,
          "eventUpdatedAt": null
        },
        "stats": {
          "available": true,
          "matchStatsId": "sb-nba-401810101",
          "endpoint": "/api/match-stats?id=sb-nba-401810101&sport=NBA&league=NBA",
          "hasTeamStats": true,
          "hasPlayerStats": true
        }
      }
    ],
    "overnight": [],
    "tomorrow": []
  }
}
```

### Exemplo 3 — Partida de futebol ao vivo (Bzzoiro real-shaped)

JSON

```
{
  "canonicalId": "ug:football:bzzoiro:891204",
  "set": "TODAY",
  "sport": "football",
  "sportFamily": "football",
  "league": {
    "code": "A-League Men",
    "name": "A-League Men",
    "id": 236,
    "country": "Austrália",
    "flag": "🇦🇺",
    "logo": "https://sports.bzzoiro.com/img/league/236/"
  },
  "providerIds": {
    "espnId": null,
    "sportsblazeId": null,
    "bzzoiroId": "891204",
    "lacrosseCollectorId": null
  },
  "providers": ["bzzoiro"],
  "reconciled": false,
  "reconciliationMode": "bzzoiro_only",
  "fixture": {
    "startTimeUtc": "2026-11-21T08:35:00.000Z",
    "rawDate": "2026-11-21T08:35:00Z",
    "timestampSource": "bzzoiro",
    "startDateBrt": "2026-11-21",
    "startTimeBrt": "05:35",
    "dateDisplay": "21/11/2026",
    "venue": "Allianz Stadium",
    "broadcaster": "Sem Transmissão",
    "season": null,
    "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
    "homeTeam": {
      "id": null,
      "name": "Sydney FC",
      "abbreviation": "SYD",
      "logo": "https://sports.bzzoiro.com/img/team/5971/"
    },
    "awayTeam": {
      "id": null,
      "name": "Brisbane Roar",
      "abbreviation": "BRI",
      "logo": "https://sports.bzzoiro.com/img/team/5968/"
    }
  },
  "liveOverlay": {
    "status": "LIVE",
    "live": true,
    "score": { "home": 1, "away": 0 },
    "timeBadge": "41'",
    "period": null,
    "clock": "41'",
    "tempo": "41'",
    "liveState": null,
    "linescores": { "header": [], "home": [], "away": [] },
    "incidents": {
      "homeGoals": ["Lolley 22'"],
      "awayGoals": []
    },
    "eventUpdatedAt": null
  },
  "stats": {
    "available": true,
    "matchStatsId": "bsd-891204",
    "endpoint": "/api/match-stats?id=bsd-891204&sport=Futebol&league=A-League%20Men",
    "hasTeamStats": false,
    "hasPlayerStats": false
  }
}
```

### Exemplo 4 — Partida de NBA ao vivo (reconciliada ESPN + SportsBlaze)

JSON

```
{
  "canonicalId": "ug:nba:espn:401810105",
  "set": "TODAY",
  "sport": "basketball",
  "sportFamily": "american",
  "league": { "code": "NBA", "name": "NBA", "id": null, "country": "USA", "flag": null, "logo": null },
  "providerIds": {
    "espnId": "401810105",
    "sportsblazeId": "7d8e9f00-2222-4444-8888-999900001111",
    "bzzoiroId": null,
    "lacrosseCollectorId": null
  },
  "providers": ["espn", "sportsblaze"],
  "reconciled": true,
  "reconciliationMode": "matched",
  "fixture": {
    "startTimeUtc": "2026-11-21T01:00:00.000Z",
    "rawDate": "2026-11-21T01:00Z",
    "timestampSource": "espn",
    "startDateBrt": "2026-11-20",
    "startTimeBrt": "22:00",
    "dateDisplay": "20/11/2026",
    "venue": "Chase Center",
    "broadcaster": "ESPN, Disney+",
    "season": { "year": 2026, "type": "Regular Season", "week": null },
    "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
    "homeTeam": {
      "id": "sb-team-gs",
      "name": "Golden State Warriors",
      "abbreviation": "GS",
      "logo": "https://a.espncdn.com/i/teamlogos/nba/500/gs.png"
    },
    "awayTeam": {
      "id": "sb-team-phx",
      "name": "Phoenix Suns",
      "abbreviation": "PHX",
      "logo": "https://a.espncdn.com/i/teamlogos/nba/500/phx.png"
    }
  },
  "liveOverlay": {
    "status": "LIVE",
    "live": true,
    "score": { "home": 54, "away": 56 },
    "timeBadge": "2:19 - 2nd",
    "period": 2,
    "clock": "2:19",
    "tempo": "2:19 - 2nd",
    "liveState": { "period": 2, "clock": "2:19" },
    "linescores": {
      "header": ["Q1", "Q2", "Q3", "Q4"],
      "home": [29, 25, "-", "-"],
      "away": [27, 29, "-", "-"]
    },
    "incidents": null,
    "eventUpdatedAt": null
  },
  "stats": {
    "available": true,
    "matchStatsId": "sb-nba-401810105",
    "endpoint": "/api/match-stats?id=sb-nba-401810105&sport=NBA&league=NBA",
    "hasTeamStats": true,
    "hasPlayerStats": true,
    "keyPlayers": {
      "home": [{ "name": "Stephen Curry", "pos": "G", "stat": "points: 17 · assists: 5" }],
      "away": [{ "name": "Kevin Durant", "pos": "F", "stat": "points: 19 · rebounds: 4" }]
    }
  }
}
```

### Exemplo 5 — Partida futura (UPCOMING)

JSON

```
{
  "canonicalId": "ug:nhl:espn:401687990",
  "set": "TOMORROW",
  "sport": "hockey",
  "sportFamily": "american",
  "league": { "code": "NHL", "name": "NHL", "id": null, "country": "USA", "flag": null, "logo": null },
  "providerIds": {
    "espnId": "401687990",
    "sportsblazeId": null,
    "bzzoiroId": null,
    "lacrosseCollectorId": null
  },
  "providers": ["espn"],
  "reconciled": false,
  "reconciliationMode": "espn_only",
  "fixture": {
    "startTimeUtc": "2026-11-22T00:00:00.000Z",
    "rawDate": "2026-11-22T00:00Z",
    "timestampSource": "espn",
    "startDateBrt": "2026-11-21",
    "startTimeBrt": "21:00",
    "dateDisplay": "21/11/2026",
    "venue": "Madison Square Garden",
    "broadcaster": "Disney+",
    "season": null,
    "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
    "homeTeam": {
      "id": null,
      "name": "New York Rangers",
      "abbreviation": "NYR",
      "logo": "https://a.espncdn.com/i/teamlogos/nhl/500/nyr.png"
    },
    "awayTeam": {
      "id": null,
      "name": "Boston Bruins",
      "abbreviation": "BOS",
      "logo": "https://a.espncdn.com/i/teamlogos/nhl/500/bos.png"
    }
  },
  "liveOverlay": {
    "status": "UPCOMING",
    "live": false,
    "score": { "home": null, "away": null },
    "timeBadge": "21:00",
    "period": null,
    "clock": null,
    "tempo": "",
    "liveState": null,
    "linescores": {
      "header": ["P1", "P2", "P3"],
      "home": ["-", "-", "-"],
      "away": ["-", "-", "-"]
    },
    "incidents": null,
    "eventUpdatedAt": null
  },
  "stats": {
    "available": true,
    "matchStatsId": "sb-nhl-401687990",
    "endpoint": "/api/match-stats?id=sb-nhl-401687990&sport=NHL&league=NHL",
    "hasTeamStats": false,
    "hasPlayerStats": false
  }
}
```

### Exemplo 6 — Resposta degradada por falha parcial de provedor (HTTP 200, ok: true, degraded: true)

JSON

```
{
  "contractVersion": "1.0",
  "ok": true,
  "degraded": true,
  "snapshotId": "ug-2026-11-21-1795300000000",
  "generatedAt": "2026-11-21T14:00:00.000Z",
  "referenceDate": "2026-11-21",
  "timezone": "America/Sao_Paulo",
  "window": { "yesterday": "2026-11-20", "today": "2026-11-21", "tomorrow": "2026-11-22" },
  "cache": { "hit": false, "ageMs": 0, "ttlMs": 60000 },
  "filters": {
    "sets": ["today", "overnight", "tomorrow"],
    "sport": null,
    "league": null,
    "status": null,
    "liveOnly": false,
    "includeProvenance": false,
    "includeStats": "summary",
    "includeCompat": false
  },
  "counts": {
    "returned": { "today": 11, "overnight": 0, "tomorrow": 8, "total": 19 },
    "snapshotTotal": { "today": 11, "overnight": 0, "tomorrow": 8, "total": 19 },
    "live": { "returned": 1, "snapshotTotal": 1 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": false,
      "partial": false,
      "feedUpdatedAt": null,
      "totalCalls": 3,
      "succeededCalls": 0,
      "failedCalls": 3,
      "emptySlateCalls": 0,
      "dedupedItemCount": 0
    },
    "espn": {
      "ok": true,
      "partial": false,
      "feedUpdatedAt": null,
      "totalCalls": 21,
      "succeededCalls": 21,
      "failedCalls": 0,
      "emptySlateCalls": 12,
      "dedupedItemCount": 19
    },
    "sportsblaze": {
      "ok": true,
      "partial": true,
      "feedUpdatedAt": "2026-11-21T13:59:10.000Z",
      "totalCalls": 16,
      "succeededCalls": 15,
      "failedCalls": 1,
      "emptySlateCalls": 10,
      "dedupedItemCount": 11
    }
  },
  "warnings": [
    "provider_unavailable:bzzoiro",
    "provider_partial:sportsblaze"
  ],
  "errors": [
    {
      "code": "PROVIDER_UNAVAILABLE",
      "message": "Bzzoiro indisponível nas 3 datas da janela.",
      "provider": "bzzoiro",
      "scope": "bzzoiro:2026-11-21",
      "statusCode": 503
    },
    {
      "code": "PROVIDER_PARTIAL_FAILURE",
      "message": "SportsBlaze falhou em 1 consulta de boxscore.",
      "provider": "sportsblaze",
      "scope": "sportsblaze:boxscores:nba:2026-11-21",
      "statusCode": 502
    }
  ],
  "sets": {
    "today": [],
    "overnight": [],
    "tomorrow": []
  }
}
```

### Exemplo 7 — Partida com includeProvenance=true (reconciliação com divergência registrada)

JSON

```
{
  "canonicalId": "ug:nba:espn:401810105",
  "set": "TODAY",
  "sport": "basketball",
  "sportFamily": "american",
  "league": { "code": "NBA", "name": "NBA", "id": null, "country": "USA", "flag": null, "logo": null },
  "providerIds": {
    "espnId": "401810105",
    "sportsblazeId": "7d8e9f00-2222-4444-8888-999900001111",
    "bzzoiroId": null,
    "lacrosseCollectorId": null
  },
  "providers": ["espn", "sportsblaze"],
  "reconciled": true,
  "reconciliationMode": "matched",
  "fixture": {
    "startTimeUtc": "2026-11-21T01:00:00.000Z",
    "rawDate": "2026-11-21T01:00Z",
    "timestampSource": "espn",
    "startDateBrt": "2026-11-20",
    "startTimeBrt": "22:00",
    "dateDisplay": "20/11/2026",
    "venue": "Chase Center",
    "broadcaster": "ESPN, Disney+",
    "season": { "year": 2026, "type": "Regular Season", "week": null },
    "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
    "homeTeam": {
      "id": "sb-team-gs",
      "name": "Golden State Warriors",
      "abbreviation": "GS",
      "logo": "https://a.espncdn.com/i/teamlogos/nba/500/gs.png"
    },
    "awayTeam": {
      "id": "sb-team-phx",
      "name": "Phoenix Suns",
      "abbreviation": "PHX",
      "logo": "https://a.espncdn.com/i/teamlogos/nba/500/phx.png"
    }
  },
  "liveOverlay": {
    "status": "LIVE",
    "live": true,
    "score": { "home": 54, "away": 56 },
    "timeBadge": "2:19 - 2nd",
    "period": 2,
    "clock": "2:19",
    "tempo": "2:19 - 2nd",
    "liveState": { "period": 2, "clock": "2:19" },
    "linescores": {
      "header": ["Q1", "Q2", "Q3", "Q4"],
      "home": [29, 25, "-", "-"],
      "away": [27, 29, "-", "-"]
    },
    "incidents": null,
    "eventUpdatedAt": null
  },
  "stats": {
    "available": true,
    "matchStatsId": "sb-nba-401810105",
    "endpoint": "/api/match-stats?id=sb-nba-401810105&sport=NBA&league=NBA",
    "hasTeamStats": true,
    "hasPlayerStats": true
  },
  "provenance": {
    "reconciliationDeltaMinutes": 0,
    "rawStatusByProvider": {
      "espn": "LIVE",
      "sportsblaze": "In Progress"
    },
    "fieldProvenance": {
      "startTime": { "source": "espn", "fallbackUsed": false },
      "status": { "source": "espn", "fallbackUsed": false },
      "score": { "source": "espn", "fallbackUsed": false },
      "timeBadge": { "source": "espn", "fallbackUsed": false },
      "period": { "source": "espn", "fallbackUsed": false },
      "clock": { "source": "espn", "fallbackUsed": false },
      "liveState": { "source": "espn", "fallbackUsed": false },
      "linescores": { "source": "espn", "fallbackUsed": false },
      "clockPeriod": { "source": "espn", "fallbackUsed": false },
      "teamStatistics": { "source": "sportsblaze", "fallbackUsed": false },
      "players": { "source": "sportsblaze", "fallbackUsed": false },
      "venueBroadcaster": { "source": "espn", "fallbackUsed": false },
      "teamLogos": { "source": "espn", "fallbackUsed": false }
    },
    "divergences": [
      {
        "field": "score",
        "espnValue": { "home": 54, "away": 56 },
        "sportsblazeValue": { "home": 49, "away": 51 },
        "selectedSource": "espn",
        "reason": "espn_primary_authority",
        "detail": "Placar divergente entre ESPN (54-56) e SportsBlaze (49-51); ESPN priorizada."
      },
      {
        "field": "clock",
        "espnValue": "2:19",
        "sportsblazeValue": "4:08",
        "selectedSource": "espn",
        "reason": "sb_score_mismatch",
        "detail": "Relógio SportsBlaze rejeitado pelo gate de consistência (placar atrasado vs ESPN)."
      }
    ]
  }
}
```

---

## ENTREGÁVEL D — Compatibilidade & Estratégia de Migração

### 1. Identificadores e preservação do /api/match-stats

- **ID canônico oficial (canonicalId):**
    
    - Futebol: ug:football:bzzoiro:<bzzoiroId>
        
    - Americanos (reconciliado ou ESPN): ug:<leagueLower>:espn:<espnId>
        
    - Americanos (sportsblaze_only): ug:<leagueLower>:sb:<sportsblazeId>
        
    - Lacrosse: ug:lacrosse:espn:<espnId>
        
- **Relação explícita com /api/match-stats:**
    
    - stats.matchStatsId entrega diretamente a chave aceita hoje por /api/match-stats:
        
        - Futebol -> bsd-<bzzoiroId>
            
        - Americanos com espnId -> sb-<leagueLower>-<espnId>
            
        - sportsblaze_only -> null (stats.available = false se não houver estatísticas já embutidas; se houver teamStatistics/players, available = true via includeStats=full, e matchStatsId = null para não tentar consultar ESPN com UUID).
            
        - Lacrosse -> lacrosse-<espnId> (ou null caso /api/match-stats não possua handler para lacrosse).
            
- **Migração futura de /api/match-stats:** em etapa posterior, /api/match-stats poderá aceitar ?canonicalId=ug:... mantendo suporte simultâneo aos IDs legados (bsd-, sb-, lacrosse-).
    

### 2. Plano de adoção incremental por consumidor

|   |   |   |
|---|---|---|
|Consumidor|Fonte atual|Como migrar para /api/games/central sem regressão|
|**JogosDoDiaPage**|/api/football/today + /api/sportsblaze/* + /api/sports/lacrosse-pll|Primeira página a migrar (Etapa 2B.5): consome GET /api/games/central?sets=today,overnight,tomorrow&includeStats=summary. Mapeia sets.today + sets.overnight + sets.tomorrow para os cards e abre estatísticas via stats.endpoint.|
|**Ticker**|/api/football/today + /api/sportsblaze/* + sportsData|Consome sets.today + sets.overnight (via hook/serviço compartilhado em memória no cliente para não duplicar chamadas com JogosDoDiaPage). Usa fixture.startDateBrt, fixture.startTimeBrt e liveOverlay.timeBadge, eliminando inferência por prefixo de ID.|
|**HeroCommandCenter + HeroCopa**|tickerItems + sportsData + /api/match-stats|Lê jogos LIVE e UPCOMING do mesmo serviço cliente alimentado por /api/games/central?includeStats=summary; usa stats.keyPlayers para destaque rápido e stats.matchStatsId para /api/match-stats. compat pode ser ativado temporariamente (includeCompat=true) durante a transição de HeroCopa.|
|**HomeLite / /api/jogos-texto**|CMSService.getJogosTexto() (/api/jogos-texto)|Backend de /api/jogos-texto (ou próprio HomeLite) passa a ler a função interna do agregador central, aplicando apenas a formatação enxuta PT-BR (AO VIVO/AGENDADO/ENCERRADO/ADIADO).|
|**MeusTimesPage**|/api/football/today + agendas individuais por time (favoritesService)|Passa a consultar /api/games/central para os jogos de hoje/amanhã dos times favoritos e mantém consultas específicas de calendário futuro de clube quando fora da janela [yesterday..tomorrow].|

### 3. Estratégia recomendada para o futuro **ORB Lite** (fields=... vs GET /api/v1/games)

- **Análise comparativa:**
    
    1. **?fields=... no /api/games/central:** permite projeção seletiva simples, mas mantém acoplamento à estrutura aninhada do portal web (fixture, liveOverlay, stats), exige parser de projeção arbitrária no servidor (risco de N+1 bugs de serialização e explosão combinatória de ETags por variação de fields).
        
    2. **Endpoint dedicado futuro GET /api/v1/games:** contrato público flat/minimalista, versionado por /v1/, com payload pensado para mobile/widget/consumo externo (apenas id, sport, league, startTimeUtc, startTimeBrt, status, teams, score, clock, broadcaster), **alimentado pela mesma função de serviço interna** que serve /api/games/central.
        
- **Recomendação:**
    
    - **Não** implementar parser genérico de ?fields=... na v1.0 de /api/games/central. Utilizar presets explícitos e controlados (includeStats=none|summary|full, includeProvenance, includeCompat), que atendem 100% do Órbita Arena e ORB Lite interno com baixo risco e cache eficiente.
        
    - Quando o **ORB Lite** for lançado como produto/API separada, criar **GET /api/v1/games** reutilizando exatamente o mesmo cache/snapshot do agregador central no backend.
        

---

## ENTREGÁVEL E — Estratégia de Cache, ETag e Versionamento

### 1. Arquitetura de Cache em 4 Camadas

1. **Camada 1 — Coletores (bzzoiroCollector, espnAmericanCollector, espnLacrosseCollector, sportsblazeAmericanCollector):**
    
    - Mantêm seus TTLs próprios (Bzzoiro 180s, ESPN 60s, SportsBlaze Boxscores 60s / Schedule 300s) e in-flight promise coalescing.
        
    - Se JogosDoDia legado e /api/games/central rodarem simultaneamente durante a migração, ambos compartilham os mesmos coletores sem duplicar tráfego nos provedores externos.
        
2. **Camada 2 — Agregador Central (UNIFIED_CACHE_TTL_MS = 60_000):**
    
    - Chave por referenceDate (unified:central:${referenceDateBrt}).
        
    - TTL de **60 segundos** com in-flight coalescing por referenceDate.
        
    - Gera um snapshotId determinístico por geração (ug-${referenceDateBrt}-${generatedAtMs}) e retorna cópia isolada em memória. Falha total (ok: false) nunca entra em cache.
        
3. **Camada 3 — HTTP / CDN (ETag + 304 Not Modified):**
    
    - ETag fraco calculado sobre (snapshotId + assinatura normalizada dos filtros aplicados), ex.: W/"ug-2026-11-20-1795224900000-f8a3c1".
        
    - Se o cliente enviar If-None-Match idêntico **e** o snapshot em cache ainda estiver dentro do TTL de 60s, o servidor responde **304 Not Modified** imediatamente sem reserializar o JSON.
        
    - **Prevenção de dados stale:** Cache-Control deve calcular max-age restante até o vencimento do snapshot (ou teto curto: Cache-Control: public, max-age=15, must-revalidate quando houver jogos LIVE ou no máximo remainingTtlSeconds, nunca ultrapassando 60s). Em degraded: true, usar Cache-Control: public, max-age=10, must-revalidate. Em ok: false (503), usar Cache-Control: no-store.
        
    - Header obrigatório Vary: Accept-Encoding.
        
4. **Camada 4 — Cliente React (Single Shared Store/Hook):**
    
    - Substitui múltiplos fetch independentes (Ticker, Hero, JogosDoDiaPage) por uma única assinatura/polling a cada 60s (ou 30s com revalidação via If-None-Match -> 304), respeitando document.visibilityState === "visible".
        

### 2. Versionamento de Contrato (contractVersion)

- Campo obrigatório contractVersion: "1.0" em todas as respostas (200, 400, 503).
    
- **Mudanças retrocompatíveis (1.x, sem quebra):** adição de novos campos opcionais ou novas modalidades/ligas nos enums extensíveis.
    
- **Mudanças incompatíveis (2.0):** alteração/remoção de campos obrigatórios ou mudança de semântica -> exige negociação via nova rota /api/v2/games/central ou header explícito, nunca quebra silenciosa na 1.0.
    

---

## ENTREGÁVEL F — Decisões Pendentes para Aprovação (Pré-Etapa 2B.4.2)

1. **Default de sets quando omitido em /api/games/central:**
    
    - Proposta recomendada: today,overnight,tomorrow (1 única requisição abastece Ticker, Hero e abas Hoje/Amanhã da página Jogos do Dia). Alternativa: sets=today por padrão.
        
2. **Default de includeStats:**
    
    - Proposta recomendada: summary (inclui keyPlayers resumidos e incidents de gols em futebol, mantendo payload enxuto e dispensando teamStatistics e listas inteiras de atletas até que o usuário abra o modal de detalhes). Alternativa: none.
        
3. **Estratégia de remoção de heroCompat:**
    
    - Proposta recomendada: manter includeCompat=false como padrão em /api/games/central, ativando includeCompat=true apenas temporariamente durante a transição dos primeiros consumidores (Etapa 2B.5), com remoção planejada após conclusão da migração.
        
4. **Política de fallback para SportsBlaze-only em /api/match-stats:**
    
    - Confirmar que partidas sportsblaze_only terão stats.matchStatsId = null e usarão exclusivamente os dados em memória (includeStats=full quando necessário), sem tentar adaptar /api/match-stats para UUIDs nesta fase.
        

---

## ENTREGÁVEL G — Arquivos Inspecionados e Confirmação de Somente Leitura

### Arquivos inspecionados durante a auditoria da Etapa 2B.4.1

- src/server/collectors/unifiedGamesCollector.ts
    
- src/server/registerCollectorRoutes.ts
    
- src/server/collectors/bzzoiroCollector.ts
    
- src/server/collectors/espnAmericanCollector.ts
    
- src/server/collectors/espnLacrosseCollector.ts
    
- src/server/collectors/sportsblazeAmericanCollector.ts
    
- src/utils/brtDate.ts
    
- src/utils/gameSets.ts
    
- server.ts (rotas /api/match-stats)
    
- src/services/sportsblazeService.ts
    
- src/services/bsdAdapter.ts
    
- src/services/matchStatsService.ts
    
- src/components/HeroCommandCenter.tsx
    
- src/components/HeroCopa.tsx
    
- src/components/Ticker.tsx
    
- src/components/JogosDoDiaPage.tsx
    
- src/components/MeusTimesPage.tsx
    
- src/components/lite/HomeLite.tsx
    

### Confirmação obrigatória

**Nenhum arquivo foi criado, editado ou removido** durante a Etapa 2B.4.1. O endpoint /api/games/central não foi implementado e os fluxos atuais de produção e o modo Shadow (/api/games/shadow) permanecem intactos.
