# ÓRBITA ARENA — ETAPA 2B.4.2
## Contrato Consolidado e Aprovado da API Central de Jogos (`GET /api/games/central`)

**Versão do Contrato:** `1.0`  
**Status:** Especificação Consolidada — Aguardando Aprovação para Etapa 2B.4.3  
**Data de Consolidação:** 2026-11-21  
**Escopo da Etapa:** Somente documentação e auditoria técnica (nenhum arquivo de código ou configuração foi modificado).

---

## A. Resumo Executivo

### O que foi revisado
Esta etapa consolidou a especificação da futura rota `GET /api/games/central` proposta na Etapa 2B.4.1, confrontando-a diretamente com o código real em produção/shadow:
- `src/server/collectors/unifiedGamesCollector.ts`
- `src/server/registerCollectorRoutes.ts`
- `src/server/collectors/bzzoiroCollector.ts`
- `src/server/collectors/espnAmericanCollector.ts`
- `src/server/collectors/espnLacrosseCollector.ts`
- `src/server/collectors/sportsblazeAmericanCollector.ts`
- `src/utils/brtDate.ts` e `src/utils/gameSets.ts`
- `server.ts` (handler `/api/match-stats`) e consumidores do frontend (`JogosDoDiaPage`, `Ticker`, `HeroCommandCenter`, `HeroCopa`, `HomeLite`, `MeusTimesPage`, `MatchStatsService`).

> **Nota sobre referência documental:** O arquivo `temp/auditoria-2b3-1.md` citado no briefing não estava presente no sistema de arquivos do repositório durante a inspeção; toda a reconciliação e correção foi executada comparando diretamente a especificação da Etapa 2B.4.1, os pontos de auditoria descritos na Etapa 2B.4.2 e o código-fonte atual.

### Principais correções realizadas
1. **Frescor temporal e proveniência de cache por provedor (Seção 5.1):** `generatedAt` passa a representar estritamente a geração do snapshot no agregador. Cada provedor em `providerStatus` ganha o bloco `freshness` (`collectorTtlMs`, `collectorFetchedAt`, `sourceCacheAgeMs`, `sourceCacheHit`, `feedUpdatedAt`), com regra rígida de **nunca fabricar timestamps** quando o coletor subjacente ainda não os expõe (`null` explícito no estado atual).
2. **Correção do exemplo e da semântica de falha parcial (Seção 5.2):** corrigida a contradição do Exemplo 6 da Etapa 2B.4.1 (em que a ESPN estava saudável com 19 jogos, mas os arrays `sets.*` apareciam vazios). Em falha parcial, os jogos dos provedores saudáveis são entregues normalmente e todos os contadores (`counts.returned` e `counts.snapshotTotal`) correspondem exatamente ao tamanho dos arrays.
3. **Eliminação da inconsistência `stats.ref` (Seção 5.3):** removida qualquer menção a `stats.ref`. Os campos planos de referência (`available`, `matchStatsId`, `endpoint`, `hasTeamStats`, `hasPlayerStats`) permanecem sempre no objeto `stats`, inclusive com `includeStats=none`.
4. **Regra inequívoca de `stats.available` e proteção de Lacrosse/SportsBlaze (Seção 5.4):** `stats.available = true` exige compatibilidade real e funcional com `/api/match-stats`. Partidas `sportsblaze_only` (UUID) e `lacrosse` (`lax-espn-*`, sem suporte em `/api/match-stats`) recebem `stats.available = false` e `matchStatsId = null`.
5. **Reconciliação de limites do estado atual:** documentados os filtros reais suportados sem criação artificial de campos inexistentes e formalizado o modelo de cache em memória por processo (sem prometer coerência entre réplicas múltiplas no Render).

### Decisões finais consolidadas
- **`sets` padrão:** `today,overnight,tomorrow` (obedecendo estritamente `brtDate.ts` e `gameSets.ts`; `OVERNIGHT` permanece auxiliar e não entra em `counts.*.today`).
- **`includeStats` padrão:** `summary`.
- **`includeCompat` padrão:** `false`.
- **Partidas `sportsblaze_only`:** preservam `providerIds.sportsblazeId`, nunca fabricam `espnId`, mantêm `matchStatsId = null` e `stats.available = false`.

---

## B. Diagnóstico de Consistência

### 1. Divergências identificadas entre a auditoria/proposta 2B.4.1 e o código atual

| Item | O que constava na proposta/auditoria 2B.4.1 | Estado real verificado no código | Resolução adotada no contrato |
|---|---|---|---|
| **Frescor dos coletores** | Envelope tinha apenas `generatedAt`, `cache.ageMs` do agregador e `feedUpdatedAt` (SportsBlaze). Não distinguia idade do cache de cada coletor. | `bzzoiroCollector` (180s), `espnAmericanCollector` (60s) e `espnLacrosseCollector` (60s) armazenam `{ timestamp }` internamente em cache, mas **não retornam** `fromCache` nem `fetchedAt`. Apenas `sportsblazeAmericanCollector` retorna `fromCache` e `updated`. | Adicionado `providerStatus[p].freshness` com campos nuláveis (`collectorFetchedAt: null`, `sourceCacheAgeMs: null`, `sourceCacheHit: null` quando o coletor ainda não os expõe). Na Etapa 2B.4.3, os coletores podem passar a retornar `fetchedAtMs`/`fromCache` sem quebra de contrato. |
| **Exemplo 6 (Falha Parcial)** | `counts.snapshotTotal.today = 11`, `tomorrow = 8`, ESPN `dedupedItemCount = 19`, porém `sets.today = []` e `sets.tomorrow = []`. | Em `unifiedGamesCollector.ts:2891-2984`, se qualquer provedor tiver `succeededCalls > 0`, o agregador monta e retorna normalmente os jogos dos provedores saudáveis com `ok: true` e `degraded: true`. | Exemplo corrigido para incluir partidas reais nos arrays de `sets`, com `counts.returned` e `counts.snapshotTotal` idênticos às quantidades presentes. |
| **`stats.ref` vs campos planos** | Prose mencionava `stats.ref` para `includeStats=none`, mas a interface `CentralGameMatch.stats` tinha campos planos (`available`, `matchStatsId`, `endpoint`, `hasTeamStats`, `hasPlayerStats`). | `stats.ref` nunca existiu no código. | Removida qualquer menção a `stats.ref`. Os 5 campos de referência planos são sempre retornados em todos os modos de `includeStats`. |
| **`stats.available` em Lacrosse** | Proposta anterior assumia `lacrosse-<espnId>` como utilizável ou neutro. | Em `server.ts:1699-2001`, `/api/match-stats` verifica `isAmerican` (`NFL`, `NCAAF`, `MLB`, `NBA`, `WNBA`, `NHL`) e depois cai no bloco `2. Football match (BSD)`, extraindo `\d+` de qualquer ID e gerando até fallback de futebol para jogos `LIVE`/`FINAL`! Logo, `lacrosse-<id>` **não funciona** em `/api/match-stats`. | Em `stats`, jogos de Lacrosse têm `available: false`, `matchStatsId: null` e `endpoint: null`. O legado `lacrosse-<espnId>` fica restrito a `compat.statsMatchId` apenas se `includeCompat=true`. |
| **Multiplicidade de chamadas por provedor** | Tratava cada provedor como uma única consulta temporal. | `unifiedGamesCollector.ts` dispara **3** chamadas ao Bzzoiro (ontem, hoje, amanhã), **21** à ESPN (`6` ligas x `3` dias + `3` lacrosse) e **16–20** à SportsBlaze (`4` ligas x `3` boxscores + `4..8` schedules). | `freshness` no nível do provedor consolida a janela de chamadas (detalhado na Seção C.5), explicando que diferentes subchamadas de um mesmo provedor podem ter idades diferentes. |
| **`fieldProvenance.clockPeriod` em ESPN sem clock/period** | Auditoria assumia `source: "none"` quando ausente. | Em `unifiedGamesCollector.ts:1114-1117`, `buildAmericanSingleEspnUnified` preenche `clockPeriod` com `source: "espn"` mesmo quando `period` e `clock` são `undefined` (mantendo `reason`). | Documentado como peculiaridade do Shadow preservada em `provenance` sem afetar `liveOverlay.period = null` nem `liveOverlay.clock = null`. |
| **`league.country` em americanos** | Exemplo 1 da 2B.4.1 trazia `country: "USA"`. | `buildAmericanSingleEspnUnified` e `buildAmericanReconciledUnified` não preenchem `fixture.country` (`undefined`); apenas `buildFootballBzzoiroUnified` preenche `country`, `leagueId`, `leagueFlag` e `leagueLogo`. | Na interface definitiva, `league.country`, `league.flag`, `league.logo` e `league.id` são `null` quando não fornecidos pelo provedor (não inventar `"USA"` onde o coletor retorna `null`). |
| **`sport` / `league` no filtro e `includeStats=full`** | Listava `teamStatistics` e `players` via `includeStats=full` sem destacar quais provedores os possuem. | Somente **SportsBlaze** (NFL, MLB, NBA, WNBA — boxscores) traz `teamStatistics` e `players` em memória. ESPN (incluindo NCAAF/NHL/Lacrosse) e Bzzoiro **não** possuem boxscores de atletas/equipe na coleta central. | Limitação formalmente documentada: `includeStats=full` expõe `teamStatistics`/`players` apenas quando já existentes em memória (SportsBlaze), retornando `null` nos demais casos (sem chamadas extras). |

