# ÓRBITA ARENA — ETAPA 2B.5.2.5
## Auditoria e Especificação do Cache Antecipado de Jogos

**Data:** segunda-feira, 23/11/2026  
**Escopo:** Auditoria técnica e especificação arquitetural somente leitura. **Nenhum** arquivo de produção (`server.ts`, `src/**`, `firestore.rules`, `firebase-blueprint.json`, `.env`, frontend ou testes) foi modificado, nem foi executado commit, push ou deploy.  
**Objetivo:** Eliminar a latência fria (`~17s–44s`, percebida em ~30s no primeiro acesso ao `/jogosdodia`) provocada pela coleta síncrona sob demanda, especificando uma arquitetura de **cache antecipado em 3 responsabilidades** (Fixtures, Live Overlay e Servimento Combinado), coordenada com warmup no boot, scheduler interno, virada de dia BRT, cronjob externo de 14 minutos do Render e persistência resiliente L2 (Firestore).

---

## 1. Sumário Executivo da Auditoria

1. **Causa raiz da latência no primeiro acesso:** embora a Etapa `2B.5.2.4` tenha otimizado o gargalo interno do coletor Bzzoiro (paralelismo limitado de 16 slots, coalescing de preload e `overnight_candidates` para `yesterday`), **toda coleta da API Central e Shadow continua estritamente reativa (*on-demand*)** — disparada apenas quando um visitante faz `GET /api/games/central` ou `GET /api/games/shadow` com cache frio ou expirado.
2. **Desconexão dos loops existentes:** o único timer recorrente no `server.ts` (`setInterval` de 30s em `server.ts:2256-2357`, `[Firebase Background Sync]`) sincroniza apenas o legado `sportsData.json` + ESPN simplificada para a coleção `realTimeScores`. Ele **não** aquece `collectUnifiedGamesSnapshot`, Bzzoiro, SportsBlaze, Lacrosse nem `/api/games/central`.
3. **Efeito de TTL sem SWR proativo:** na camada central (`unifiedGamesCollector.ts:408-411`), o TTL saudável é de `60s` (`UNIFIED_SHADOW_CACHE_TTL_MS`) e a janela *stale* (`180s`) só é usada em falha total dos provedores — nunca para *stale-while-revalidate* assíncrono em revalidação normal. Quando o snapshot expira, o próximo usuário bloqueia de forma síncrona.
4. **Solução especificada:** desacoplar **Preparação de Grade (`10–15 min`)**, **Atualização Dinâmica de Estado/Placar (`30–60s` adaptativo)** e **Servimento Instantâneo de Snapshot Combinado (`RAM L1` + hidratação opcional `L2 Firestore`)**, com proteções contra truncamento, coalescing com o cronjob de 14 minutos (via rota autenticada), detecção automática de virada de dia BRT e *graceful shutdown*.

---

## 2. Auditoria Detalhada do Estado Atual

### 2.1. Inventário de Caches, TTLs e Timers Existentes no Backend

| Camada / Módulo | Estrutura de Cache | TTL Saudável / Cooldown | Escopo / Comportamento Atual | Limitação frente ao Problema |
|---|---|---|---|---|
| **API Central / Shadow**<br>`src/server/collectors/unifiedGamesCollector.ts:408-417` | `unifiedSharedCache`, `unifiedColdFailureCache`, `unifiedSharedInFlight` | Saudável: **`60.000 ms`** (`60s`)<br>Degradado: **`20.000 ms`** (`20s`)<br>Stale fallback: **`180.000 ms`** (`180s`)<br>Cooldown falha: **`20.000 ms`** (`20s`) | Coleta janela `[yesterday, today, tomorrow]` via `Promise.all` (até ~40 chamadas lógicas de topo + subchamadas Bzzoiro). Coalesce concorrência entre Central e Shadow via `unifiedSharedInFlight`. | Preenchido **apenas** quando chega requisição HTTP. Ao vencer 60s, o próximo usuário paga revalidação síncrona (~5s com Bzzoiro quente ou mais se frio). |
| **HTTP Cache Central**<br>`src/server/centralGamesApi.ts:1201-1213, 1268-1297` | `ETag` (`W/"snapshotId:hash8"`) + `Cache-Control` | Saudável: `public, max-age=15, must-revalidate`<br>Degradado: `public, max-age=10, must-revalidate`<br>Stale/erro: `no-store` | Retorna `304 Not Modified` (~84 ms em prod) se `If-None-Match` bater (exceto em modo `stale`). | No Render (`cf-cache-status: DYNAMIC`), Cloudflare não cacheia na borda; cada novo usuário sem ETag local cai direto no Node. |
| **Bzzoiro Football**<br>`src/server/collectors/bzzoiroCollector.ts:13-16, 540, 638, 976, 1603` | `footballCache`, `footballLastCompleteCache`, `EVENT_BROADCASTS_CACHE`, `EVENT_INCIDENTS_CACHE`, `BSD_LEAGUES_CACHE`, `BSD_VENUES_CACHE` | Eventos: **`180s`** (`3 min`)<br>Último completo: **`30 min`**<br>Broadcasts: **`5 min`**<br>Incidents (`FINAL` only): **`5 min`**<br>Leagues: **`1h`** / Venues: **`2h`** | Pipeline Stage 1 (`live + events pagination + preload`) + Stage 2 paralelo limitado (`16` slots) + filtro pré-enriquecimento + `overnight_candidates` para `yesterday`. | Sem warmup proativo: no primeiro acesso após boot ou expiração de `180s`, dispara paginação + enriquecimento em tempo real na thread do request. |
| **ESPN American**<br>`src/server/collectors/espnAmericanCollector.ts:84, 620-648` | `espnAmericanCache`, `espnAmericanInFlight` | **`60.000 ms`** (`60s`) | 18 chamadas em `unifiedGamesCollector` (6 ligas × 3 datas: `boxscores` para ontem/hoje, `schedule` para amanhã). Nunca cacheia falhas. | Recarregado integralmente (inclusive `yesterday` e `tomorrow`) a cada 60s quando há tráfego. |
| **ESPN Lacrosse PLL**<br>`src/server/collectors/espnLacrosseCollector.ts:53-56` | `espnLacrosseCache`, `espnLacrosseInFlight` | **`60.000 ms`** (`60s`) | 3 chamadas por janela (`yesterday`, `today`, `tomorrow`). | Mesmo comportamento síncrono sob demanda. |
| **SportsBlaze American**<br>`src/server/collectors/sportsblazeAmericanCollector.ts:122-123, 165-166` | `sportsblazeCache`, `sportsblazeInFlight` | Boxscores: **`60.000 ms`** (`60s`)<br>Schedule: **`300.000 ms`** (`5 min`) | 12 chamadas de boxscores (4 ligas × 3 datas) + 4+ chamadas de schedule. Fornece `teamStatistics`, `players`, `liveState`, `updated`. | Consultas de `yesterday` e `today` vencem a cada 60s e bloqueiam o `Promise.all` do agregador. |
| **Sync Loop Legado**<br>`server.ts:2256-2357` | Timer `setInterval(..., 30000)` | **`30.000 ms`** (`30s`) | Lê `sportsData.json` (GitHub/local) + `fetchESPNScores()` simples e grava em `realTimeScores` no Firestore. | **Não aquece** Bzzoiro, SportsBlaze, Lacrosse, `unifiedSharedCache` nem `/api/games/central`; dispara escritas Firestore a cada 30s sem hash diff. |
| **Rotas Legadas de Coleta**<br>`src/server/registerCollectorRoutes.ts:300-653` | Consomem diretamente `bzzoiroCollector`, `espnAmericanCollector`, `espnLacrosseCollector` | Mesmos TTLs internos (`180s` / `60s`) | `/api/football/today`, `/api/jogos-texto`, `/api/jogos-lite`, `/api/football/standings`, `/api/football/fixtures`. | Não leem o snapshot unificado nem beneficiam Central/Shadow com aquecimento próprio se o visitante bater direto nelas antes da Central. |

