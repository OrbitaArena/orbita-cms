# ÓRBITA ARENA — ETAPA 2B.5.2.4
## Otimização de Performance do Coletor Bzzoiro

**Data:** domingo, 22/11/2026  
**Escopo:** Exclusivamente otimização de performance, paralelismo controlado, redução de requisições HTTP redundantes e observabilidade de gargalos no coletor Bzzoiro (`bzzoiroCollector.ts`) e no agregador (`unifiedGamesCollector.ts`), preservando integralmente regras BRT, sets (`TODAY`, `OVERNIGHT`, `TOMORROW`), integridade/cache da Etapa 2B.5.2.3, contrato Central `1.0`, rotas legadas, `stats.matchStatsId` e frontend intocado.

---

## 1. Diagnóstico Detalhado do Gargalo

A auditoria pós-deploy da **Etapa 2B.5.2.2** em produção (`https://orbitaarena-v2.onrender.com`) e a inspeção de código da **Etapa 2B.5.2.3** comprovaram que a latência elevada da primeira coleta (`~30s–44,35s`, além de `17,36s` após expiração do `footballCache` de `180s`) decorria de seis fatores estruturais dentro de `src/server/collectors/bzzoiroCollector.ts`:

1. **Preload redundante em cold start (`3×` concorrentes):**  
   O agregador (`unifiedGamesCollector.ts`) dispara 3 chamadas simultâneas para `[yesterday, today, tomorrow]`. Como `ensureBsdLeaguesLoaded` e `ensureBsdVenuesLoaded` não tinham *in-flight coalescing*, as 3 chamadas disparavam paralelamente as mesmas requisições iniciais para `/api/v2/leagues/?page_size=100` e `/api/v2/venues/?page_size=100` (`3×` chamadas duplicadas).
2. **Estágio inicial sequencial:**  
   Para `today`, `/events/live/?page_size=100` era aguardado de forma bloqueante **antes** de iniciar a paginação `/events/?page_size=100&date_from=...`.
3. **Enriquecimento executado antes dos filtros de descarte:**  
   Os eventos brutos da janela UTC (`02:00:00Z` até `04:00:00Z` do dia seguinte) incluíam jogos de dias adjacentes (padding de fuso) e ligas finlandesas (`league_id === 55` / palavras-chave). Todos passavam por `resolveMissingVenues`, `resolveMissingLeagues` e `getBsdBroadcastsForEvents` **antes** de serem descartados em `processMatchObj`.
4. **N+1 massivo em partidas de `yesterday` descartadas logo após a coleta:**  
   Em dias cheios (ex.: `21/11/2026` com ~199 jogos, maioria `FINAL` com gols), o coletor consultava centenas de chamadas `GET /api/v2/broadcasts/?event_id=...`, `/venues/...`, `/leagues/...` e `/events/{id}/incidents/` para partidas `FINAL` de ontem que seriam 100% descartadas para o conjunto `NONE` em `partitionGamesIntoSets` (`gameSets.ts`) — mantendo em `OVERNIGHT` apenas partidas ainda `AO VIVO` após a meia-noite ou `FINAL` com timestamp explícito e confiável de término pós-meia-noite.
5. **Barreiras sequenciais rígidas e lotes em cascata no enriquecimento:**  
   `venues` + `leagues` (sem limite de concorrência em `Promise.all`) $\rightarrow$ `broadcasts` (lotes rígidos sequenciais de `25` onde 1 resposta lenta segurava todo o lote) $\rightarrow$ `incidents` (lotes rígidos sequenciais de `10`, sem cache nem para jogos `ENCERRADO`).
6. **Ausência de telemetria granular para coletas lentas (`> 10s`):**  
   Não existia medição por suboperação no Bzzoiro nem quebra de tempo por provedor no `unifiedGamesCollector.ts` quando a coleta ultrapassava `10s`.

---

## 2. Arquivos Modificados

| Arquivo | Tipo de Alteração | Escopo |
|---|---|---|
| `src/server/collectors/bzzoiroCollector.ts` | Modificado | Coalescing de `leagues`/`venues`/`broadcasts`, pool rolante limitado (`16` slots), execução concorrente de `live + events + preload`, pré-filtragem antes do enriquecimento, `enrichmentScope` (`"all"` vs `"overnight_candidates"`) com cache isolado, `venues+leagues+broadcasts+incidents` paralelos, cache de `incidents` apenas para `ENCERRADO` (`5 min`) e `timingsMs` + log `[Bzzoiro Perf]`. |
| `src/server/collectors/unifiedGamesCollector.ts` | Modificado | Passagem de `enrichmentScope: "overnight_candidates"` exclusivamente na chamada de `yesterdayBrt` (quando diferente de `referenceDateBrt`) e medição de duração por provedor/data com log `[Unified Collector Perf]` acima de `10s`. |
| `tests/etapa2b524.test.ts` | Criado | Suíte determinística (offline) validando os 10 cenários exigidos pela Etapa 2B.5.2.4. |
| `temp/etapa-2b-5-2-4-otimizacao-bzzoiro.md` | Criado | Este relatório técnico. |