### 2. Limitações estruturais ainda existentes no estado atual
1. **Cache em memória por processo Node.js:** tanto os coletores quanto o agregador usam `Map` / objetos em memória do processo. Em deploy multi-instância (ex.: múltiplas réplicas no Render), cada instância possui seu próprio cache e `snapshotId` independente.
2. **Ausência de `fetchedAt` / `fromCache` nos retornos de Bzzoiro e ESPN:** embora já guardem `timestamp` internamente, suas interfaces públicas ainda não retornam esses metadados para o agregador.
3. **Ausência de `eventUpdatedAt` por partida nos provedores atuais:** Bzzoiro, ESPN scoreboard e SportsBlaze listagem diária não fornecem timestamp individual confiável da última atualização do evento (SportsBlaze fornece apenas `updated` no nível do feed). Portanto, `liveOverlay.eventUpdatedAt` será `null` para a maioria dos eventos até que provedores passem a enviá-lo.
4. **Estatísticas de Futebol, NCAAF, NHL e Lacrosse:** não vêm embutidas na coleta diária; Futebol e Americanos com `espnId` dependem de `/api/match-stats`, enquanto Lacrosse e `sportsblaze_only` não possuem consulta on-demand funcional hoje.

---

## C. Contrato HTTP Definitivo

### 1. Endpoint e Métodos
- **URL:** `/api/games/central`
- **Métodos permitidos:** `GET` e `HEAD`
- **Demais métodos (`POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS` se aplicável):** `HTTP 405 Method Not Allowed` com header `Allow: GET, HEAD` e JSON de erro padrão.
- **Comportamento `HEAD`:** executa a mesma validação e resolução de snapshot/ETag, retornando apenas os headers (`200` ou `304`) sem corpo.
- **Princípio de execução:** rota estritamente somente leitura. Toda filtragem e projeção (`sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeProvenance`, `includeCompat`) ocorre **em memória** após obter o snapshot do agregador para `referenceDate`. Nenhum filtro dispara chamadas extras aos provedores externos.

### 2. Parâmetros de Consulta (Query Parameters)

| Parâmetro | Tipo / Domínio aceito | Padrão | Regra de validação e erro |
|---|---|---|---|
| `date` | `YYYY-MM-DD` em `America/Sao_Paulo` (validado por `normalizeBrtDateIso`) | Data atual em BRT (`getBrtTodayStr()`) | String vazia equivale a omitido (usa hoje). Formato inválido ou data inexistente (`2026-02-30`, `2026/11/20`) -> `HTTP 400` (`INVALID_DATE`). |
| `sets` | Lista separada por vírgula entre `today`, `overnight`, `tomorrow` (case-insensitive) ou `all` | `today,overnight,tomorrow` | `all` expande para `["today","overnight","tomorrow"]`. Token vazio (ex.: `sets=today,`) ou desconhecido -> `HTTP 400` (`INVALID_SETS`). Chaves não solicitadas retornam `[]` no objeto `sets` do JSON. |
| `sport` | Lista separada por vírgula. Canônicos: `football`, `basketball`, `american_football`, `baseball`, `hockey`, `lacrosse`. Aliases aceitos: `futebol`->`football`, `nba`/`wnba`->`basketball`, `nfl`/`ncaaf`->`american_football`, `mlb`->`baseball`, `nhl`->`hockey`, `pll`->`lacrosse`. | `null` (todas as modalidades) | Case-insensitive. Qualquer valor não reconhecido -> `HTTP 400` (`INVALID_SPORT`). Máx. 20 itens. |
| `league` | Lista separada por vírgula (case-insensitive). Aceita código (`nba`, `nfl`, `ncaaf`, `mlb`, `nhl`, `wnba`, `pll`), `leagueId` numérico de futebol (`9`, `1`, `34`) ou nome/slug da liga normalizado (ex.: `premier league`, `brasileirao serie a`). | `null` (todas as ligas) | Match em memória contra `league.code`, `league.name` (sem acentos, case-insensitive) ou `String(league.id)`. Sintaticamente válido sem jogos ativos retorna `[]` (`200 OK`). Token vazio, caracteres de controle ou > 20 itens -> `HTTP 400` (`INVALID_LEAGUE`). |
| `status` | Lista separada por vírgula: `LIVE`, `UPCOMING`, `FINAL`, `POSTPONED` (case-insensitive). Aliases: `AO VIVO`->`LIVE`, `AGENDADO`->`UPCOMING`, `ENCERRADO`/`FIM`->`FINAL`, `ADIADO`/`PP`->`POSTPONED`. | `null` (todos os status) | Qualquer token fora da lista -> `HTTP 400` (`INVALID_STATUS`). Máx. 10 itens. |
| `liveOnly` | `true` \| `false` \| `1` \| `0` (case-insensitive) | `false` | Se `true`/`1`, restringe aos jogos com `liveOverlay.status === "LIVE"`. Qualquer outro valor (ex.: `yes`, `2`) -> `HTTP 400` (`INVALID_BOOLEAN`). |
| `includeStats` | `none` \| `summary` \| `full` (case-insensitive) | `summary` | Qualquer outro valor -> `HTTP 400` (`INVALID_INCLUDE_STATS`). |
| `includeCompat` | `true` \| `false` \| `1` \| `0` (case-insensitive) | `false` | Qualquer outro valor -> `HTTP 400` (`INVALID_BOOLEAN`). |
| `includeProvenance` | `true` \| `false` \| `1` \| `0` (case-insensitive) | `false` | Qualquer outro valor -> `HTTP 400` (`INVALID_BOOLEAN`). |

**Regras adicionais de parâmetros e limites:**
- Parâmetros desconhecidos na query string são ignorados (não quebram clientes por parâmetros de cache-busting como `_t`, mas **não** devem variar o cache interno do agregador).
- Limites de segurança por parâmetro de lista: máximo **20 tokens**, máximo **64 caracteres** por token, rejeição de caracteres de controle (`[\x00-\x1F\x7F]`).
- Combinação de filtros: `OR` dentro de um mesmo parâmetro multivalorado (ex.: `league=nba,nhl`); `AND` entre parâmetros distintos (ex.: `sport=basketball&liveOnly=true`). Se `liveOnly=true` e `status=FINAL`, a interseção lógica é vazia (`sets` retornam `[]` com `200 OK`).

### 3. Regras de Classificação, Deduplicação e Ordenação

1. **Partição oficial (`src/utils/gameSets.ts` + `src/utils/brtDate.ts`):**
   - `TODAY`: `startDateBrt === referenceDate`.
   - `OVERNIGHT`: iniciadas em `yesterdayBrt` e ainda `LIVE`, ou `FINAL` com encerramento após `00:00` BRT respaldado por timestamp explícito confiável de término (`endDate`/`endTime`/`finishedAt`/`completedAt`). **Permanecem exclusivamente em `sets.overnight` e nunca somam em `counts.*.today`.**
   - `TOMORROW`: `startDateBrt === tomorrowBrt`.
   - `excluded`: partidas que não pertencem a nenhum dos 3 conjuntos (`NONE`). Não são retornadas em `/api/games/central` (diferente de `/api/games/shadow`, onde aparecem para auditoria); apenas seu total agregado pode ser ignorado ou não interfere em `counts.snapshotTotal`.