### 2.2. Diagnóstico do Comportamento Pós-Deploy Observado em Produção

Conforme medido em produção no Render (`temp/etapa-2b-5-2-2-validacao-pos-deploy-performance.md`):
- **Cache frio completo (pré-2B.5.2.4):** `44.353 ms` (sendo ~39s do antigo N+1 sequencial Bzzoiro + ~5,07s ESPN/SportsBlaze).
- **Revalidação após expiração de 180s do Bzzoiro:** `17.357 ms`.
- **Revalidação após 60s com Bzzoiro ainda quente:** `5.072 ms`.
- **Cache HIT na RAM:** **`128–169 ms`** (e **`84 ms`** com `ETag` `304 Not Modified`).
- **Tamanhos observados em produção (`166` jogos no domingo `22/11/2026`):**
  - `GET /api/games/central` (`includeStats=summary` padrão): **`~247.438 bytes`** (~241,6 KiB).
  - `GET /api/games/central?includeStats=none&includeCompat=true`: **`~269.425 bytes`** (~263,1 KiB).
  - `GET /api/games/shadow`: **`755.557 bytes`** (~737,8 KiB).

**Conclusão operacional:** a arquitetura de projeção da Central em RAM já entrega respostas sub-200ms em HIT. O único motivo de o usuário sentir espera (~5s na revalidação de 60s ou ~17–30s em cold start pós-restart/expiração) é **a ausência de aquecimento proativo desacoplado do ciclo de requisição HTTP**.

### 2.3. Lacunas Estruturais Identificadas no `server.ts` e Firestore

1. **Ausência de boot warmup (`server.ts:2445-2447`):** `app.listen(PORT, "0.0.0.0")` não dispara pré-carga de `unifiedGamesCollector`.
2. **Ausência de *graceful shutdown*:** `server.ts` não registra `SIGTERM`/`SIGINT` para limpar timers (`clearInterval`/`clearTimeout`).
3. **Middleware CORS × `OPTIONS /api/games/central`:** já mapeado na Etapa `2B.5.2.2` (`server.ts:50-51` intercepta `OPTIONS` retornando `200 OK` antes de `registerCollectorRoutes.ts:717-724` que enviaria `204` + `Allow`). Não interfere no cache, mas merece registro para alinhamento futuro.
4. **SDK Firebase Client no servidor (`server.ts:6-88`):** o backend usa o **Firebase Client SDK** (`firebase/app` + `initializeFirestore(..., dbId)`), **sem `firebase-admin`**. No momento, `firestore.rules:19-30` mantém `allow read, write: if true` em `realTimeScores` e `news_comments` porque o servidor escreve sem autenticação (`request.auth == null`). Qualquer coleção nova de cache L2 (`games_cache_snapshots`) precisa resolver explicitamente essa restrição de segurança (ver Seção 6.3).

---

## 3. Arquitetura Alvo: Cache Antecipado em 3 Responsabilidades

A arquitetura substitui o ciclo monolítico síncrono ("coletar tudo a cada 60s quando um usuário chega") por um **orquestrador proativo em memória** composto por três responsabilidades independentes e complementares:

```
┌─────────────────────────────────────────────────────────────────────────┐
│               ORQUESTRADOR DE CACHE ANTECIPADO (Node RAM)               │
│                                                                         │
│  ┌──────────────────────────────┐    ┌───────────────────────────────┐  │
│  │ A. PREPARAÇÃO DA GRADE       │    │ B. ATUALIZAÇÃO DE ESTADO LIVE │  │
│  │    (Fixtures Preparer)       │    │    (Live Overlay Updater)     │  │
│  │  • Janela: D-1, D0, D+1 BRT  │    │  • Intervalo: 30s ativo /     │  │
│  │  • Intervalo: 12 min (10-15m)│    │    120-180s ocioso (0 LIVE)   │  │
│  │  • Boot + Virada 00:00 BRT   │    │  • Somente deltas dinâmicos   │  │
│  │  • Times, ligas, venues, TV, │    │  • Placares, status, relógio, │  │
│  │    logos, sets TODAY/OVN/TOM │    │    período, gols/incidents    │  │
│  └──────────────┬───────────────┘    └───────────────┬───────────────┘  │
│                 │ FixturesSnapshot                   │ LiveOverlayMap   │
│                 └───────────────┬────────────────────┘                  │
│                                 ▼                                       │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ C. SNAPSHOT COMBINADO EM MEMÓRIA (L1 Store — Troca Atômica)       │  │
│  │  • Merge determinístico: fixture + liveOverlay + heroCompat       │  │
│  │  • Hash de conteúdo -> ETag / snapshotId estáveis em 304          │  │
│  │  • Fallback preservado: erro parcial NUNCA zera snapshot íntegro  │  │
│  └──────────────┬───────────────────────────┬────────────────────────┘  │
└─────────────────┼───────────────────────────┼───────────────────────────┘
                  │ Leitura < 5 ms (sem I/O)  │ Persistência assíncrona
                  ▼                           │ (Hash diff + debounce)
   GET/HEAD /api/games/central                ▼
   GET /api/games/shadow             L2 Firestore (Comprimido / Enxuto)
   (Opcional: /api/jogos-texto,       Restaurado no Boot se BRT == hoje
    /api/jogos-lite)
```

### 3.1. Responsabilidade A — Preparação Antecipada da Grade (`Fixtures Preparer`)

**Objetivo:** manter pré-carregada e classificada a grade estrutural completa do dia (`TODAY`, `OVERNIGHT`, `TOMORROW`) antes que qualquer usuário acesse o site.