*Confirmado:* **zero alterações** em `src/utils/brtDate.ts`, `src/utils/gameSets.ts`, `src/server/centralGamesApi.ts`, rotas legadas, `.env` ou frontend (`JogosDoDiaPage.tsx` e demais arquivos `src/App.tsx`, `src/pages/**`, `src/services/**`).

---

## 3. Mudanças no Fluxo de Requisições HTTP (Antes vs Depois)

### Antes (Etapa 2B.5.2.3)
Para cada data em `[yesterday, today, tomorrow]`:
1. `preloadMetadata` (`leagues/` + `venues/`) disparado 3× sem deduplicação entre chamadas concorrentes.
2. **Espera** `GET /events/live/` (em `today`, até `3.500ms`).
3. **Depois**, paginação sequencial `GET /events/?page=1..N` (`4.000ms`/página).
4. **Depois**, `Promise.all` irrestrito em `resolveMissingVenues` + `resolveMissingLeagues` sobre `rawResults` **não filtrado**.
5. **Depois**, lotes sequenciais de `25` em `getBsdBroadcastsForEvents` para todos os eventos de `yesterday`, `today` e `tomorrow`.
6. **Depois**, processamento e filtragem das ligas/datas/Finlândia.
7. **Depois**, lotes sequenciais de `10` em `/events/{id}/incidents/` para todas as partidas com placar `> 0`, inclusive `~140+` partidas `FINAL` de `yesterday` (sem cache nem reuso).

### Depois (Etapa 2B.5.2.4)
1. **Estágio 1 paralelo (`t = 0`):**  
   `preloadPromise` (com coalescing por `fetchFn` via `WeakMap`, reduzindo `3×` para `1×`), `liveFetchPromise` (`/events/live/`, apenas para `todayBrt`) e `eventsPaginationPromise` (`/events/?page=1..N`) executam **concorrentemente**, aplicando merge determinístico (`/events/` base + overlay de `/events/live/` com `_isFromLiveEndpoint = true`).
2. **Pré-filtragem imediata:**  
   Antes de qualquer subchamada de enriquecimento, eventos fora da `targetDateStr` (exceto `_isFromLiveEndpoint`), padding do intervalo UTC `02:00Z–04:00Z` e ligas finlandesas identificáveis são removidos em `survivingRawEvents`.
3. **Seleção de escopo para `yesterday` (`overnight_candidates`):**  
   Quando `unifiedGamesCollector` coleta `yesterdayBrt` apenas para composição de `OVERNIGHT`, `collectBzzoiroFootballToday` recebe `enrichmentScope: "overnight_candidates"`:
   - Todas as partidas sobreviventes de ontem continuam retornando no payload para `partitionGamesIntoSets` e auditoria `rawItemCount`.
   - Apenas candidatas reais de `OVERNIGHT` (`AO VIVO` ou `FINAL` com timestamp explícito `endDate`/`endTime`/`finishedAt`/`completedAt` em BRT $\ge$ data de referência) entram em `enrichableEvents` e disparam `venues`, `leagues`, `broadcasts` e `incidents`.
   - A chave de cache de `"overnight_candidates"` usa sufixo `__overnight_<referenceDateBrt>`, mantendo **100% isolado** o cache padrão de `GET /api/football/today?date=<yesterday>` (`"all"`).
4. **Estágio 2 totalmente paralelo com pool rolante limitado (`16` slots):**  
   `resolveMissingVenues`, `resolveMissingLeagues`, `getBsdBroadcastsForEvents` (agora com deduplicação por `event.id` e coalescing in-flight por `fetchFn`) e `/events/{id}/incidents/` executam **simultaneamente** compartilhando um semáforo rolante (`BZZOIRO_MAX_ENRICHMENT_CONCURRENCY = 16`, sem barreiras por chunk).
5. **Cache curto de `incidents` apenas para jogos `ENCERRADO`:**  
   Partidas `ENCERRADO` gravam `EVENT_INCIDENTS_CACHE` por `5 minutos` (`BZZOIRO_INCIDENT_CACHE_TTL_MS = 300_000 ms`, zerado em `clearBzzoiroCaches()`). Partidas `AO VIVO` **nunca** leem nem gravam esse cache.

---

## 4. Estratégia de Paralelismo e Limites Aplicados