2. **Deduplicação:** garantida por `canonicalId` em `partitionGamesIntoSets` e nas etapas de deduplicação por provedor (`bzzDeduped`, `espnAmericanByLeague`, `espnLacrosseDeduped`, `sbAmericanByLeague`).
3. **Ordenação determinística dentro de cada conjunto (`today`, `overnight`, `tomorrow`):**
   1. Peso de status: `LIVE` (`0`) -> `UPCOMING` (`1`) -> `FINAL` (`2`) -> `POSTPONED` (`3`).
   2. Horário de início em BRT (`fixture.startTimeBrt` / `fixture.startTimeUtc`) ascendente.
   3. Desempate final por `canonicalId` ascendente (`localeCompare`).

### 4. Semântica de Respostas HTTP e Falhas

| Cenário | HTTP | `ok` | `degraded` | Comportamento detalhado |
|---|---|---|---|---|
| **Todos os provedores saudáveis** (com ou sem jogos no dia) | `200` | `true` | `false` | Resposta completa. Slate vazio (ex.: data sem jogos ou `404` de boxscore futuro da SportsBlaze) **é resposta válida vazia**, nunca erro. |
| **1 provedor falha** (ex.: Bzzoiro fora do ar; ESPN e SportsBlaze respondem) | `200` | `true` | `true` | Jogos dos provedores saudáveis são retornados normalmente em `sets`. `providerStatus.bzzoiro.ok = false`; `warnings` e `errors` registram a falha. |
| **Vários provedores falham, ao menos 1 responde** (ex.: Bzzoiro e SportsBlaze falham; ESPN responde) | `200` | `true` | `true` | Jogos do provedor saudável permanecem disponíveis em `sets`. `providerStatus` aponta exatamente quais falharam. |
| **1 provedor entrega cache do coletor enquanto outro falha** | `200` | `true` | `true` | O resultado em cache do provedor saudável compõe normalmente os `sets` (`freshness.sourceCacheHit = true` quando reportado), enquanto o provedor com falha é marcado como `ok: false`. |
| **Todos os provedores falham** (`anyProviderSucceeded === false`) | `503` | `false` | `true` | `sets` retornam vazios, contadores zerados, `errors` contém `ALL_PROVIDERS_FAILED` + detalhes seguros por provedor. **Nunca é gravado no cache do agregador.** |
| **Parâmetro inválido** | `400` | `false` | `false` | Envelope enxuto de erro (`{ contractVersion, ok: false, error: { code, message, param } }`). |
| **Método HTTP inválido** | `405` | `false` | `false` | Header `Allow: GET, HEAD`. |
| **Validação condicional (`If-None-Match` bate com `ETag`)** | `304` | — | — | Corpo vazio; retorna `ETag`, `Cache-Control`, `Vary`. |

### 5. Modelo de Frescor Temporal (`generatedAt` vs `collectorFetchedAt` vs `eventUpdatedAt`)

Para não confundir **geração do snapshot**, **consulta ao provedor** e **atualização do evento**, o contrato separa 3 níveis distintos:

1. **Nível do Snapshot (Envelope raiz + `cache`):**
   - `generatedAt`: timestamp ISO UTC em que o agregador montou o snapshot em memória.
   - `cache.hit`, `cache.ageMs`, `cache.ttlMs` (`60000`): refletem exclusivamente o cache do **agregador** (`unified:central:${referenceDate}`).
2. **Nível de Cada Provedor (`providerStatus[provider].freshness`):**
   - `collectorTtlMs`: TTL configurado no coletor (`180000` Bzzoiro; `60000` ESPN; `{ boxscores: 60000, schedule: 300000 }` SportsBlaze).
   - `feedUpdatedAt`: timestamp informado pelo próprio provedor quando existente (`sb.updated` na SportsBlaze; `null` em Bzzoiro e ESPN scoreboard).
   - `collectorFetchedAt`: timestamp ISO UTC da efetiva obtenção dos dados no coletor subjacente. Em consultas com múltiplas subchamadas na janela (`yesterday..today..tomorrow`), representa o timestamp **mais antigo** (pior caso de frescor) entre as subchamadas bem-sucedidas que compuseram o provedor — ou `null` enquanto o coletor não expuser esse campo.
   - `sourceCacheAgeMs`: `Math.max(0, nowMs - oldestFetchedAtMs)`, ou `null` quando indisponível no coletor.
   - `sourceCacheHit`: `true` se ao menos uma subchamada do provedor reutilizou cache do coletor; `false` se todas foram coletadas nesta execução; `null` quando o coletor não expõe `fromCache` (estado atual de Bzzoiro e ESPN).
   - **Nota evolutiva:** no estado atual sem alterações nos coletores, apenas SportsBlaze possui `fromCache` e `feedUpdatedAt` no retorno; Bzzoiro e ESPN terão `collectorFetchedAt: null`, `sourceCacheAgeMs: null` e `sourceCacheHit: null` até que a Etapa 2B.4.3 opcionalmente exponha o `timestamp` já armazenado nos respectivos `Map`/objetos de cache — transição 100% retrocompatível (`null` -> valor concreto).
3. **Nível do Evento (`liveOverlay.eventUpdatedAt`):**
   - Timestamp explícito da última mutação **daquela partida** (ex.: `liveState.updatedAt` ou `espnLiveDetail.updatedAt`), ou `null` quando ausente. **Proibido** copiar `generatedAt`, `collectorFetchedAt`, `feedUpdatedAt` (`sb.updated`) ou `startTime` para `eventUpdatedAt` ou `completedAt`.

### 6. Cache, Coalescing, ETag e Limites de Infraestrutura

- **Camada 1 — Coletores internos:** mantêm seus caches e *in-flight coalescing* existentes (`bzzoiroCollector`: 180s; `espnAmericanCollector`: 60s; `espnLacrosseCollector`: 60s; `sportsblazeAmericanCollector`: 60s boxscores / 300s schedule).
- **Camada 2 — Agregador Central:**
  - Chave de cache: `unified:central:${referenceDate}` com TTL de **60 segundos** (`60000 ms`) e *in-flight coalescing* para requisições simultâneas na mesma data.
  - **Cache degradado (`ok: true, degraded: true`):** é armazenado em cache com **TTL reduzido de 20 segundos** (`20000 ms`) — ou, mantendo a regra atual do Shadow caso não seja parametrizado TTL distinto, no máximo 60s — para permitir recuperação mais rápida de provedor que caiu transitoriamente sem martelar APIs a cada requisição. (Preferencial: `degradedTtlMs = 20000`, explícito em `cache.ttlMs`).
  - **Falha total (`ok: false`, HTTP `503`):** **nunca** é armazenada em cache.
  - **Invalidação:** expiração por TTL ou limpeza explícita administrativa/testes (`clearUnifiedGamesCache`). Não há invalidação por query de filtro, pois filtros operam sobre o snapshot em cache.
- **Camada 3 — Cache HTTP e `ETag`:**
  - `ETag` fraco determinístico baseado no `snapshotId` e na assinatura canônica dos filtros aplicados:  
    `W/"${snapshotId}:${filtersHash}"` (ex.: `W/"ug-2026-11-20-1795224900000:a1b2c3d4"`).
  - Quando o cliente envia `If-None-Match` contendo essa mesma tag (comparação após remover prefixo `W/` conforme RFC 9110 ou comparação exata) e o snapshot ainda está válido em cache, o servidor responde **`304 Not Modified`** com `ETag`, `Cache-Control` e `Vary: Accept-Encoding`.
  - **Headers `Cache-Control`:**
    - `200` saudável (`degraded: false`): `Cache-Control: public, max-age=15, must-revalidate` (mantém frescor próximo ao tempo real sem depender de cache longo no navegador/CDN e nunca excede o TTL de 60s do snapshot).
    - `200` degradado (`degraded: true`): `Cache-Control: public, max-age=10, must-revalidate`.
    - `503` (`ok: false`) ou `400`: `Cache-Control: no-store`.
  - **Diferença entre cache interno e cache HTTP:** o cache interno guarda o snapshot unificado por 60s em RAM (compartilhado por todos os consumidores e combinações de filtros); o cache HTTP controla apenas a revalidação condicional (`304`) e o reaproveitamento curto (`10s–15s`) da resposta HTTP específica para aquele conjunto de query params.