- **Periodicidade:** a cada **`12 minutos`** (configurável entre `10–15 min`, ex.: `CACHE_FIXTURES_INTERVAL_MS = 720_000`), além de execução imediata no **boot** (se L2 ausente/expirado/de outra data) e na **virada de data BRT (`00:00 America/Sao_Paulo`)**.
- **O que coleta e prepara:**
  - Executa o pipeline completo de `executeUnifiedShadowCollection` para `referenceDateBrt = getBrtTodayStr()`, cobrindo `[yesterdayBrt, referenceDateBrt, tomorrowBrt]`:
    - **Bzzoiro:** `/events/live/`, `/events/?date_from=...` (com `enrichmentScope: "overnight_candidates"` em `yesterdayBrt`), `venues/`, `leagues/`, `broadcasts/` e `incidents/`.
    - **ESPN:** `boxscores` (`yesterday`, `today`), `schedule` (`tomorrow`) + `lacrosse/pll`.
    - **SportsBlaze:** `boxscores` (`yesterday`, `today`, `tomorrow`) + `schedule` (que já possui TTL interno de `300s`).
  - Executa reconciliação americana (`reconcileAmericanLeagueGames`) e classificação oficial de conjuntos (`partitionGamesIntoSets` -> `TODAY`, `OVERNIGHT`, `TOMORROW`, `excluded`).
- **Artefato produzido (`FixtureGradeSnapshot`):**
  - Mapa indexado por `canonicalId` + índices auxiliares por IDs externos (`bzzoiroId`, `espn:{league}:{id}`, `sb:{league}:{id}`, `lax:{id}`) para permitir aplicação O(1) de overlays na Camada B.
  - Metadados estáticos/semi-estáticos de cada partida: `fixture`, `league`, `providerIds`, `providers`, `reconciliationMode`, `set`, `fieldProvenance` base, `stats` base (incluindo `matchStatsId`, `teamStatistics`, `players`).
  - `fixturesHash`: hash SHA-256 calculado sobre IDs canônicos, horários, times, ligas, estádios e transmissões — usado para detectar mudanças estruturais (ex.: novo jogo adicionado, jogo adiado, alteração de horário/transmissão) e decidir se há necessidade de persistência em L2.
- **Proteção contra degradação:**
  - Se a execução da Camada A falhar totalmente (`!anyProviderSucceeded`) ou retornar Bzzoiro truncado (`eventsComplete === false` / `materialPartial === true`) **quando já existir um `FixtureGradeSnapshot` completo da mesma data BRT**:
    - O snapshot estrutural anterior **não é substituído** por lista truncada ou vazia.
    - Atualiza-se apenas `degraded: true`, `providerStatus` e `warnings` do envelope servido, evitando repetição do incidente de `22/11/2026` (perda de 46 jogos por paginação incompleta).

### 3.2. Responsabilidade B — Atualização Dinâmica de Estado (`Live Overlay Updater`)

**Objetivo:** manter placares, status, minutagem/período/relógio, `liveState`, `linescores` e `incidents` (`homeGoals`/`awayGoals`) atualizados em tempo quase real **sem repetir chamadas pesadas de grade, `yesterday`, `tomorrow`, `leagues`, `venues` ou `broadcasts`**.

- **Cadência adaptativa:**
  - **Modo Ativo (`30s`, configurável `30–60s`):** quando existe ao menos 1 partida `LIVE` **ou** ao menos 1 partida `UPCOMING` dentro da janela de iminência `[now - 15 min, now + 15 min]` (ou jogo `LIVE` aguardando confirmação de encerramento).
  - **Modo Ocioso (`120s–180s`):** quando `liveCount === 0` e nenhuma partida está próxima de começar (ex.: madrugada sem jogos ativos). Se o relógio se aproximar do próximo kickoff programado (`nextKickoffMs`), agenda/reduz automaticamente o próximo tick para não atrasar a transição `UPCOMING -> LIVE`.
- **Escopo estrito de requisições por provedor (caminho rápido incremental):**
  1. **Bzzoiro (`fetchBzzoiroLiveOverlayDelta` — novo caminho enxuto a ser adicionado em `bzzoiroCollector.ts`):**
     - Dispara **`1` chamada HTTP leve**: `GET /api/v2/events/live/?page_size=100` (paginando apenas se `next !== null`, raríssimo em jogos simultâneos).
     - **Zero** chamadas para `/leagues/`, `/venues/`, `/broadcasts/`, `yesterday` ou `tomorrow`.
     - **Incidents sob delta estrito:** consulta `GET /api/v2/events/{id}/incidents/` **somente** para partidas cujo placar total (`home_score + away_score`) aumentou em relação ao último overlay ou que estão `LIVE` com placar `> 0` e ainda sem gols sincronizados para aquele placar.
     - **Detecção segura de transição `LIVE -> FINAL`:** jogos que estavam `LIVE` (ou `UPCOMING` com `startTime <= now - 5 min`) no snapshot anterior e **não apareceram** no `/events/live/` não podem ficar eternamente travados como `LIVE`. O updater mantém uma lista curta `missingLiveIds` e:
       - Prioridade 1: consulta pontual `GET /api/v2/events/{id}/` (bounded pool, 1 req por jogo recém-saído do live, tipicamente 0–5 jogos) + último `/incidents/` se houve gol no final, confirmando `FINISHED` e placar definitivo.
       - Fallback: se mais de 10 partidas saírem ao mesmo tempo (ex.: término simultâneo de rodada), agenda revalidação antecipada de `today` na Camada A.
  2. **ESPN American & Lacrosse PLL:**
     - Identifica apenas o subconjunto `activeLeagues` que possuem jogos `LIVE` ou na janela de início (`todayBrt`, mais `yesterdayBrt` **exclusivamente** se houver partida americana de ontem ainda marcada `LIVE` no conjunto `OVERNIGHT`).
     - Consulta apenas o endpoint `scoreboard` dessas ligas ativas (ex.: 1–4 requests paralelos de ~150–270 ms cada, em vez de 21 chamadas fixas).
  3. **SportsBlaze:**
     - Consulta `boxscores` **apenas** para as ligas suportadas (`NFL`, `MLB`, `NBA`, `WNBA`) e datas que contenham jogos `LIVE` ou recém-finalizados nos últimos 10 minutos (mantendo placar, `liveState`, `linescores`, `teamStatistics` e `players` atualizados). **Zero** chamadas de `schedule` na Camada B.