| Controle | Valor / Mecanismo | Objetivo |
|---|---|---|
| Pool rolante de enriquecimento (`BZZOIRO_MAX_ENRICHMENT_CONCURRENCY`) | **`16` slots simultâneos** por coleta (`createBzzoiroConcurrencyLimiter`, configurável em testes via `maxEnrichmentConcurrency`) | Eliminar *head-of-line blocking* dos antigos lotes rígidos de 25 e 10 sem causar explosão de conexões/rate-limit no Bzzoiro. |
| Coalescing de preload | `WeakMap<typeof fetch, Promise<boolean>>` em `ensureBsdLeaguesLoaded` e `ensureBsdVenuesLoaded` | Garantir no máximo `1` execução simultânea de `/leagues/` e `/venues/` por cold start concorrente de 3 datas. |
| Coalescing de `broadcasts` + deduplicação de IDs | `WeakMap<typeof fetch, Map<string, Promise<...>>>` + `Set` por `eventId` | Evitar chamadas duplicadas quando a mesma partida aparece concorrentemente em janelas adjacentes. |
| Cache de `incidents` somente `ENCERRADO` | `EVENT_INCIDENTS_CACHE` (`TTL = 5 min`) | Reaproveitar artilheiros imutáveis em revalidações próximas após expirar `footballCache`, sem jamais congelar gols de jogos `AO VIVO`. |
| Preservação integral de timeouts e integridade | Timeouts individuais (`live 3.5s`, `page 4s`, `meta 3.5s`, `broadcast 2s`, `incident 3s`) e mesmas regras de proteção de cache da 2B.5.2.3 | Garantir que falhas parciais/truncadas nunca contaminem `footballCache` (`180s`) nem sobrescrevam `footballLastCompleteCache`. |
| Telemetria de lentidão (`> 10s`) | `timingsMs` em `BzzoiroTodayIntegrityMeta` + `console.warn("[Bzzoiro Perf] ...")` e `console.warn("[Unified Collector Perf] ...")` | Permitir diagnóstico objetivo em produção no Render separando `bzzoiro` (`live`, `events`, `preload`, `venues`, `leagues`, `broadcasts`, `incidents`), `espn` e `sportsblaze`. |

---

## 5. Comparativo de Latência e Redução de Chamadas HTTP

### 5.1. Dados Reais de Referência (Produção Render — Etapa 2B.5.2.2)
- `GET /api/games/central` (cold start completo): **`44.353 ms`** (~`39s` atribuídos ao Bzzoiro + `~5,07s` atribuídos a ESPN/SportsBlaze).
- Revalidação pós-TTL `180s` do Bzzoiro (com `leagues`/`venues`/`broadcasts` já em cache de 5m/2h): **`17.357 ms`**.
- Revalidação da Central com Bzzoiro ainda quente: **`5.072 ms`**.
- Cache HIT na Central: **`128–169 ms`** (`304 Not Modified`: **`84 ms`**).

### 5.2. Redução Quantitativa Verificada em Código e Testes (`tests/etapa2b524.test.ts`)
*(Medições em ambiente de teste local com fetch mockado determinístico, sem chamadas externas reais)*:
- **Preload inicial (`/leagues/` + `/venues/`):** caiu de até **`6` requisições** (`3` datas × `2`) para **`2` requisições** (`1` coalescida de cada) em cold start concorrente (`2B.5.2.4 [1]`).
- **Enriquecimento N+1 de `yesterday` (`overnight_candidates`):** em teste com `15` jogos `FINAL` comuns de ontem + `2` candidatos reais de `OVERNIGHT` (`LIVE` e `FINAL` com `finishedAt` pós-meia-noite) + `2` itens descartados (padding + Finlândia):
  - **Antes:** disparava subchamadas para todos os `15 + 2 + 2 = 19` eventos (`venues`, `leagues`, `broadcasts` e `incidents` dos gols).
  - **Depois:** **`0` requisições** de `venues`, `broadcasts` e `incidents` para os `15` jogos comuns de ontem e para os `2` itens descartados, enriquecendo apenas os **`2`** candidatos reais de `OVERNIGHT` (`2B.5.2.4 [7 & 8]`).
- **Execução concorrente limitada (`2B.5.2.4 [1]`):** `12` partidas com latência simulada de `20ms` por subchamada + Estágio 1 de `30ms` concluíram em **`< 550 ms`** (vs `> 900 ms` caso fossem em barreiras sequenciais), respeitando pico de concorrência $\le 4$ com `maxEnrichmentConcurrency: 4`.

---

## 6. Testes Executados e Resultados

Execução completa das suítes (`npx tsx --test tests/etapa2a.test.ts tests/etapa2b2.test.ts tests/etapa2b21.test.ts tests/etapa2b3.test.ts tests/etapa2b32.test.ts tests/etapa2b43.test.ts tests/etapa2b523.test.ts tests/etapa2b524.test.ts`):