- **Limitação explícita de multi-instância (Render):** como o armazenamento atual dos coletores e do agregador é **em memória por processo Node.js**, em cenários com múltiplas instâncias/workers atrás de balanceador:
  - Cada processo mantém seu próprio `snapshotId` e idade de cache;
  - Uma requisição subsequente com `If-None-Match` que caia em outra réplica pode gerar `200 OK` em vez de `304` caso os `snapshotId`s divirjam. Essa limitação só seria eliminada com cache distribuído externo (ex.: Redis), fora do escopo atual.

---

## D. Interfaces TypeScript Definitivas

```ts
export type CentralContractVersion = "1.0";
export type ProviderId = "bzzoiro" | "espn" | "sportsblaze";
export type CentralSetQueryKey = "today" | "overnight" | "tomorrow";
export type CentralGameSetValue = "TODAY" | "OVERNIGHT" | "TOMORROW";

export type CanonicalSport =
  | "football"
  | "basketball"
  | "american_football"
  | "baseball"
  | "hockey"
  | "lacrosse";

export type CanonicalSportFamily = "football" | "american" | "lacrosse";
export type CentralMatchStatus = "LIVE" | "FINAL" | "UPCOMING" | "POSTPONED";
export type IncludeStatsMode = "none" | "summary" | "full";
export type ReconciliationMode = "matched" | "espn_only" | "sportsblaze_only" | "bzzoiro_only";

export interface ProviderFreshnessMetadata {
  /** TTL configurado no coletor (ou por tipo de consulta na SportsBlaze) */
  collectorTtlMs: number | { boxscores: number; schedule: number };
  /** Timestamp informado pelo provedor no feed (ex.: sb.updated). null quando não fornecido */
  feedUpdatedAt: string | null;
  /** Timestamp ISO UTC da consulta efetiva mais antiga usada na janela. null se coletor não expõe */
  collectorFetchedAt: string | null;
  /** Idade em ms desde collectorFetchedAt até o instante da resposta. null se indisponível */
  sourceCacheAgeMs: number | null;
  /** true se alguma subchamada reutilizou cache do coletor; false se todas foram live fetch; null se não reportado */
  sourceCacheHit: boolean | null;
}

export interface CentralProviderStatus {
  ok: boolean;
  partial: boolean;
  totalCalls: number;
  succeededCalls: number;
  failedCalls: number;
  emptySlateCalls: number;
  rawItemCount: number;
  dedupedItemCount: number;
  freshness: ProviderFreshnessMetadata;
}

export interface CentralApiErrorItem {
  code: string;
  message: string;
  provider?: ProviderId;
  scope?: string;
  statusCode?: number;
}

export interface CentralGamesResponse {
  contractVersion: CentralContractVersion;
  ok: boolean;
  degraded: boolean;
  /** Identificador determinístico da geração em cache no processo servidor */
  snapshotId: string;
  /** ISO-8601 UTC da construção do snapshot agregado */
  generatedAt: string;
  /** Data de referência esportiva YYYY-MM-DD em America/Sao_Paulo */
  referenceDate: string;
  timezone: "America/Sao_Paulo";
  window: {
    yesterday: string;
    today: string;
    tomorrow: string;
  };
  cache: {
    hit: boolean;
    ageMs: number;
    ttlMs: number;
  };
  filters: {
    sets: CentralSetQueryKey[];
    sport: CanonicalSport[] | null;
    league: string[] | null;
    status: CentralMatchStatus[] | null;
    liveOnly: boolean;
    includeStats: IncludeStatsMode;
    includeCompat: boolean;
    includeProvenance: boolean;
  };
  counts: {
    /** Quantidade exata retornada em `sets` após aplicação dos filtros */
    returned: {
      today: number;
      overnight: number;
      tomorrow: number;
      total: number;
    };
    /** Quantidade total disponível no snapshot antes dos filtros */
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
  providerStatus: Record<ProviderId, CentralProviderStatus>;
  warnings: string[];
  errors: CentralApiErrorItem[];
  sets: {
    today: CentralGameMatch[];
    overnight: CentralGameMatch[];
    tomorrow: CentralGameMatch[];
  };
}

export interface CentralErrorResponse {
  contractVersion: CentralContractVersion;
  ok: false;
  error: {
    code: string;
    message: string;
    param?: string;
  };
}

export interface CentralTeamFixture {
  /** ID do time no provedor quando disponível (ex.: SportsBlaze team.id), ou null */
  id: string | null;
  name: string;
  abbreviation: string;
  /** URL do escudo/logo ou ícone (ex.: "🥍" em Lacrosse); "" se ausente (sportsblaze_only) */
  logo: string;
}

export interface CentralLeagueInfo {
  /** Sigla/código canônico (ex.: "NBA", "NFL", "PLL") ou nome da liga no futebol */
  code: string;
  name: string;
  /** ID numérico Bzzoiro no futebol; null nas demais modalidades */
  id: number | null;
  /** País da liga quando informado pelo coletor (Bzzoiro); caso contrário null */
  country: string | null;
  flag: string | null;
  logo: string | null;
}

export interface CentralGameFixture {
  /** ISO-8601 UTC normalizado quando parseável; null se ausente ou inválido */
  startTimeUtc: string | null;
  /** String bruta original do provedor */
  rawDate: string;
  timestampSource: ProviderId;
  /** YYYY-MM-DD em America/Sao_Paulo */
  startDateBrt: string | null;
  /** HH:mm em BRT ou "" se indisponível */
  startTimeBrt: string;
  /** DD/MM/YYYY em BRT ou "" se indisponível */
  dateDisplay: string;
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
}

export interface CentralLiveOverlay {
  status: CentralMatchStatus;
  live: boolean;
  score: {
    /** Sempre null quando status é UPCOMING ou POSTPONED */
    home: number | null;
    /** Sempre null quando status é UPCOMING ou POSTPONED */
    away: number | null;
  };
  /** Badge formatado autoritativo (ex.: "2:19 - 2nd", "41'", "FIM", "ADIADO", "21:30") */
  timeBadge: string;
  /** Inteiro >= 0..25 quando disponível e não pré-jogo/adiado; caso contrário null */
  period: number | null;
  /** Relógio validado ("M:SS", "SS.t" ou minuto "41'") ou null durante intervalo/pré-jogo/fim */
  clock: string | null;
  tempo: string;
  liveState: {
    period?: number;
    clock?: string;
    down?: number;
    distance?: number;
    possession?: string;
    yard_line?: number;
    timeouts?: {
      away?: number;
      home?: number;
    };
  } | null;
  linescores: {
    header: string[];
    home: Array<string | number>;
    away: Array<string | number>;
    /** Presente quando SportsBlaze fornece periodScores (opcional) */
    periodScores?: Record<string, { away?: number; home?: number }>;
  };
  /** Gols formatados em futebol; null para americanos e lacrosse */
  incidents: {
    homeGoals: string[];
    awayGoals: string[];
  } | null;
  /** Timestamp explícito de atualização do evento, ou null se provedor não fornece */
  eventUpdatedAt: string | null;
}

export interface CentralMatchStatsBlock {
  /**
   * true SOMENTE quando existe endpoint funcional compatível para consulta on-demand
   * (`/api/match-stats` com ID Bzzoiro de futebol ou ID numérico ESPN de esporte americano).
   * Sempre false para `sportsblaze_only` e `lacrosse`.
   */
  available: boolean;
  /** ID compatível com `/api/match-stats` ("bsd-<id>" ou "sb-<league>-<espnId>"); caso contrário null */
  matchStatsId: string | null;
  /** Caminho relativo pronto para consulta quando available=true; caso contrário null */
  endpoint: string | null;
  /** Indica se há teamStatistics já carregadas em memória no snapshot (atualmente apenas SportsBlaze) */
  hasTeamStats: boolean;
  /** Indica se há players boxscores já carregados em memória no snapshot (atualmente apenas SportsBlaze) */
  hasPlayerStats: boolean;
  /** Incluído apenas quando includeStats="summary" e há atletas disponíveis em memória */
  keyPlayers?: {
    home: Array<{ name: string; pos?: string; stat?: string }>;
    away: Array<{ name: string; pos?: string; stat?: string }>;
  };
  /** Incluído apenas quando includeStats="full"; null quando não disponível em memória */
  teamStatistics?: {
    away?: Record<string, unknown>;
    home?: Record<string, unknown>;
  } | null;
  /** Incluído apenas quando includeStats="full"; null quando não disponível em memória */
  players?: {
    away?: Array<{
      id?: string;
      name: string;
      position?: string;
      played?: boolean;
      starter?: boolean;
      statistics?: Record<string, unknown>;
    }>;
    home?: Array<{
      id?: string;
      name: string;
      position?: string;
      played?: boolean;
      starter?: boolean;
      statistics?: Record<string, unknown>;
    }>;
  } | null;
}

export interface CentralMatchProvenance {
  reconciliationDeltaMinutes: number | null;
  rawStatusByProvider: Partial<Record<ProviderId, string>>;
  fieldProvenance: Record<
    | "startTime"
    | "status"
    | "score"
    | "timeBadge"
    | "period"
    | "clock"
    | "liveState"
    | "linescores"
    | "clockPeriod"
    | "teamStatistics"
    | "players"
    | "venueBroadcaster"
    | "teamLogos",
    {
      source: ProviderId | "fallback" | "none";
      fallbackUsed: boolean;
      reason?: string;
      detail?: string;
      dataAgeMs?: number;
    }
  >;
  divergences: Array<{
    field:
      | "score"
      | "status"
      | "startTime"
      | "period"
      | "clock"
      | "timeBadge"
      | "linescores"
      | "venue"
      | "teamName";
    espnValue: unknown;
    sportsblazeValue: unknown;
    selectedSource: ProviderId | "none";
    reason?: string;
    detail: string;
  }>;
}

export interface CentralMatchLegacyCompat {
  /** ID compatível com componentes/rotas legadas ("bsd-<id>", "sb-<league>-<espnId>", "sb-<league>-sb_<uuid>", "lacrosse-<id>") */
  legacyId: string;
  tickerId: string;
  /** Preservado apenas para compatibilidade histórica com heroCompat; consumidores novos devem usar stats.matchStatsId */
  statsMatchId: string | null;
  apiId: string | null;
  heroStatus: "LIVE" | "FINAL" | "UPCOMING" | "ADIADO";
}

export interface CentralGameMatch {
  canonicalId: string;
  set: CentralGameSetValue;
  sport: CanonicalSport;
  sportFamily: CanonicalSportFamily;
  league: CentralLeagueInfo;
  providerIds: {
    espnId: string | null;
    sportsblazeId: string | null;
    bzzoiroId: string | null;
    lacrosseCollectorId: string | null;
  };
  providers: ProviderId[];
  reconciled: boolean;
  reconciliationMode: ReconciliationMode;
  fixture: CentralGameFixture;
  liveOverlay: CentralLiveOverlay;
  stats: CentralMatchStatsBlock;
  /** Presente somente quando includeProvenance=true */
  provenance?: CentralMatchProvenance;
  /** Presente somente quando includeCompat=true */
  compat?: CentralMatchLegacyCompat;
}
```