- **Controle de versão por diff de conteúdo (`overlayHash`):**
  - Calcula assinatura determinística dos campos dinâmicos (`status`, `score.home`, `score.away`, `period`, `clock`, `tempo`, `timeBadge`, `linescores`, `homeGoals`, `awayGoals`, `liveState`) de todas as partidas.
  - **Se nada mudou** (`newOverlayHash === currentOverlayHash` e estado `degraded`/erros materiais inalterados):
    - **Não** gera novo `snapshotId`, mantendo o **`ETag` idêntico** para que clientes recorrentes recebam `304 Not Modified` (84 ms, `0 B` de corpo).
    - Atualiza apenas `lastLiveCheckAtMs` (para telemetria interna e observabilidade).
  - **Se qualquer placar, status, gol ou clock mudou:**
    - Gera novo `LiveOverlaySnapshot` e combina atomicamente na Camada C, emitindo novo `snapshotId` (`ug:${referenceDateBrt}:${nowMs}:${seq}`) para invalidar imediatamente o `ETag` dos clientes na próxima revalidação.
- **Proteção contra sobrescrita vazia:**
  - Falha ou timeout na Camada B **nunca** apaga overlays existentes nem rebaixa placar conhecido; marca apenas `liveOverlayDegraded` / erro no relatório do provedor afetado.

### 3.3. Responsabilidade C — Servimento do Snapshot Combinado (`Combined Snapshot Store`)

**Objetivo:** garantir que todo consumidor (`GET/HEAD /api/games/central` e `GET /api/games/shadow`) leia um snapshot pronto em RAM em poucos milissegundos, sem jamais disparar coleta fria upstream durante o atendimento de requisições da data corrente.

- **Merge determinístico (`buildCombinedResolution`):**
  - Combina `FixtureGradeSnapshot` + `LiveOverlayMap` em um objeto compatível com `UnifiedSnapshotResolution`:
    - Atualiza `match.status`, `match.isLive`, `match.isPostponed`, `match.completed`, `match.homeTeam.score`, `match.awayTeam.score`, `match.liveOverlay`, `match.heroCompat`, `match.endDate`/`endTime`/`finishedAt`/`completedAt` e `match.fieldProvenance` afetada.
    - Mantém **re-classificação leve de conjunto (`partitionGamesIntoSets`)** após aplicar o overlay: garante que um jogo de `yesterday` em `OVERNIGHT` que passe de `LIVE -> FINAL` com `finishedAt` ou que passe de `23:59` permaneça no conjunto exato definido pelas regras da Etapa 2A.
    - Recalcula contadores (`counts.today`, `counts.overnight`, `counts.tomorrow`, `reconciliationSummary`).
  - **Troca atômica em memória:** a referência `activeCombinedState` (imutável para leitura) é substituída de uma só vez após concluir o merge, impedindo leitura de estado meio-atualizado em requisições concorrentes.
- **Atendimento aos endpoints públicos:**
  - **`GET/HEAD /api/games/central` (para `date === todayBrt` ou omitido):**
    1. Se `activeCombinedState` em RAM corresponde a `todayBrt` $\rightarrow$ projeta filtros (`projectUnifiedSnapshotToCentral`), calcula `ETag`, `Cache-Control` e responde em **< 5–10 ms** (mais RTT de rede).
    2. Se o servidor acabou de subir e o warmup inicial ainda está nos primeiros segundos (`bootWarmup.status === "pending"`):
       - Se L2 Firestore já hidratou $\rightarrow$ responde imediatamente com L2 (`cache.hit = true`, `cache.stale` conforme idade).
       - Se L2 também estava vazio (cold boot absoluto antes do 1º warmup terminar) $\rightarrow$ aguarda a mesma `Promise` `warmupInFlight` em andamento (coalescing único), evitando disparar coleta paralela duplicada.
  - **`GET /api/games/central?date=YYYY-MM-DD` (datas passadas/futuras diferentes de `todayBrt`):**
    - Mantém cache LRU secundário limitado (ex.: `MAX_HISTORICAL_DATE_CACHE = 5` entradas, TTL `5 min` para datas passadas fechadas) sem interferir nem competir com o snapshot principal de `todayBrt`.
  - **`GET /api/games/shadow`:**
    - Consome o **mesmo** `Combined Snapshot Store` para `todayBrt`, preservando paridade total de totais e `generatedAt`.
  - **Projeção futura para rotas legadas (`/api/jogos-texto`, `/api/jogos-lite`, `/api/football/today`):**
    - Enquanto ainda houver clientes ou páginas chamando rotas legadas (`App.tsx` / versões antigas em cache), o aquecimento da Camada A já preenche os caches internos (`footballCache` / `espnAmericanCache` / `espnLacrosseCache`), **ou** em etapa posterior essas rotas poderão ser convertidas para ler direto do `Combined Snapshot Store` sem qualquer nova chamada externa.
  - **Sobre `cache.hit`, `cache.stale`, `cache.ageMs` e `Cache-Control` na Central:**
    - `cache.ageMs` passa a refletir `Math.max(0, nowMs - snapshot.lastVerifiedOrUpdatedMs)` (ou idade da última coleta de fixtures/live conforme semântica abaixo), mantendo `cache.ttlMs` coerente com o ciclo de atualização (`30_000–60_000 ms`) e `Cache-Control: public, max-age=15, must-revalidate` (`max-age=10` se `degraded: true`; `no-store` se `stale: true` ou `!ok`).
    - Se o snapshot servido vier de fallback restaurado (ex.: L2 antigo acima da tolerância ou falha persistente dos provedores além da janela normal), `cache.stale = true`, `degraded = true` e header `Warning: 110 - "Response is Stale"` são emitidos exatamente como exige o contrato Central `1.0`.

---

## 4. Coordenação: Boot Warmup, Scheduler, Virada BRT e Cronjob de 14 Minutos

### 4.1. Fluxo de Inicialização no Boot (`server.ts`)

Para não bloquear o *healthcheck* inicial do Render (que monitora a abertura da porta `3000`):

1. `app.listen(PORT, "0.0.0.0")` abre imediatamente a porta HTTP.
2. Em paralelo imediato (`void prewarmedCache.start()`):
   - **Passo 1 — Hidratação L2 rápida (timeout rígido `1.500 ms`):** tenta ler do Firestore o último snapshot salvo para `todayBrt = getBrtTodayStr()`.
     - Se existir, for íntegro e pertencer **estritamente** a `todayBrt`: carrega imediatamente na RAM L1 (`bootWarmup.source = "l2_firestore"`). Caso sua idade seja `<= 15 min`, marca como utilizável imediatamente enquanto agenda Live Overlay em segundo plano; se tiver `> 15 min`, marca `stale: true, degraded: true` até a Camada A concluir a atualização.
     - Se for de data BRT diferente (`snapshot.referenceDateBrt !== todayBrt`), **descarta integralmente** o L2 para não servir jogos de ontem como se fossem de hoje.
   - **Passo 2 — Warmup L1 imediato (não bloqueante):** se L2 não existia, era de outro dia BRT ou tinha mais de `10 min`, dispara imediatamente a execução inicial da Camada A (`Fixtures Preparer`), armazenada em `fixturesInFlight` para que qualquer requisição recebida nos primeiros segundos aguarde a mesma execução coalescida em vez de iniciar outra.
   - **Passo 3 — Ativação dos loops periódicos e listeners de encerramento.**