- **Total executado:** **`76` testes em `8` suítes**
- **Aprovados:** **`76 / 76`**
- **Falhos:** **`0`**

Cobertura validada na nova suíte `tests/etapa2b524.test.ts`:
1. `[1]` Paralelismo controlado respeitando `maxEnrichmentConcurrency`, aceleração real frente a barreiras sequenciais, coalescing de preload (`/leagues/` e `/venues/`) entre datas concorrentes e cache de `incidents` (`5 min`) somente para `ENCERRADO` (nunca para `AO VIVO`).
2. `[2]` Falhas opcionais simultâneas em `broadcasts`, `incidents`, `venues` e `leagues` preservam todas as partidas e sinalizam `enrichment.partial: true`, `partial: true` sem falsamente marcar `degraded: true`.
3. `[3]` Falha na página 2 de `/events/` marca `partial: true`, `truncated: true`, `complete: false`, `degraded: true` e registra `failedPage: 2`.
4. `[4]` Cache saudável (`180s`) **não** é contaminado por resposta truncada nem por falha parcial de enriquecimento.
5. `[5]` Preservação do `footballLastCompleteCache` sem sobrescrita por coleta parcial e mantendo o `collectedAt` original em stale fallback.
6. `[6]` Falha isolada apenas em `yesterday` preserva jogos de `today`, `tomorrow`, ESPN e SportsBlaze com `partial: true` e `degraded: true`.
7. `[7 & 8]` Preservação exata de `TODAY`, `OVERNIGHT` e `TOMORROW`; enriquecimento completo (estádio, transmissão, artilheiros) de `LIVE` e `FINAL` com `finishedAt` pós-meia-noite em `OVERNIGHT`; descarte de `FINAL` comuns de ontem/padding/Finlândia sem N+1 HTTP; e isolamento de cache entre `"overnight_candidates"` e `GET /api/football/today?date=<yesterday>`.
8. `[9]` Recuperação imediata na Central após expiração do TTL degradado de `20s`, mais verificação de geração de `timingsMs` e logs estruturados `[Bzzoiro Perf]` e `[Unified Collector Perf]` acima de `10s`.
9. `[10]` Contrato Central `1.0` (`ETag`, `304 Not Modified`, `Cache-Control`, `stats.matchStatsId`, independência ESPN/SportsBlaze) e rotas legadas (`/api/football/today`, `/api/jogos-texto`, `/api/jogos-lite`) 100% íntegros.

---

## 7. Riscos e Limitações Restantes

1. **Paginação sequencial cursor-based do Bzzoiro (`next`):**  
   A API `/api/v2/events/` utiliza paginação DRF por URL `next` (cursor/página dependente da resposta anterior), impedindo prefetch paralelo das páginas 2..N antes de receber a página 1. Se a página 2 no servidor upstream do Bzzoiro exceder o timeout de `4.000 ms`, a coleta continua sujeita à proteção de truncamento (`partial: true, truncated: true, degraded: true`).
2. **Latência externa dos provedores no Render:**  
   Embora a quantidade de requisições e barreiras em cascata no Bzzoiro tenha sido drasticamente reduzida, o tempo real da primeira coleta fria no Render depende também das latências simultâneas das APIs externas (`sports.bzzoiro.com`, `site.api.espn.com` e `cache.sportsblaze.com` — estas duas últimas observadas em `~5,07s` agregados na Etapa 2B.5.2.2) e do eventual *cold start* de instância do próprio Render.
3. **Sem impacto sobre cache HTTP de CDN (`cf-cache-status: DYNAMIC`):**  
   O cache da Central permanece em memória no processo Node (HIT `~128–169ms`), sujeito a aquecimento após cada novo deploy/restart do serviço no Render.

---

## 8. Impacto Esperado na Primeira Coleta

Com base na eliminação das ~300+ subchamadas de enriquecimento de jogos comuns de ontem, no paralelismo simultâneo de `live + events + preload` e de `venues + leagues + broadcasts + incidents` em pool rolante de `16` slots (sem barreiras por lote de 25/10), o gargalo interno do Bzzoiro deixa de dominar em cascata por `~39s` e tende a convergir próximo à janela natural dos demais provedores (`~5s–8s` na primeira coleta saudável esperada no Render, respeitando sempre que os números reais de produção só poderão ser confirmados após publicação e nova medição no Render).

---

## 9. Lista Exata de Arquivos a Publicar

Para publicação da otimização no backend (mantendo frontend intacto e sem commits/deploy automáticos nesta etapa):

- `src/server/collectors/bzzoiroCollector.ts`
- `src/server/collectors/unifiedGamesCollector.ts`
- `tests/etapa2b524.test.ts`
- `temp/etapa-2b-5-2-4-otimizacao-bzzoiro.md` *(documentação técnica de referência, opcional para o histórico de etapas)*