---

## E. Exemplos de Payload

### 1. Resposta normal com `TODAY`, `OVERNIGHT` e `TOMORROW` (`GET /api/games/central?date=2026-11-20`)

```json
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
    "includeStats": "summary",
    "includeCompat": false,
    "includeProvenance": false
  },
  "counts": {
    "returned": { "today": 2, "overnight": 1, "tomorrow": 1, "total": 4 },
    "snapshotTotal": { "today": 2, "overnight": 1, "tomorrow": 1, "total": 4 },
    "live": { "returned": 3, "snapshotTotal": 3 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": true,
      "partial": false,
      "totalCalls": 3,
      "succeededCalls": 3,
      "failedCalls": 0,
      "emptySlateCalls": 1,
      "rawItemCount": 1,
      "dedupedItemCount": 1,
      "freshness": {
        "collectorTtlMs": 180000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "espn": {
      "ok": true,
      "partial": false,
      "totalCalls": 21,
      "succeededCalls": 21,
      "failedCalls": 0,
      "emptySlateCalls": 18,
      "rawItemCount": 3,
      "dedupedItemCount": 3,
      "freshness": {
        "collectorTtlMs": 60000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "sportsblaze": {
      "ok": true,
      "partial": false,
      "totalCalls": 16,
      "succeededCalls": 16,
      "failedCalls": 0,
      "emptySlateCalls": 14,
      "rawItemCount": 2,
      "dedupedItemCount": 2,
      "freshness": {
        "collectorTtlMs": { "boxscores": 60000, "schedule": 300000 },
        "feedUpdatedAt": "2026-11-21T01:34:42.000Z",
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": false
      }
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
          "country": null,
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
            "logo": "https://a.espncdn.com/i/teamlogos/soccer/500/819.png"
          },
          "awayTeam": {
            "id": null,
            "name": "Palmeiras",
            "abbreviation": "PAL",
            "logo": "https://a.espncdn.com/i/teamlogos/soccer/500/2029.png"
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
        "league": { "code": "NBA", "name": "NBA", "id": null, "country": null, "flag": null, "logo": null },
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
        "league": { "code": "NFL", "name": "NFL", "id": null, "country": null, "flag": null, "logo": null },
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

### 2. Resposta válida sem partidas (`HTTP 200`, `ok: true`, `degraded: false`)

```json
{
  "contractVersion": "1.0",
  "ok": true,
  "degraded": false,
  "snapshotId": "ug-2026-07-15-1784131200000",
  "generatedAt": "2026-07-15T15:00:00.000Z",
  "referenceDate": "2026-07-15",
  "timezone": "America/Sao_Paulo",
  "window": {
    "yesterday": "2026-07-14",
    "today": "2026-07-15",
    "tomorrow": "2026-07-16"
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
    "includeStats": "summary",
    "includeCompat": false,
    "includeProvenance": false
  },
  "counts": {
    "returned": { "today": 0, "overnight": 0, "tomorrow": 0, "total": 0 },
    "snapshotTotal": { "today": 0, "overnight": 0, "tomorrow": 0, "total": 0 },
    "live": { "returned": 0, "snapshotTotal": 0 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": true,
      "partial": false,
      "totalCalls": 3,
      "succeededCalls": 3,
      "failedCalls": 0,
      "emptySlateCalls": 3,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": 180000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "espn": {
      "ok": true,
      "partial": false,
      "totalCalls": 21,
      "succeededCalls": 21,
      "failedCalls": 0,
      "emptySlateCalls": 21,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": 60000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "sportsblaze": {
      "ok": true,
      "partial": false,
      "totalCalls": 16,
      "succeededCalls": 16,
      "failedCalls": 0,
      "emptySlateCalls": 16,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": { "boxscores": 60000, "schedule": 300000 },
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": false
      }
    }
  },
  "warnings": [],
  "errors": [],
  "sets": {
    "today": [],
    "overnight": [],
    "tomorrow": []
  }
}
```

### 3. Falha parcial de provedor (`HTTP 200`, `ok: true`, `degraded: true` — contadores 100% consistentes)

> Cenário: **Bzzoiro falhou** nas 3 chamadas (`ok: false`); **SportsBlaze** falhou em 1 consulta (`partial: true`), usando dados de cache nas demais; **ESPN** respondeu 100% saudável com 2 partidas (`1` em `today` e `1` em `tomorrow`), sendo a de `today` reconciliada com o cache da SportsBlaze.

```json
{
  "contractVersion": "1.0",
  "ok": true,
  "degraded": true,
  "snapshotId": "ug-2026-11-21-1795300000000",
  "generatedAt": "2026-11-21T14:00:00.000Z",
  "referenceDate": "2026-11-21",
  "timezone": "America/Sao_Paulo",
  "window": {
    "yesterday": "2026-11-20",
    "today": "2026-11-21",
    "tomorrow": "2026-11-22"
  },
  "cache": {
    "hit": false,
    "ageMs": 0,
    "ttlMs": 20000
  },
  "filters": {
    "sets": ["today", "overnight", "tomorrow"],
    "sport": null,
    "league": null,
    "status": null,
    "liveOnly": false,
    "includeStats": "summary",
    "includeCompat": false,
    "includeProvenance": false
  },
  "counts": {
    "returned": { "today": 1, "overnight": 0, "tomorrow": 1, "total": 2 },
    "snapshotTotal": { "today": 1, "overnight": 0, "tomorrow": 1, "total": 2 },
    "live": { "returned": 1, "snapshotTotal": 1 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": false,
      "partial": false,
      "totalCalls": 3,
      "succeededCalls": 0,
      "failedCalls": 3,
      "emptySlateCalls": 0,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": 180000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "espn": {
      "ok": true,
      "partial": false,
      "totalCalls": 21,
      "succeededCalls": 21,
      "failedCalls": 0,
      "emptySlateCalls": 19,
      "rawItemCount": 2,
      "dedupedItemCount": 2,
      "freshness": {
        "collectorTtlMs": 60000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "sportsblaze": {
      "ok": true,
      "partial": true,
      "totalCalls": 16,
      "succeededCalls": 15,
      "failedCalls": 1,
      "emptySlateCalls": 14,
      "rawItemCount": 1,
      "dedupedItemCount": 1,
      "freshness": {
        "collectorTtlMs": { "boxscores": 60000, "schedule": 300000 },
        "feedUpdatedAt": "2026-11-21T13:59:10.000Z",
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": true
      }
    }
  },
  "warnings": [
    "provider_unavailable:bzzoiro",
    "provider_partial:sportsblaze"
  ],
  "errors": [
    {
      "code": "PROVIDER_UNAVAILABLE",
      "message": "Bzzoiro collector retornou falha nas 3 datas da janela.",
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
    "today": [
      {
        "canonicalId": "ug:nba:espn:401810200",
        "set": "TODAY",
        "sport": "basketball",
        "sportFamily": "american",
        "league": { "code": "NBA", "name": "NBA", "id": null, "country": null, "flag": null, "logo": null },
        "providerIds": {
          "espnId": "401810200",
          "sportsblazeId": "11112222-3333-4444-5555-666677778888",
          "bzzoiroId": null,
          "lacrosseCollectorId": null
        },
        "providers": ["espn", "sportsblaze"],
        "reconciled": true,
        "reconciliationMode": "matched",
        "fixture": {
          "startTimeUtc": "2026-11-21T13:30:00.000Z",
          "rawDate": "2026-11-21T13:30Z",
          "timestampSource": "espn",
          "startDateBrt": "2026-11-21",
          "startTimeBrt": "10:30",
          "dateDisplay": "21/11/2026",
          "venue": "Madison Square Garden",
          "broadcaster": "ESPN, Disney+",
          "season": { "year": 2026, "type": "Regular Season", "week": null },
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": {
            "id": "sb-team-ny",
            "name": "New York Knicks",
            "abbreviation": "NY",
            "logo": "https://a.espncdn.com/i/teamlogos/nba/500/ny.png"
          },
          "awayTeam": {
            "id": "sb-team-phi",
            "name": "Philadelphia 76ers",
            "abbreviation": "PHI",
            "logo": "https://a.espncdn.com/i/teamlogos/nba/500/phi.png"
          }
        },
        "liveOverlay": {
          "status": "LIVE",
          "live": true,
          "score": { "home": 42, "away": 39 },
          "timeBadge": "5:10 - 2nd",
          "period": 2,
          "clock": "5:10",
          "tempo": "5:10 - 2nd",
          "liveState": { "period": 2, "clock": "5:10" },
          "linescores": {
            "header": ["Q1", "Q2", "Q3", "Q4"],
            "home": [26, 16, "-", "-"],
            "away": [22, 17, "-", "-"]
          },
          "incidents": null,
          "eventUpdatedAt": null
        },
        "stats": {
          "available": true,
          "matchStatsId": "sb-nba-401810200",
          "endpoint": "/api/match-stats?id=sb-nba-401810200&sport=NBA&league=NBA",
          "hasTeamStats": true,
          "hasPlayerStats": true,
          "keyPlayers": {
            "home": [{ "name": "Karl-Anthony Towns", "pos": "C", "stat": "points: 14 · rebounds: 7" }],
            "away": [{ "name": "Tyrese Maxey", "pos": "G", "stat": "points: 15 · assists: 3" }]
          }
        }
      }
    ],
    "overnight": [],
    "tomorrow": [
      {
        "canonicalId": "ug:nfl:espn:401772915",
        "set": "TOMORROW",
        "sport": "american_football",
        "sportFamily": "american",
        "league": { "code": "NFL", "name": "NFL", "id": null, "country": null, "flag": null, "logo": null },
        "providerIds": {
          "espnId": "401772915",
          "sportsblazeId": null,
          "bzzoiroId": null,
          "lacrosseCollectorId": null
        },
        "providers": ["espn"],
        "reconciled": false,
        "reconciliationMode": "espn_only",
        "fixture": {
          "startTimeUtc": "2026-11-22T18:00:00.000Z",
          "rawDate": "2026-11-22T18:00Z",
          "timestampSource": "espn",
          "startDateBrt": "2026-11-22",
          "startTimeBrt": "15:00",
          "dateDisplay": "22/11/2026",
          "venue": "Lambeau Field",
          "broadcaster": "NFL Game Pass / DAZN",
          "season": null,
          "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
          "homeTeam": {
            "id": null,
            "name": "Green Bay Packers",
            "abbreviation": "GB",
            "logo": "https://a.espncdn.com/i/teamlogos/nfl/500/gb.png"
          },
          "awayTeam": {
            "id": null,
            "name": "Chicago Bears",
            "abbreviation": "CHI",
            "logo": "https://a.espncdn.com/i/teamlogos/nfl/500/chi.png"
          }
        },
        "liveOverlay": {
          "status": "UPCOMING",
          "live": false,
          "score": { "home": null, "away": null },
          "timeBadge": "15:00",
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
          "matchStatsId": "sb-nfl-401772915",
          "endpoint": "/api/match-stats?id=sb-nfl-401772915&sport=NFL&league=NFL",
          "hasTeamStats": false,
          "hasPlayerStats": false
        }
      }
    ]
  }
}
```

### 4. Todos os provedores indisponíveis (`HTTP 503 Service Unavailable`, `ok: false`)

```json
{
  "contractVersion": "1.0",
  "ok": false,
  "degraded": true,
  "snapshotId": "ug-2026-11-21-1795300500000",
  "generatedAt": "2026-11-21T14:08:20.000Z",
  "referenceDate": "2026-11-21",
  "timezone": "America/Sao_Paulo",
  "window": {
    "yesterday": "2026-11-20",
    "today": "2026-11-21",
    "tomorrow": "2026-11-22"
  },
  "cache": {
    "hit": false,
    "ageMs": 0,
    "ttlMs": 0
  },
  "filters": {
    "sets": ["today", "overnight", "tomorrow"],
    "sport": null,
    "league": null,
    "status": null,
    "liveOnly": false,
    "includeStats": "summary",
    "includeCompat": false,
    "includeProvenance": false
  },
  "counts": {
    "returned": { "today": 0, "overnight": 0, "tomorrow": 0, "total": 0 },
    "snapshotTotal": { "today": 0, "overnight": 0, "tomorrow": 0, "total": 0 },
    "live": { "returned": 0, "snapshotTotal": 0 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": false,
      "partial": false,
      "totalCalls": 3,
      "succeededCalls": 0,
      "failedCalls": 3,
      "emptySlateCalls": 0,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": 180000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "espn": {
      "ok": false,
      "partial": false,
      "totalCalls": 21,
      "succeededCalls": 0,
      "failedCalls": 21,
      "emptySlateCalls": 0,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": 60000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "sportsblaze": {
      "ok": false,
      "partial": false,
      "totalCalls": 16,
      "succeededCalls": 0,
      "failedCalls": 16,
      "emptySlateCalls": 0,
      "rawItemCount": 0,
      "dedupedItemCount": 0,
      "freshness": {
        "collectorTtlMs": { "boxscores": 60000, "schedule": 300000 },
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    }
  },
  "warnings": [
    "provider_unavailable:bzzoiro",
    "provider_unavailable:espn",
    "provider_unavailable:sportsblaze"
  ],
  "errors": [
    {
      "code": "ALL_PROVIDERS_FAILED",
      "message": "Todos os provedores falharam na coleta centralizada."
    }
  ],
  "sets": {
    "today": [],
    "overnight": [],
    "tomorrow": []
  }
}
```

### 5. Partida reconciliada ESPN + SportsBlaze (com `includeProvenance=true`)

```json
{
  "canonicalId": "ug:nba:espn:401810105",
  "set": "TODAY",
  "sport": "basketball",
  "sportFamily": "american",
  "league": { "code": "NBA", "name": "NBA", "id": null, "country": null, "flag": null, "logo": null },
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

### 6. Partida exclusiva SportsBlaze (`sportsblaze_only`)

> Note que `stats.hasTeamStats = true` e `stats.hasPlayerStats = true` porque os dados vieram do boxscore da SportsBlaze em memória, **mas `stats.available = false`, `stats.matchStatsId = null` e `stats.endpoint = null`**, pois não existe consulta compatível via `/api/match-stats` para UUIDs.

```json
{
  "canonicalId": "ug:nba:sb:99aa88bb-77cc-44dd-88ee-112233445566",
  "set": "TODAY",
  "sport": "basketball",
  "sportFamily": "american",
  "league": { "code": "NBA", "name": "NBA", "id": null, "country": null, "flag": null, "logo": null },
  "providerIds": {
    "espnId": null,
    "sportsblazeId": "99aa88bb-77cc-44dd-88ee-112233445566",
    "bzzoiroId": null,
    "lacrosseCollectorId": null
  },
  "providers": ["sportsblaze"],
  "reconciled": false,
  "reconciliationMode": "sportsblaze_only",
  "fixture": {
    "startTimeUtc": "2026-11-21T02:00:00.000Z",
    "rawDate": "2026-11-21T02:00:00Z",
    "timestampSource": "sportsblaze",
    "startDateBrt": "2026-11-20",
    "startTimeBrt": "23:00",
    "dateDisplay": "20/11/2026",
    "venue": "",
    "broadcaster": "",
    "season": { "year": 2026, "type": "Regular Season", "week": null },
    "endTime": { "endDate": null, "endTime": null, "finishedAt": null, "completedAt": null },
    "homeTeam": {
      "id": "sb-team-por",
      "name": "Portland Trail Blazers",
      "abbreviation": "POR",
      "logo": ""
    },
    "awayTeam": {
      "id": "sb-team-sac",
      "name": "Sacramento Kings",
      "abbreviation": "SAC",
      "logo": ""
    }
  },
  "liveOverlay": {
    "status": "FINAL",
    "live": false,
    "score": { "home": 112, "away": 107 },
    "timeBadge": "FIM",
    "period": 4,
    "clock": null,
    "tempo": "Fim",
    "liveState": { "period": 4 },
    "linescores": {
      "header": ["Q1", "Q2", "Q3", "Q4"],
      "home": [28, 30, 24, 30],
      "away": [25, 29, 27, 26],
      "periodScores": {
        "1": { "away": 25, "home": 28 },
        "2": { "away": 29, "home": 30 },
        "3": { "away": 27, "home": 24 },
        "4": { "away": 26, "home": 30 }
      }
    },
    "incidents": null,
    "eventUpdatedAt": null
  },
  "stats": {
    "available": false,
    "matchStatsId": null,
    "endpoint": null,
    "hasTeamStats": true,
    "hasPlayerStats": true,
    "keyPlayers": {
      "home": [{ "name": "Shaedon Sharpe", "pos": "G", "stat": "points: 26 · rebounds: 5" }],
      "away": [{ "name": "De'Aaron Fox", "pos": "G", "stat": "points: 29 · assists: 7" }]
    }
  }
}
```

### 7. Consulta com `includeStats=none` (`GET /api/games/central?date=2026-11-20&sets=today&sport=basketball&includeStats=none`)

```json
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
    "hit": true,
    "ageMs": 4500,
    "ttlMs": 60000
  },
  "filters": {
    "sets": ["today"],
    "sport": ["basketball"],
    "league": null,
    "status": null,
    "liveOnly": false,
    "includeStats": "none",
    "includeCompat": false,
    "includeProvenance": false
  },
  "counts": {
    "returned": { "today": 1, "overnight": 0, "tomorrow": 0, "total": 1 },
    "snapshotTotal": { "today": 2, "overnight": 1, "tomorrow": 1, "total": 4 },
    "live": { "returned": 1, "snapshotTotal": 3 }
  },
  "providerStatus": {
    "bzzoiro": {
      "ok": true,
      "partial": false,
      "totalCalls": 3,
      "succeededCalls": 3,
      "failedCalls": 0,
      "emptySlateCalls": 1,
      "rawItemCount": 1,
      "dedupedItemCount": 1,
      "freshness": {
        "collectorTtlMs": 180000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "espn": {
      "ok": true,
      "partial": false,
      "totalCalls": 21,
      "succeededCalls": 21,
      "failedCalls": 0,
      "emptySlateCalls": 18,
      "rawItemCount": 3,
      "dedupedItemCount": 3,
      "freshness": {
        "collectorTtlMs": 60000,
        "feedUpdatedAt": null,
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": null
      }
    },
    "sportsblaze": {
      "ok": true,
      "partial": false,
      "totalCalls": 16,
      "succeededCalls": 16,
      "failedCalls": 0,
      "emptySlateCalls": 14,
      "rawItemCount": 2,
      "dedupedItemCount": 2,
      "freshness": {
        "collectorTtlMs": { "boxscores": 60000, "schedule": 300000 },
        "feedUpdatedAt": "2026-11-21T01:34:42.000Z",
        "collectorFetchedAt": null,
        "sourceCacheAgeMs": null,
        "sourceCacheHit": true
      }
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
        "league": { "code": "NBA", "name": "NBA", "id": null, "country": null, "flag": null, "logo": null },
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
          "hasPlayerStats": true
        }
      }
    ],
    "overnight": [],
    "tomorrow": []
  }
}
```

### 8. Resposta HTTP `304 Not Modified`

**Requisição:**
```http
GET /api/games/central?date=2026-11-20&sets=today&sport=basketball&includeStats=none HTTP/1.1
If-None-Match: W/"ug-2026-11-20-1795224900000:c4e9f201"
```

**Resposta:**
```http
HTTP/1.1 304 Not Modified
ETag: W/"ug-2026-11-20-1795224900000:c4e9f201"
Cache-Control: public, max-age=15, must-revalidate
Vary: Accept-Encoding
```
*(Sem corpo HTTP; válido somente enquanto o snapshot `ug-2026-11-20-1795224900000` permanecer dentro do TTL de 60s.)*

---

## F. Matriz de Compatibilidade

### 1. Identificadores, Estatísticas e Reconciliação

| Origem / Modo | `canonicalId` | `providerIds` | `stats.available` | `stats.matchStatsId` | Proteção / Regra |
|---|---|---|---|---|---|
| **Futebol (`bzzoiro_only`)** | `ug:football:bzzoiro:<id>` | `bzzoiroId: "<id>"` | `true` (se `<id>` numérico) | `bsd-<id>` | `/api/match-stats` remove `bsd-` e consulta `/api/v2/events/<id>/stats/`. |
| **Americanos (`matched`)** | `ug:<lg>:espn:<espnId>` | `espnId`, `sportsblazeId` | `true` | `sb-<lg>-<espnId>` | Reconciliado 1-para-1 com delta máximo `<= 90 min` (protege doubleheaders MLB de colapsar). ESPN é autoridade de placar/status/relógio/período/logos. |
| **Americanos (`espn_only`)** | `ug:<lg>:espn:<espnId>` | `espnId` | `true` | `sb-<lg>-<espnId>` | Suporta NFL, NCAAF, MLB, NHL, NBA, WNBA via `/api/match-stats`. |
| **Americanos (`sportsblaze_only`)** | `ug:<lg>:sb:<uuid>` | `sportsblazeId` | **`false`** | **`null`** | Nunca fabrica `espnId` nem envia UUID para `/api/match-stats`. Pode ter `hasTeamStats`/`hasPlayerStats = true` em memória. |
| **Lacrosse (`espn_only`)** | `ug:lacrosse:espn:<espnId>` | `espnId`, `lacrosseCollectorId` (`lax-espn-<id>`) | **`false`** | **`null`** | `/api/match-stats` não suporta Lacrosse hoje (cairia no fallback de futebol). `compat.statsMatchId` mantém `lacrosse-<id>` somente se `includeCompat=true`. |

### 2. Mapeamento para Consumidores do Órbita Arena e ORB Lite

| Consumidor | Campos do `/api/games/central` utilizados | Observação de transição / legado |
|---|---|---|
| **`JogosDoDiaPage.tsx`** | `sets.{today,overnight,tomorrow}`, `fixture.*`, `liveOverlay.*`, `stats.endpoint` | Primeira página alvo de migração após Etapa 2B.4.3. Substitui 3+ fetches paralelos por 1 chamada. |
| **`Ticker.tsx`** | `sets.today`, `sets.overnight`, `fixture.{startDateBrt,startTimeBrt,timestampSource,homeTeam,awayTeam}`, `liveOverlay.{status,score,timeBadge}` | Elimina inferência frágil de `timestampSource` por prefixo (`bsd-`/`sb-`). |
| **`HeroCommandCenter.tsx` & `HeroCopa.tsx`** | `liveOverlay.{status,score,timeBadge,linescores,incidents}`, `fixture.*`, `stats.{available,matchStatsId,keyPlayers}`, `compat` (transitório) | Durante a migração, `includeCompat=true` fornece `heroStatus` (`"ADIADO"`), `tickerId`, `apiId` e `statsMatchId` sem contaminar o schema canônico. |
| **`MatchStatsService` (`/api/match-stats`)** | `stats.available`, `stats.matchStatsId`, `stats.endpoint`, `league.code`, `sport` | Só chama `/api/match-stats` quando `stats.available === true` e `stats.matchStatsId !== null`. |
| **`HomeLite.tsx` & `/api/jogos-texto`** | Mesma coleção do agregador central no backend | `/api/jogos-texto` passa a consumir a função interna do agregador central (mantendo saída enxuta PT-BR sem alterar `HomeLite` inicialmente). |
| **`MeusTimesPage.tsx` (`favoritesService`)** | `sets.{today,overnight,tomorrow}` para agenda imediata dos times favoritos | Mantém endpoints específicos por time para jogos fora da janela `[yesterday..tomorrow]`. |
| **ORB Lite (futuro)** | Presets `includeStats=none` / `sets=today` na v1.0; futura rota dedicada `GET /api/v1/games` | **Não** usar `?fields=` genérico em `/api/games/central`. Quando o ORB Lite externo for lançado, expor `GET /api/v1/games` reutilizando o mesmo serviço interno do agregador. |

---

## G. Decisões Finais e Pendências

### 1. Decisões aprovadas nesta etapa
- `GET /api/games/central` e `HEAD /api/games/central` (demais métodos -> `405`).
- Defaults oficiais: `sets=today,overnight,tomorrow`, `includeStats=summary`, `includeCompat=false`, `includeProvenance=false`.
- Separação formal entre `generatedAt` (snapshot), `providerStatus[p].freshness` (coletor/feed) e `liveOverlay.eventUpdatedAt` (evento).
- Remoção definitiva de `stats.ref`; preservação sempre dos 5 campos planos de referência em `stats`.
- Regra estrita de `stats.available`: exige suporte funcional em `/api/match-stats` (`false` para `sportsblaze_only` e `lacrosse`).
- `OVERNIGHT` permanece estritamente auxiliar (nunca entra em `counts.*.today`).
- Zero filtragem com disparo de novas chamadas aos provedores externos.

### 2. Limitações técnicas conhecidas
- Cache em memória por processo (sem compartilhamento entre múltiplas instâncias/réplicas no Render).
- `bzzoiroCollector`, `espnAmericanCollector` e `espnLacrosseCollector` ainda não expõem `fetchedAt`/`fromCache` no objeto de retorno (portanto `collectorFetchedAt`, `sourceCacheAgeMs` e `sourceCacheHit` retornam `null` até evolução opcional na Etapa 2B.4.3).
- `includeStats=full` só entrega `teamStatistics`/`players` em partidas com cobertura SportsBlaze Boxscores (NFL, MLB, NBA, WNBA); demais ligas/esportes dependem de `/api/match-stats` quando `stats.available === true`.
- Ausência de suporte a Lacrosse em `/api/match-stats`.

### 3. Pendências para a Etapa 2B.4.3 (Implementação da rota `/api/games/central`)
1. Expor `fetchedAtMs` / `fromCache` (a partir do `timestamp` já existente no cache dos coletores) ou iniciar com `null` seguro nos coletores que ainda não os retornam, preenchendo `providerStatus[p].freshness` sem fabricar datas.
2. Implementar a camada de transformação `CentralGamesResponse` + `CentralGameMatch` reutilizando `collectUnifiedGamesShadow` (ou extraindo função base compartilhada em `unifiedGamesCollector.ts` preservando `/api/games/shadow` intacto).
3. Implementar validação rigorosa de query params (`400 Bad Request`), método (`405`), `ETag`/`If-None-Match` (`304 Not Modified`), `Cache-Control` e TTL reduzido (`20s`) para snapshots `degraded: true`.
4. Garantir suíte de testes unitários e de contrato cobrindo todos os cenários da Seção E e critérios da Seção H.

### 4. Melhorias futuras fora do escopo (Etapas posteriores)
- Suporte nativo de `/api/match-stats` para `?canonicalId=ug:...`, SportsBlaze UUID e Lacrosse PLL.
- Cache distribuído (Redis) caso o serviço escale para múltiplas instâncias no Render.
- Endpoint dedicado `GET /api/v1/games` para o ORB Lite.

---

## H. Critérios de Aceite da Implementação (Etapa 2B.4.3)

A implementação da Etapa 2B.4.3 só poderá ser aprovada quando **todos** os itens abaixo forem verificados por testes automatizados e inspeção:

- [ ] `GET /api/games/central` e `HEAD /api/games/central` respondem corretamente; `POST/PUT/DELETE/PATCH` retornam `405` com `Allow: GET, HEAD`.
- [ ] Defaults aplicados exatamente como especificado: `sets=today,overnight,tomorrow`, `includeStats=summary`, `includeCompat=false`, `includeProvenance=false`.
- [ ] Query params inválidos (`date`, `sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat`, `includeProvenance`, limites de lista > 20) retornam `400` com `ok: false` e `error.code`.
- [ ] `sets.today`, `sets.overnight` e `sets.tomorrow` obedecem integralmente `brtDate.ts` e `gameSets.ts`, sem misturar `overnight` em `counts.*.today`.
- [ ] `counts.returned.*` correspondem exatamente a `sets.today.length`, `sets.overnight.length` e `sets.tomorrow.length` em todos os cenários (normal, filtrado, vazio, degradado e `503`).
- [ ] Falha parcial de 1 ou mais provedores (com ao menos 1 saudável) retorna `200 OK`, `ok: true`, `degraded: true` e preserva integralmente os jogos dos provedores saudáveis.
- [ ] Falha total de todos os provedores retorna `503`, `ok: false`, `degraded: true` e nunca grava em cache.
- [ ] Resposta sem partidas (`empty slate`) com provedores saudáveis retorna `200 OK`, `ok: true`, `degraded: false`.
- [ ] `stats.ref` não existe; os 5 campos planos (`available`, `matchStatsId`, `endpoint`, `hasTeamStats`, `hasPlayerStats`) estão presentes mesmo com `includeStats=none`.
- [ ] Partidas `sportsblaze_only` e `lacrosse` retornam `stats.available === false`, `stats.matchStatsId === null` e `stats.endpoint === null` (nunca enviando UUID para `/api/match-stats`).
- [ ] `providerStatus[p].freshness` nunca inventa timestamps quando o coletor não fornece (`null` explícito) e `liveOverlay.eventUpdatedAt` nunca reutiliza `sb.updated`, `generatedAt` ou `startTime`.
- [ ] `providerSnapshots` e `excluded` do Shadow nunca vazam em `/api/games/central`.
- [ ] `ETag` e `If-None-Match` retornam `304 Not Modified` dentro da validade do snapshot e invalidam ao expirar o TTL; `Cache-Control` nunca excede o TTL do snapshot.
- [ ] `/api/games/shadow` e todos os fluxos existentes do site permanecem 100% inalterados e funcionando sem regressões.