### 4.2. Virada de Meia-Noite BRT (`00:00 America/Sao_Paulo`)

- Todo cálculo de data utiliza `getBrtTodayStr()` (`America/Sao_Paulo`, nunca UTC do container Render).
- A cada tick de live (`30–120s`) e no início de cada ciclo de fixtures (além de verificação no momento de servir requisições), compara-se `currentBrtDate = getBrtTodayStr()` com `activeCombinedState.referenceDateBrt`.
- **Ao detectar `currentBrtDate !== activeReferenceDateBrt` (ex.: `23:59 -> 00:00 BRT`):**
  1. Dispara imediatamente uma nova coleta completa de Camada A para a nova `currentBrtDate` (com coalescing em `rolloverInFlight` para não duplicar).
  2. Mantém um estado transitório controlado durante os poucos segundos da transição **ou** aplica corte limpo mantendo `OVERNIGHT` da nova janela `[newYesterday, newToday, newTomorrow]`.
  3. Ao concluir, substitui `activeCombinedState` pela nova data, limpa chaves vencidas do L1 e grava o novo snapshot inicial do dia no L2.
  4. **Regra crítica para L2 em virada de dia:** qualquer snapshot L2 com `referenceDateBrt !== currentBrtDate` é considerado inválido para o novo dia no boot e no scheduler.

### 4.3. Integração com o Cronjob Externo de 14 Minutos (Render)

O Render coloca instâncias *free/starter* em hibernação após 15 minutos sem tráfego HTTP externo; por isso existe o cronjob externo a cada **14 minutos**.

- **Papel de cada mecanismo (zero conflito):**
  - **Cronjob externo (14 min):** mantém o container Render acordado e, opcionalmente, atua como gatilho/watchdog externo de saúde do cache.
  - **Scheduler interno (Node):** mantém a grade fresca a cada `12 min` e os placares ao vivo a cada `30–60s` enquanto o processo está rodando.
- **Por que NÃO acoplar o refresh ao `GET /api/health` aberto:**
  - `GET /api/health` é público e consultado por *load balancers*/browser (`JogosDoDiaPage` faz healthcheck com timeout de `2.5s`!). Disparar coleta pesada ou mutação de estado dentro de `/api/health` causaria timeout, risco de DoS anônimo e desperdício de cota de APIs.
  - Portanto, `/api/health` permanece **100% passivo, rápido (< 10 ms) e somente leitura**.
- **Endpoint dedicado e autenticado para cron/manual (`POST /api/internal/cache/refresh`):**
  - **Autenticação obrigatória:** exige `Authorization: Bearer <CACHE_REFRESH_SECRET>` (ou header `x-cache-refresh-secret`), validado com `crypto.timingSafeEqual`.
    - Sem secret configurado no ambiente: retorna `503 Service Unavailable` (nunca libera acesso público por omissão).
    - Token ausente ou inválido: retorna `401 Unauthorized` / `403 Forbidden`.
  - **Deduplicação e Cooldown (anti-tempestade):**
    - Aceita parâmetro opcional `mode`: `"auto"` (padrão), `"live"`, `"fixtures"` ou `"force"` (limitado por cooldown mínimo de segurança, ex.: `15s`).
    - Em `mode: "auto"` (recomendado para o cronjob de 14 min):
      - Se `fixturesInFlight` já estiver rodando $\rightarrow$ aguarda/reutiliza a mesma Promise (`coalesced: true`), sem disparar nova coleta.
      - Se o `FixtureGradeSnapshot` atual tem idade `< 10 min` (`600_000 ms`) $\rightarrow$ não refaz grade; executa apenas um tick de Camada B (`Live Overlay`) se idade live `>= 25s`, ou retorna `200 { refreshed: false, reason: "already_fresh" }`.
      - Se idade de fixtures `>= 10 min` $\rightarrow$ executa Camada A completa (`refreshed: true, scope: "fixtures"`).
  - **Flexibilidade operacional:** se o cronjob externo atual apenas bater em `GET /api/health`, o **scheduler interno** já mantém tudo aquecido continuamente; se for configurado para bater em `POST /api/internal/cache/refresh` com o token Bearer, ele atua como *heartbeat* duplo garantindo que nem mesmo um timer travado fique sem revalidação acima de 14 min.

### 4.4. Locks de Concorrência e Coordenação Multi-Instância

1. **No mesmo processo Node (single-instance Render atual):**
   - `fixturesInFlight: Promise<...> | null` (impede 2 coletas completas simultâneas entre scheduler de 12 min, cronjob de 14 min, rollover BRT ou warmup).
   - `liveInFlight: Promise<...> | null` (impede 2 ticks live concorrentes).
   - Se um tick live disparar enquanto `fixturesInFlight` estiver ativo, o tick live **pula/aguarda** sem sobrepor dados mais antigos sobre a grade nova.
2. **Em cenário futuro com `>= 2` instâncias Node (escala horizontal):**
   - Sem coordenação, 2 instâncias multiplicariam por 2× as chamadas ao Bzzoiro/SportsBlaze/ESPN e poderiam sobrescrever o mesmo documento no Firestore fora de ordem.
   - **Especificação de Lock Distribuído via Firestore (`runTransaction`):**
     - Documento `games_cache_locks/central_{referenceDateBrt}` com `{ ownerInstanceId, leaseUntilMs, updatedAt }`.
     - Apenas a instância que adquirir o *lease* (ex.: TTL `90s` para fixtures) executa a coleta externa e publica no L2 (`games_cache_snapshots`).
     - As demais instâncias leem o novo documento L2 quando `snapshotVersion` / `generatedAt` avançar (com *jitter* aleatório de `1–3s` nos timers para evitar *thundering herd*).
     - Se a instância líder cair no meio do lease (`now > leaseUntilMs`), qualquer instância assume imediatamente após expiração do lease.

### 4.5. Graceful Shutdown

- Registrar handlers para `SIGTERM` e `SIGINT` em `server.ts`:
  - Interromper ambos os schedulers (`clearInterval` / `clearTimeout`), definir flag `isShuttingDown = true` para abortar reagendamentos pendentes e liberar o event loop limpo (fundamental também para que suítes de teste com `node:test` / `tsx` encerrem sem handles pendentes).

---

## 5. Persistência Opcional L2 no Firestore: Limites, Custos e Segurança

### 5.1. Análise de Tamanho vs Limite de `1 MiB` (`1.048.576 bytes`) do Firestore

Com base nas medições reais de produção da Etapa `2B.5.2.2` (`166` jogos no domingo) e na escala máxima observada aos sábados (`301+` jogos no baseline de `21/11/2026`):

| Tipo de Payload | Tamanho Medido (166 jogos) | Projeção Sábado Cheio (~350 jogos + boxscores) | Viabilidade no Doc Firestore (`1 MiB` max) |
|---|---|---|---|
| **Shadow JSON bruto** (`providerSnapshots`, `fieldProvenance`, `divergences`, `players`) | **`755.557 bytes`** (~738 KiB) | **`~1,4 – 1,8 MiB`** | **INVIÁVEL** (excede 1 MiB e falha na escrita). |
| **Central JSON (`includeStats=full`)** | `~500+ KiB` (com `players` americanos) | **`~1,0 – 1,4 MiB`** | **RISCO ALTO** de estourar 1 MiB. |
| **Central JSON (`includeStats=summary`)** | **`247.438 bytes`** (~242 KiB) | `~480 – 550 KiB` | Viável sem `players`/`providerSnapshots`, porém ainda grande para gravações frequentes. |
| **Snapshot Enxuto Comprimido (`zlib.gzipSync` -> `Bytes`)** | **`~45 – 75 KiB`** (estimativa conservadora gzip 5–8× sobre JSON repetitivo) | **`~90 – 150 KiB`** | **IDEAL** (margem de **7× a 10×** abaixo do limite de 1 MiB!). |

### 5.2. Estrutura Recomendada do Documento L2 (Firestore)

- **Coleção dedicada:** `games_cache_snapshots` (documentada no `firebase-blueprint.json` quando implementada), chaveada por data BRT ou documento ativo + documento por data (`central_active` ou `{referenceDateBrt}`), mantendo histórico mínimo (apenas data atual + limpeza automática de chaves antigas).
- **O que DEVE ser persistido no L2:**
  - Cabeçalho leve não comprimido (consultável e inspecionável sem descomprimir):
    - `schemaVersion`: `1`
    - `referenceDateBrt`: `"YYYY-MM-DD"` (usado na validação obrigatória de boot)
    - `snapshotId`: `string`
    - `generatedAt`: ISO UTC
    - `fixturesCollectedAt`: ISO UTC
    - `liveUpdatedAt`: ISO UTC
    - `fixturesHash`: SHA-256
    - `overlayHash`: SHA-256
    - `degraded`: `boolean`
    - `counts`: `{ today, overnight, tomorrow, live }`
    - `encoding`: `"gzip+json"`
    - `uncompressedBytes`: `number`
    - `compressedBytes`: `number`
  - Campo binário **`payloadGzip`** (`Bytes` / `Uint8Array` nativo do Firestore, sem inflar +33% com base64; limite de segurança rígido antes de gravar: `compressedBytes <= 850_000 bytes`).
  - **Conteúdo dentro de `payloadGzip`:**
    - `yesterdayBrt`, `tomorrowBrt`, `providerMeta`, `providerStatus`, `warnings`, `errors`, `reconciliationSummary`, `excluded`.
    - Lista `canonicalMatches` enxuta: mantém `fixture`, `liveOverlay`, `heroCompat`, `stats` (`summary` + `matchStatsId` + flags `hasTeamStats`/`hasPlayerStats`), `provenance` resumida — **removendo** `providerSnapshots` brutos e arrays gigantes de `players` boxscore (que podem ser repovoados na primeira passada da Camada B/A para ligas americanas e nunca impedem `/api/match-stats`).
- **Verificação pré-escrita:** se mesmo comprimido o blob ultrapassar `850 KB`, aplica-se *fallback* automático removendo `teamStatistics`/`provenance` detalhada antes de abortar a escrita L2 (garantindo que nunca lance erro de tamanho no Firestore).

### 5.3. Política de Escrita e Cota (Spark / Free Tier Friendly)

**Proibição explícita:** nunca gravar no Firestore a cada tick live de 30s (`2.880 writes/dia` desperdiçariam cota de escrita gratuitamente).

Regras de persistência em L2:
1. **Após Camada A (`Fixtures`, cada 12 min):** grava somente se `fixturesHash !== lastPersistedFixturesHash` **ou** se a última escrita no L2 tiver mais de `12 min` (`~120 writes/dia` máx.).
2. **Após Camada B (`Live Overlay`, cada 30–60s):** **não grava** se nada mudou (`overlayHash` igual). Quando houver alteração real de placar ou transição de status (`UPCOMING -> LIVE` / `LIVE -> FINAL`), aplica **debounce mínimo de `90–120s`** desde a última escrita live no Firestore (ou grava imediatamente apenas em transição crítica de placar/status respeitando *rate limit* máximo de 1 escrita/minuto).
3. **Leituras no boot:** apenas **`1` leitura (`getDoc`)** por inicialização de instância!
4. **Impacto estimado na cota Spark (`20k writes/dia`, `50k reads/dia`):** `~120 a 400 writes/dia` (< 2% da cota diária gratuita) e `< 50 reads/dia`.

### 5.4. Restrição Crítica de Autenticação Firestore no `server.ts`

Na auditoria de `package.json`, `server.ts:6-88` e `firestore.rules`:
- O backend atual inicializa o **Firebase Client SDK (`firebase/firestore`)** sem `firebase-admin`.
- Sem sessão autenticada, as gravações do `server.ts` chegam às Security Rules com `request.auth == null`.
- **Requisito obrigatório de segurança para implementação futura:** **não** utilizar `allow write: if true;` na coleção `games_cache_snapshots` (isso permitiria que qualquer usuário anônimo na web adulterasse placares e jogos servidos no boot!).
- **Alternativas viáveis para escolher antes da implementação:**
  - **Alternativa A (Recomendada para Client SDK atual — compatível com qualquer tier sem `firebase-admin`):** configurar uma conta de serviço dedicada via Firebase Auth (`signInWithEmailAndPassword` em `server.ts` usando variáveis exclusivamente backend `FIREBASE_CACHE_WRITER_EMAIL` / `FIREBASE_CACHE_WRITER_PASSWORD` ou Custom Auth), e restringir `firestore.rules` para `allow write: if request.auth != null && request.auth.uid == "<CACHE_WRITER_UID>"; allow read: if false;` (ninguém do frontend lê nem escreve direto nessa coleção).
  - **Alternativa B (`firebase-admin` no Render):** usar `firebase-admin` com `GOOGLE_APPLICATION_CREDENTIALS` / Service Account JSON em variável de ambiente do Render (somente se o projeto GCP/Firebase suportar chaves de conta de serviço no plano ativo), ignorando regras client-side e mantendo `allow read, write: if false;` no `firestore.rules`.
- **Fallback se Firestore estiver indisponível:**
  - Qualquer falha de rede, cota (`resource-exhausted`), permissão (`permission-denied`) ou ausência de credenciais **apenas loga warning controlado** e segue 100% operacional usando **RAM L1** (+ opcional cache comprimido efêmero em `os.tmpdir()/orbita-central-cache-${date}.json.gz` como *fast-path* em reinícios locais). A indisponibilidade do Firestore **jamais** pode derrubar `GET /api/games/central` ou o `server.ts`.

---

## 6. Observabilidade, Métricas e Exposição Segura no `/api/health`

### 6.1. Métricas Internas em Memória

O orquestrador manterá estrutura determinística `CacheObservabilitySnapshot`:

```ts
export interface PrewarmedCacheObservability {
  enabled: boolean;
  bootWarmup: {
    status: "pending" | "ready" | "degraded" | "failed";
    source: "none" | "l2_firestore" | "l1_collection";
    startedAt: string | null;
    completedAt: string | null;
    durationMs: number | null;
  };
  activeSnapshot: {
    snapshotId: string | null;
    referenceDateBrt: string | null;
    generatedAt: string | null;
    ageMs: number | null;
    degraded: boolean;
    stale: boolean;
    counts: { today: number; overnight: number; tomorrow: number; live: number };
  };
  fixturesCycle: {
    intervalMs: number;
    inFlight: boolean;
    lastRunAt: string | null;
    lastSuccessAt: string | null;
    lastDurationMs: number | null;
    runsTotal: number;
    failuresTotal: number;
  };
  liveCycle: {
    currentIntervalMs: number;
    mode: "active" | "idle";
    inFlight: boolean;
    lastTickAt: string | null;
    lastSuccessAt: string | null;
    lastChangeAt: string | null;
    lastDurationMs: number | null;
    ticksTotal: number;
    noOpTicksTotal: number;
    changedTicksTotal: number;
    failuresTotal: number;
  };
  l2Persistence: {
    enabled: boolean;
    lastReadStatus: "none" | "hit" | "miss" | "stale_date" | "error";
    lastWriteAt: string | null;
    lastWriteStatus: "none" | "ok" | "skipped_unchanged" | "skipped_debounced" | "error";
    lastCompressedBytes: number | null;
  };
  requests: {
    centralServedFromPrewarm: number;
    centralFallbackCollections: number;
    internalRefreshAccepted: number;
    internalRefreshCoalesced: number;
  };
}
```

### 6.2. Resumo Seguro em `GET /api/health`

Preservando 100% os campos atuais (`status`, `uptime`, `timestamp`, `service`, `environment` em `server.ts:766-774`), adicionar um bloco enxuto `cache`:

```json
{
  "status": "ok",
  "uptime": 1245.32,
  "timestamp": "2026-11-23T14:00:00.000Z",
  "service": "Órbita Arena API",
  "environment": "production",
  "cache": {
    "warmupStatus": "ready",
    "warmupSource": "l1_collection",
    "referenceDateBrt": "2026-11-23",
    "snapshotId": "ug:2026-11-23:1795442400000:4",
    "snapshotAgeMs": 14200,
    "degraded": false,
    "counts": { "today": 142, "overnight": 3, "tomorrow": 38, "live": 11 },
    "liveMode": "active",
    "liveIntervalMs": 30000,
    "fixturesInFlight": false,
    "liveInFlight": false,
    "l2Status": "ok"
  }
}
```

**Regras de segurança no `/api/health`:** nunca expor chaves de API, stack traces, URLs autenticadas, mensagens brutas de erro interno ou tokens.

---

## 7. Critérios de Aceite Obrigatórios (Matriz de Verificação)

Todos os 12 critérios abaixo deverão passar obrigatoriamente na implementação futura (via suíte automatizada determinística com relógio simulado `now` e `fetchFn` mockado + validação pós-deploy no Render):

| # | Cenário / Requisito | Critério Objetivo de Aprovação |
|---|---|---|
| **1** | **Primeiro acesso pós-boot warmup** | Após conclusão do warmup de boot (L1 ou L2 válido), o 1º `GET /api/games/central` responde imediatamente (`from prewarm`, **zero** novas chamadas externas aos provedores) com tempo de processamento em servidor `< 50 ms` (esperado `< 200 ms` ponta a ponta). |
| **2** | **Leitura recorrente em cache quente** | Requisições consecutivas para Central (`GET`/`HEAD`) e Shadow com diferentes filtros (`sport`, `league`, `status`, `includeStats`) executam 100% em RAM com `0` chamadas HTTP upstream. |
| **3** | **Refresh periódico de fixtures (`10–15 min`)** | Ao avançar o relógio para o intervalo de fixtures (12 min), a grade é re-preparada mantendo `TODAY`, `OVERNIGHT`, `TOMORROW`, estádios, transmissões, logos e reconciliação americana consistentes; mudanças de grade alteram `fixturesHash` e `snapshotId`. |
| **4** | **Refresh periódico live (`30–60s`) e `ETag`** | Tick live consulta apenas endpoints dinâmicos (`/events/live/`, `incidents` com novo gol, scoreboards de ligas ativas), atualiza placar/status/tempo/`homeGoals` e gera novo `snapshotId`/`ETag`; quando nada muda, mantém mesmo `ETag` permitindo `304 Not Modified`. |
| **5** | **Backoff inteligente com `liveCount === 0` e saída de `LIVE`** | Com `liveCount === 0` e nenhum jogo iminente, intervalo sobe para `120–180s`; ao entrar na janela de kickoff ou surgir `LIVE`, volta para `30–45s`. Jogo que sai de `/events/live/` tem seu estado `FINAL` confirmado sem congelar em `LIVE`. |
| **6** | **Resiliência contra falha parcial / truncamento** | Falha na página 2 do Bzzoiro ou erro de provedor durante refresh de fixtures ou live **nunca** apaga o último snapshot completo da mesma data BRT; mantém dados íntegros, marca `degraded: true` e expõe erro em `providerStatus`. |
| **7** | **Deduplicação (Scheduler + Cronjob 14 min + Concorrência)** | Chamadas simultâneas de warmup, scheduler de 12 min, cronjob de 14 min e múltiplos requests concorrentes compartilham a mesma execução `inFlight` (ou respeitam cooldown de `already_fresh`), executando apenas 1 coleta upstream. |
| **8** | **Proteção autenticada do endpoint de refresh** | `POST /api/internal/cache/refresh` retorna `401`/`403` sem `Bearer <CACHE_REFRESH_SECRET>` válido (comparação via `crypto.timingSafeEqual`) e `503` se segredo não configurado; `/api/health` nunca dispara coleta pesada. |
| **9** | **L2 Firestore (limite `< 1 MiB`, throttle e fallback)** | Snapshot serializado + gzip fica `< 850 KB`; ticks sem mudança não escrevem no Firestore; escritas live respeitam debounce; no boot, L2 só hidrata se `referenceDateBrt === getBrtTodayStr()`; se Firestore falhar, API funciona normalmente em RAM L1. |
| **10** | **Virada de meia-noite BRT (`00:00 America/Sao_Paulo`)** | Ao virar `23:59 -> 00:00 BRT`, detector aciona coleta imediata para a nova data BRT, descartando/arquivando snapshot do dia anterior tanto na RAM quanto na validação de leitura L2. |
| **11** | **Paridade integral de contrato (`1.0`, Shadow, `matchStatsId`)** | `GET`/`HEAD /api/games/central`, `ETag`, `304`, `Cache-Control`, `400`, `405`, `stats.matchStatsId` (inclusive pós-hidratação L2), `sets` e `GET /api/games/shadow` mantêm 100% de compatibilidade com os testes das Etapas `2A` a `2B.5.2.4`. |
| **12** | **Graceful shutdown (`SIGTERM`/`SIGINT`)** | Parada controlada limpa todos os timers (`clearInterval`/`clearTimeout`) e cancela agendamentos futuros sem erros ou handles abertos. |

---

## 8. Riscos, Trade-offs e Pontos de Atenção Antes de Implementar

1. **Jogos que saem do `/events/live/` ao terminar:** no Bzzoiro, partidas encerradas deixam `/events/live/`. Sem a checagem pontual de `missingLiveIds` (`/events/{id}/` ou refresh leve de `today`), um jogo encerrado ficaria preso em `LIVE` até o próximo ciclo de 12 min de fixtures. Essa verificação pontual é **mandatória** na Camada B.
2. **Boxscores pesados (`players`) no L2 Firestore:** para cumprir com folga o limite de `1 MiB`, o documento L2 comprimido deve armazenar `canonicalMatches` enxutos (com `stats.matchStatsId` e flags preservadas, sem arrays massivos de `players` e sem `providerSnapshots`), re-hidratando `players`/`teamStatistics` via SportsBlaze na primeira passada de Camada B/A após restart.
3. **Autenticação Firestore no backend (`server.ts`):** como o projeto usa Client SDK (`firebase/firestore`) sem `firebase-admin`, antes de gravar `games_cache_snapshots` em produção no Render é necessário decidir entre **Alternativa A** (usuário de serviço Firebase Auth no backend + regra `request.auth.uid == ...`) ou **Alternativa B** (`firebase-admin` via conta de serviço no Render). Sem isso, ou as regras bloquearão a escrita (`PERMISSION_DENIED`) ou ficarão inseguras (`allow write: if true`).
4. **Conflito de horário com o Sync Loop legado (`server.ts:2256`):** o antigo loop de 30s gravando em `realTimeScores` continuará intacto caso ainda sirva overrides de operadores no carrossel, mas **não** deve ser misturado com a coleção `games_cache_snapshots` nem compartilhar os mesmos locks.
5. **Cold start do próprio container no Render:** o cache antecipado elimina o tempo de espera das APIs esportivas quando o serviço está de pé. Se o Render hibernar por falha do cronjob de 14 min, ainda existirá ~1–3s de spin-up de container Node + hidratação L2 (~200–500 ms), muito inferior aos ~17–44s atuais sem L2.

---

## 9. Plano Sugerido de Implementação Futura (Após Aprovação)

### Fase 1 — Núcleo de Cache Antecipado em RAM + 3 Camadas + Rota Protegida (Prioridade Máxima)
- Adicionar caminho rápido `collectBzzoiroLiveOverlayDelta` em `src/server/collectors/bzzoiroCollector.ts` (`/events/live/` + `incidents` delta + confirmação de `missingLiveIds`).
- Expor helpers de composição parcial/merge (`buildFixtureGradeSnapshot`, `applyLiveOverlayToFixtures`, `composeUnifiedResolution`) em `src/server/collectors/unifiedGamesCollector.ts` garantindo zero regressão na `collectUnifiedGamesSnapshot` síncrona.
- Criar `src/server/cache/prewarmedGamesCache.ts` com as 3 camadas, backoff adaptativo (`30–45s` / `120–180s`), detector de virada BRT, locks `inFlight`, observabilidade e `shutdown()`.
- Integrar leitura prioritária em `src/server/centralGamesApi.ts` (`date === todayBrt`) e em `/api/games/shadow` + registrar `POST /api/internal/cache/refresh` (autenticado com `timingSafeEqual`) em `src/server/registerCollectorRoutes.ts`.
- Conectar boot warmup não bloqueante, resumo seguro no `GET /api/health` e `SIGTERM`/`SIGINT` em `server.ts`, além de documentar `CACHE_REFRESH_SECRET` e intervalos opcionais em `.env.example`.

### Fase 2 — Persistência Resiliente L2 no Firestore + Regras
- Criar `src/server/cache/firestoreSnapshotStore.ts` (serialização enxuta + `zlib.gzipSync` + verificação `< 850 KB` + validação estrita `referenceDateBrt === getBrtTodayStr()` + debounce/hash diff + *lease* opcional multi-instância).
- Atualizar `firebase-blueprint.json` e `firestore.rules` (definindo autenticação segura de escrita backend conforme decisão entre Alternativa A ou B) utilizando o workflow Firebase do projeto.

### Fase 3 — Testes Determinísticos e Validação em Produção
- Criar suíte `tests/etapa2b526.test.ts` cobrindo todos os 12 critérios de aceite da Seção 7 com mocks determinísticos de rede e relógio.
- Publicar backend no Render, configurar `CACHE_REFRESH_SECRET` no Render e atualizar (ou manter em paralelo) o cronjob de 14 minutos, validando métricas reais de TTFT/TTFB no primeiro acesso pós-restart antes da migração final do frontend (`JogosDoDiaPage.tsx`).

### Arquivos Previstos para Alteração na Implementação Futura (Não Alterados Agora)
- **Novos arquivos:**
  - `src/server/cache/prewarmedGamesCache.ts`
  - `src/server/cache/firestoreSnapshotStore.ts` *(Fase 2)*
  - `tests/etapa2b526.test.ts`
- **Arquivos backend a modificar futuramente:**
  - `src/server/collectors/bzzoiroCollector.ts`
  - `src/server/collectors/unifiedGamesCollector.ts`
  - `src/server/centralGamesApi.ts`
  - `src/server/registerCollectorRoutes.ts`
  - `server.ts`
  - `.env.example`
  - `firebase-blueprint.json` e `firestore.rules` *(Fase 2)*
- **Intocados (fora do escopo):** todo o frontend (`src/pages/JogosDoDiaPage.tsx`, `src/App.tsx`, `src/services/**`, `src/components/**`) e lógica central de datas/conjuntos (`src/utils/brtDate.ts`, `src/utils/gameSets.ts`).
