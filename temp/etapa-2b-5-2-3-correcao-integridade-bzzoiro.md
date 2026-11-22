# ÓRBITA ARENA — ETAPA 2B.5.2.3
## Correção de Integridade, Falhas Silenciosas e Resiliência do Coletor Bzzoiro

**Data de conclusão:** 22/11/2026  
**Escopo:** `src/server/collectors/bzzoiroCollector.ts`, `src/server/collectors/unifiedGamesCollector.ts`, `src/server/centralGamesApi.ts`, `tests/etapa2b523.test.ts`  
**Restrições respeitadas:** Nenhuma alteração em frontend (`JogosDoDiaPage.tsx`, `src/App.tsx`, `src/services/*`), Vercel, Render, `.env` ou segredos; nenhum `git commit`, `push` ou `deploy` executado; nenhuma chamada HTTP externa em testes.

---

## 1. Resumo Executivo

A Etapa **2B.5.2.3** eliminou os dois defeitos críticos de integridade identificados na auditoria da Etapa 2B.5.2.2 e blindou toda a cadeia de coleta do Bzzoiro até a API Central (`/api/games/central` e `/api/games/shadow`):

1. **Fim das falhas silenciosas mascaradas como sucesso (`fallback_resilience`):** Falhas HTTP, timeouts, erros de rede e payloads inválidos com cache frio deixam de retornar `ok: true` e `totalMatches: 0` com `source: "fallback_resilience"`. Agora retornam `ok: false`, `statusCode: 502` e `source: "bzzoiro_upstream_error"` (ou `503` sem `BZZOIRO_API_KEY`).
2. **Detecção rigorosa de paginação truncada e falha parcial:** A falha na página 2+ (como no incidente real de 22/11/2026: 100 de 146 jogos) ou no endpoint `/events/live/` não é mais tratada como coleta completa. O coletor registra `partial`, `truncated`, `eventsComplete: false`, `failedPage`, motivo e erro detalhado (`scope`, `errorType`, `statusCode`, `page`), propagando degradação real até `providerStatus.bzzoiro`, `degraded: true`, `warnings` e `errors` da Central.
3. **Proteção absoluta dos caches (`footballCache` e `footballLastCompleteCache`):**
   - Coletas truncadas, com falha em `/events/live/` ou falhas totais **nunca** entram no `footballCache` saudável (`180s`).
   - Foi criado o cache separado `footballLastCompleteCache` (janela máxima de `30 min`), que **nunca** é sobrescrito por coletas parciais, truncadas ou falhas, e preserva o timestamp original `collectedAt` quando servido como stale fallback isolado (sem renovar idade falsa).
   - Em truncamento na Central, o snapshot recebe TTL degradado de `20s` (em vez de `60s` saudável ou `180s` contaminado), permitindo recuperação imediata assim que a Bzzoiro estabiliza.
4. **Cobertura completa de timeouts (`fetchWithTimeout`):** Todas as chamadas auxiliares que antes não tinham timeout (`ensureBsdLeaguesLoaded`, `ensureBsdVenuesLoaded`, `resolveMissingVenues`, `resolveMissingLeagues`, `getBsdBroadcastsForEvents`, `incidents` em `collectBzzoiroFootballToday` e `collectBzzoiroFootballFixtures`, e `standings`) passaram a utilizar `fetchWithTimeout` com limites finitos. Falhas em broadcasts não contaminam mais `EVENT_BROADCASTS_CACHE` por `300s`.
5. **Isolamento de falhas e preservação de integridade:** Falha isolada do Bzzoiro ou de apenas uma data (`yesterday`) preserva integralmente ESPN/SportsBlaze e os jogos válidos das demais datas, além de diferenciar integridade da lista de partidas (`eventsComplete`) de falha apenas de enriquecimento (`venues`, `leagues`, `broadcasts`, `incidents`).

---

## 2. Causa Raiz Confirmada

A investigação sobre `src/server/collectors/bzzoiroCollector.ts` e `src/server/collectors/unifiedGamesCollector.ts` confirmou os mecanismos exatos do incidente de **22/11/2026** (100 partidas entregues vs 146 esperadas):

1. **Silenciamento de paginação truncada:**  
   No loop `while (currentUrl && pageCount < maxPages)` de `collectBzzoiroFootballToday`, ao ocorrer erro HTTP (`!response.ok`), JSON inválido ou timeout/erro de rede na página 2+, o loop executava `break` mantendo os 100 eventos da página 1. O restante da função assumia conclusão normal, gravava os 100 eventos em `footballCache` (`180s`) e retornava `ok: true` sem sinalizar `partial`, `truncated` ou erro para `unifiedGamesCollector.ts`.
2. **Mascaramento de falha total (`fallback_resilience`):**  
   Quando a página 1 falhava (HTTP 5xx, timeout, rede ou payload inválido) e o cache estava frio (`rawResults.length === 0`), o coletor devolvia `ok: true`, `statusCode: 200`, `source: "fallback_resilience"` e `leagues: []`. Para `unifiedGamesCollector.ts`, isso parecia um dia legitimamente vazio (`emptySlateCalls++`, `succeededCalls++`), deixando `degraded: false` e escondendo a queda do provedor.
3. **Ausência de `fetchWithTimeout` em enriquecimentos e rotas auxiliares:**  
   As chamadas a `resolveMissingVenues`, `resolveMissingLeagues`, `getBsdBroadcastsForEvents`, `/incidents/` e `standings` usavam `fetch` sem `AbortController`/timeout, podendo pendurar além do SLA do Render. Além disso, `getBsdBroadcastsForEvents` gravava arrays vazios em `EVENT_BROADCASTS_CACHE` mesmo quando a consulta HTTP falhava, suprimindo `ondeAssistir` por `300s`.
4. **Contaminação de cache e ausência de separação de estado completo:**  
   Havia apenas um `footballCache` compartilhado por respostas completas e respostas parciais/vazias sem distinção de integridade, e não existia registro separado do último resultado completo com timestamp preservado.

---

## 3. Alterações no `bzzoiroCollector.ts`

Arquivo: `src/server/collectors/bzzoiroCollector.ts`

### 3.1 Constantes e Orçamento de Timeouts
Foram exportadas constantes explícitas de timeout e retenção:
- `FOOTBALL_CACHE_TTL_MS = 180 * 1000` (apenas coletas íntegras e sem falha de enriquecimento)
- `FOOTBALL_LAST_COMPLETE_MAX_AGE_MS = 30 * 60 * 1000` (janela máxima de stale isolado do último completo)
- `BZZOIRO_LIVE_TIMEOUT_MS = 3500`
- `BZZOIRO_PAGE_TIMEOUT_MS = 4000`
- `BZZOIRO_METADATA_TIMEOUT_MS = 3000`
- `BZZOIRO_BROADCAST_TIMEOUT_MS = 2000`
- `BZZOIRO_INCIDENT_TIMEOUT_MS = 3000`
- `BZZOIRO_STANDINGS_TIMEOUT_MS = 4000`

*(Observação: `footballLastCompleteCache` também é limpo em `clearBzzoiroCaches()`, garantindo isolamento determinístico em testes).*

### 3.2 Paginação e Validação de Payload
- **`/events/live/`:** Usa `fetchWithTimeout(..., liveTimeoutMs)`. Registra `liveAttempted`, `liveOk` e `liveError` (`http_error`, `invalid_payload`, `timeout`, `network_error`). Se falhar com `/events/` íntegro, preserva as partidas de `/events/`, mas marca `isComplete = false`, `degraded = true`, `partial = true` e bloqueia escrita no `footballCache`.
- **`/events/` paginado:** Usa `fetchWithTimeout(..., pageTimeoutMs)`. Valida `response.ok`, parse JSON e presença de array (`Array.isArray(json)` ou `Array.isArray(json.results)`).
  - Rastreia `pagesAttempted`, `pagesSucceeded`, `paginationCompleted`, `hitPageLimit` e `pageFailure` (`page`, `errorType`, `statusCode`, `message`).
  - Detecta anomalia de lista vazia com `next` pendente (`results: [], next: "..."`) como `invalid_payload`.
  - Detecta estouro de `maxPages = 10` com `next` pendente (`hitPageLimit = true`, `pagination_limit`).
  - Só define `eventsPaginationOk = pagesSucceeded > 0 && paginationCompleted && !pageFailure && !hitPageLimit`.
  - Diferencia **vazio legítimo** (`page 1` retorna HTTP 200, `results: []`, `next: null`, `paginationCompleted: true`) de **falha fria/vazia**.

### 3.3 Enriquecimento com Timeouts e Proteção de `EVENT_BROADCASTS_CACHE`
- `ensureBsdLeaguesLoaded` e `ensureBsdVenuesLoaded` passaram a receber `timeoutMs`, retornar `Promise<boolean>`, e só avançar `BSD_*_LAST_LOAD` quando nenhuma falha HTTP/rede interromper o carregamento.
- `resolveMissingVenues` e `resolveMissingLeagues` foram envolvidos em `fetchWithTimeout(..., metadataTimeoutMs)` e retornam `BzzoiroEnrichmentStepResult` (`ok`, `attempted`, `succeeded`, `failed`, `errors`).
- `getBsdBroadcastsForEvents` foi envolvido em `fetchWithTimeout(..., broadcastTimeoutMs)`, valida `response.ok` e formato JSON antes de processar, e **recusa gravar** `EVENT_BROADCASTS_CACHE` em caso de falha HTTP, timeout, erro de rede ou payload inválido — retornando `{ byId, byTeam, meta }`.
- As chamadas de `/incidents/` (tanto em `collectBzzoiroFootballToday` quanto em `collectBzzoiroFootballFixtures`) e `collectBzzoiroFootballStandings` passaram a usar `fetchWithTimeout`.
- Falhas isoladas de enriquecimento (`venues`, `leagues`, `broadcasts`, `incidents`) preservam todos os fixtures válidos coletados e são reportadas em `meta.enrichment.partial = true`, sem mascarar como falha da lista de jogos (`eventsComplete: true`).

### 3.4 Política de Resposta e Proteção de Caches
1. **Falha total (`rawResults.length === 0 && eventsFailedCompletely`):**
   - Se `allowStaleFallback && prevEntry && withinMaxStale` (em `footballLastCompleteCache`, `<= 30 min`): retorna `ok: true`, `statusCode: 200` com `payload` clonado (`stale: true`, `usingPreviousData: true`, `degraded: true`, `complete: false`, `collectedAt: prevEntry.collectedAt`) e `meta.failed = true`, `meta.usingPreviousData = true`, `meta.cacheHit = true`, `meta.cacheAgeMs` real, preservando `prevEntry.collectedAt` sem renovar timestamp.
   - Sem fallback válido (cache frio ou expirado): retorna `ok: false`, `statusCode: 502`, `source: "bzzoiro_upstream_error"`, `complete: false`, `degraded: true`, `leagues: []` — **sem** `fallback_resilience`.
2. **Coleta truncada ou com falha no `/events/live/` (`!isComplete`):**
   - Retorna `ok: true`, `statusCode: 200` preservando as partidas válidas já coletadas, com `complete: false`, `partial: true`, `truncated: isTruncated`, `degraded: true` e `meta` detalhado.
   - **Nunca** grava em `footballCache` nem em `footballLastCompleteCache`.
3. **Coleta de eventos completa (`isComplete === true`), mas `enrichmentPartial === true`:**
   - Grava `footballLastCompleteCache` (lista completa de jogos preservada), mas **não** grava em `footballCache` saudável (`180s`), permitindo nova tentativa de enriquecimento na próxima janela da Central.
4. **Coleta 100% íntegra (`isComplete && !enrichmentPartial`):**
   - Atualiza `footballLastCompleteCache` e `footballCache` com cópias clonadas (`structuredClone`).

---

## 4. Alterações no `unifiedGamesCollector.ts` e `centralGamesApi.ts`

Arquivos: `src/server/collectors/unifiedGamesCollector.ts` e `src/server/centralGamesApi.ts`

1. **Inspeção por data com rastreamento livre de corrida (`bzzDateStates`):**
   - Cada chamada na janela `[yesterdayBrt, referenceDateBrt, tomorrowBrt]` preenche seu respectivo índice em `bzzDateStates` (`usedStaleFallback`, `materialPartial`, `enrichmentPartial`, `collectedAt`, `cacheHit`).
2. **Rejeição de `fallback_resilience`:**
   - Se qualquer wrapper retornar `source === "fallback_resilience"` ou `!res.ok`, `bzzReport.failedCalls++` e o erro é registrado com `errorType: "masked_empty_fallback"` (ou `meta.errors`).
3. **Classificação exata de estados do Bzzoiro:**
   - **`isUsingPreviousFallback`:** Incrementa `bzzReport.failedCalls++` (não `succeededCalls`), ativa `usedStaleFallback = true`, preserva os jogos anteriores na unificação e preserva o timestamp mais antigo original em `providerMeta.bzzoiro.collectorFetchedAt` com `sourceCacheHit = true` e `sourceCacheAgeMs` exato.
   - **`isTruncatedOrLiveFailed`:** Incrementa `bzzReport.succeededCalls++` (pois entregou partidas válidas), marca `materialPartial = true` e `bzzReport.partial = true`, adiciona erros detalhados (incluindo sufixo `:p{page}` quando paginado) e emite warning `provider_bzzoiro_partial`.
   - **`isEnrichmentPartial`:** Incrementa `bzzReport.succeededCalls++`, marca `enrichmentPartial = true` e `bzzReport.partial = true`, adiciona warning `provider_bzzoiro_enrichment_partial` e erros de escopo `:incidents` / `:broadcasts` / `:venues` / `:leagues`, **sem** marcar `degraded = true` caso todas as listas de eventos estejam íntegras (`materialPartial === false` e `failedCalls === 0`).
4. **Cálculo de `degraded` e TTL da Central:**
   - `result.degraded` torna-se `true` sempre que `bzzReport.failedCalls > 0`, `bzzMaterialPartialCount > 0`, ou houver falhas em ESPN/SportsBlaze (excluindo 404 esperado de SportsBlaze).
   - Quando `result.degraded === true`, `collectUnifiedGamesSnapshot` aplica automaticamente `degradedTtlMs = 20_000` (`20s`) em vez de `60_000` (`60s`), e `centralGamesApi.ts` envia `Cache-Control: public, max-age=10, must-revalidate` em vez de `max-age=15`.
   - Quando todos os provedores falham (`!anyProviderSucceeded`), `collectUnifiedGamesSnapshot` retorna `result.ok = false` e **nunca** grava no `unifiedSharedCache`, acionando o stale fallback global limitado (`UNIFIED_MAX_STALE_AGE_MS = 180s`, `Cache-Control: no-store`, header `Warning: 110`) se existir snapshot anterior válido, ou `503` em cold start.
5. **Projeção de `warnings` e `freshness` no `centralGamesApi.ts`:**
   - `projectUnifiedSnapshotToCentral` deduplica `resolution.warnings` (`Array.from(new Set([...resolution.warnings]))`) e acrescenta `provider_bzzoiro_partial` quando `degraded && providerStatus.bzzoiro.partial`.
   - Para mocks sem `meta`/`collectedAt` (compatibilidade com testes 2B.4.3), `providerMeta.bzzoiro.collectorFetchedAt` permanece `null` em vez de inventar um timestamp artificial.

---

## 5. Análise de Stale Fallback Isolado do Bzzoiro vs Global da Central

| Cenário | Comportamento | Benefício / Garantia |
|---|---|---|
| **Apenas Bzzoiro cai após coleta completa prévia (`<= 30 min`), ESPN/SportsBlaze operando** | `bzzoiroCollector` serve `footballLastCompleteCache` (`usingPreviousData: true`, `collectedAt` original); ESPN/SportsBlaze continuam atualizando normalmente ao vivo; Central retorna `200`, `degraded: true`, `ttlMs: 20s`, `providerStatus.bzzoiro.ok: false`, `warnings: ["provider_bzzoiro_stale_fallback"]`, preservando futebol + esportes americanos. | Evita zerar futebol quando só o Bzzoiro oscila e impede congelamento de placares ao vivo da NBA/NFL/NHL/MLB/NCAA/PLL. |
| **Bzzoiro cai em cold start (sem `footballLastCompleteCache`) com ESPN/SportsBlaze operando** | `bzzoiroCollector` retorna `ok: false` (`502`, `bzzoiro_upstream_error`); Central entrega jogos americanos íntegros com `degraded: true`, `providerStatus.bzzoiro.ok: false`, `ttlMs: 20s`. | Nunca inventa lista vazia saudável nem bloqueia ligas americanas. |
| **Bzzoiro entrega truncamento (P1=100, P2 falha) após coleta completa prévia** | `bzzoiroCollector` devolve os 100 jogos marcados `partial: true, truncated: true`, **sem** gravar em `footballCache` nem sobrescrever `footballLastCompleteCache`; Central marca `degraded: true`, `ttlMs: 20s`, recupera as 146 partidas na primeira revalidação bem-sucedida após `20s`. | Evita o aprisionamento de 180s do incidente de 22/11/2026 e protege o último snapshot íntegro. |
| **Todos os provedores falham após snapshot saudável da Central expirar (`60s` < idade <= `180s` e sem cache local quente do Bzzoiro)** | `anyProviderSucceeded` resulta `false`; Central ativa `REVALIDATION_FAILED_USING_STALE`, mantendo o último snapshot íntegro completo com `cache.stale: true`, `Cache-Control: no-store`, `Warning: 110`, `cooldownMs: 20s`. | Resiliência global sem sobrescrever o snapshot saudável nem renovar sua idade. |
| **Todos os provedores falham em cold start ou após `staleUntilMs` (`> 180s`)** | Central retorna `503 ALL_PROVIDERS_FAILED` e Shadow retorna `502`, sem cachear erro como sucesso. | Transparência operacional total. |

---

## 6. Estratégia e Cenários de Teste (`tests/etapa2b523.test.ts`)

Foi criada a suíte determinística `tests/etapa2b523.test.ts` (14 blocos de teste, zero chamadas externas reais, cobrindo todos os **18 cenários exigidos**):

1. **[1]** Falha HTTP 500 na página 1 com cache frio não mascara como `ok: true` nem `fallback_resilience` (retorna `502`, `bzzoiro_upstream_error`).
2. **[2]** Classificação correta de timeout, erro de rede (`ECONNRESET`) e JSON inválido na página 1.
3. **[3]** Reprodução fiel do incidente de 22/11/2026 (`P1=100` eventos + falha na `P2` por HTTP 504 e por timeout): `partial: true`, `truncated: true`, `complete: false`, `degraded: true`, `failedPage: 2`, não gravado no `footballCache`.
4. **[4]** Falha na página 3 após páginas 1 e 2 válidas registra `failedPage: 3` e `pagesSucceeded: 2`.
5. **[5]** Resposta legitimamente vazia (`results: [], next: null`) é cacheável com `complete: true`, `partial: false`, `degraded: false`.
6. **[6]** Timeout em `/events/live/` com `/events/` completo preserva os jogos, sinaliza `degraded: true`, `liveEndpoint.ok: false` e não contamina `footballCache`.
7. **[7]** Timeouts/erros em enriquecimento (`venues`, `leagues`, `broadcasts`, `incidents`) preservam fixtures, não cacheiam broadcast falho em `EVENT_BROADCASTS_CACHE` (recuperando na chamada seguinte) e diferenciam integridade na Central (`degraded: false`, `partial: true`, `provider_bzzoiro_enrichment_partial`, e não entra no `footballCache` saudável).
8. **[8 & 9]** Falha total usa `footballLastCompleteCache` sem renovar `collectedAt` (com `cacheAgeMs` crescente) e falha parcial **nunca** sobrescreve o último completo.
9. **[10 & 11]** `footballCache` saudável (`180s`) não é contaminado nem por coleta truncada nem por erro vazio.
10. **[12]** Falha isolada do Bzzoiro na Central preserva ESPN/SportsBlaze tanto em cold start quanto via stale fallback isolado do Bzzoiro (mantendo timestamp original e `sourceCacheAgeMs` real).
11. **[13]** Falha apenas na data `yesterday` no Bzzoiro preserva partidas de `today` e `tomorrow` com `partial: true`, `degraded: true` e erro escopado em `bzzoiro:{yesterday}`.
12. **[14 & 15]** Resposta HTTP `GET /api/games/central` reflete truncamento com `TTL 20s` (`Cache-Control: public, max-age=10`), recupera as 146 partidas após `21s`, e após falha total de todos os provedores devolve stale fallback global (`cache.stale: true`, `no-store`, `Warning: 110`).
13. **[16]** Rotas legadas `/api/football/today` (HTTP `502` em falha fria, recuperação para `200` com 2 jogos) e `/api/jogos-texto` (preservando resposta válida durante falha isolada do Bzzoiro).
14. **[17 & 18]** Freshness honesto (`collectorFetchedAt: null` para mocks sem `meta`; timestamp real no coletor real), preservação de `stats.matchStatsId` (`bsd-7701/7702/7703`) e particionamento em `TODAY`, `OVERNIGHT` e `TOMORROW` sob coleta parcial.

---

## 7. Resultados Executados

| Verificação | Comando / Ferramenta | Resultado | Observações |
|---|---|---|---|
| **TypeScript Lint** | `npm run lint` (`tsc --noEmit`) | **PASSOU (0 erros)** | Co-executado com build final sem erros de tipo ou imports. |
| **Build de Produção** | `npm run build` (`copy-assets.cjs` + `vite build` + `esbuild server.ts`) | **PASSOU (0 erros)** | Bundle client e `dist/server.cjs` compilados com sucesso. |
| **Suíte 2B.5.2.3** | `npx tsx --test tests/etapa2b523.test.ts` | **14/14 PASSARAM (100%)** | Todos os 18 cenários validados deterministicamente. |
| **Suítes Centrais & Contrato** | `npx tsx --test tests/etapa2a.test.ts tests/etapa2b3.test.ts tests/etapa2b32.test.ts tests/etapa2b43.test.ts tests/etapa2b523.test.ts` | **52/52 PASSARAM (100%)** | `2A`, `2B.3`, `2B.3.2`, `2B.4.3` e `2B.5.2.3` integralmente verdes. |
| **Suítes Completas (7 arquivos)** | `npx tsx --test tests/etapa2a.test.ts tests/etapa2b2.test.ts tests/etapa2b21.test.ts tests/etapa2b3.test.ts tests/etapa2b32.test.ts tests/etapa2b43.test.ts tests/etapa2b523.test.ts` | **65 passaram, 2 falharam** | **Nota abaixo** sobre as 2 falhas pré-existentes em `etapa2b2.test.ts` e `etapa2b21.test.ts`. |

> **Nota técnica sobre as 2 falhas pré-existentes (`etapa2b2.test.ts:625` e `etapa2b21.test.ts:413`):**  
> Ambos os arquivos de teste legados chamam `classifyGameForSets(match, refNowDate)` passando um objeto `Date` bruto (`2026-11-20T18:00:00-03:00`) diretamente no 2º argumento, quando a assinatura em `src/utils/gameSets.ts:246` espera `{ referenceDateBrt?: string; now?: Date; source?: TimestampSource }`. Por isso `options?.now` ficava `undefined` e usava o relógio do sistema (`2026-11-20` no dia em que foram escritos, vs `2026-11-22` hoje), classificando fixtures de `20/11` como `'NONE'` em vez de `'TODAY'`. Não é regressão desta etapa e não foram alteradas regras de classificação esportiva.

---

## 8. Compatibilidade com Rotas Legadas & Contratos

- **`GET /api/games/central` e `HEAD /api/games/central`:** Contrato `1.0` integralmente preservado (`canonicalId`, `fixture`, `liveOverlay`, `stats`, `heroCompat`, `counts`, `cache`, `providerStatus`, `providerMeta`, `warnings`, `errors`, `ETag`, `Cache-Control`).
- **`GET /api/games/shadow`:** Continua compartilhando o mesmo snapshot/coalescing sem stale fallback (`allowStaleFallback: false`) e reflete o novo comportamento de `providerStatus.bzzoiro`, `degraded` e `errors`.
- **`GET /api/football/today`:** Preserva formato de payload para consumidores legados; passa a devolver HTTP `502` real em falha fria sem `fallback_resilience`, mantendo campos adicionais informativos (`complete`, `partial`, `truncated`, `degraded`, `stale`, `usingPreviousData`, `collectedAt`) totalmente retrocompatíveis.
- **`GET /api/jogos-texto`:** Continua utilizando ESPN + Bzzoiro; durante falha isolada do Bzzoiro, preserva os jogos americanos válidos sem quebrar a rota.
- **`stats.matchStatsId` (`bsd-{id}`):** Preservado em todos os cenários completos, parciais e de fallback (`heroCompat.statsMatchId` e `stats.matchStatsId`).

---

## 9. Riscos Residuais & Limitações

1. **Orçamento global de latência no Render:** Os timeouts finitos em Bzzoiro (`2s`–`4s`) mitigam travamentos longos, porém páginas com muitos gols que disparem dezenas de lotes sequenciais de `/incidents/` (chunks de 10) ainda podem somar alguns segundos sob alta latência upstream.
2. **Co-ocorrência de `FOOTBALL_CACHE_TTL_MS = 180s` e `UNIFIED_MAX_STALE_AGE_MS = 180s`:** Quando Bzzoiro possui `footballCache` saudável recém-populado (`< 180s`) e Central revalida aos `60s`, Bzzoiro continua servindo seu cache saudável local enquanto apenas os demais provedores vão ao upstream — comportamento intencional para poupar cota da Bzzoiro, mas que implica que uma queda total da Bzzoiro só aciona `footballLastCompleteCache` após os `180s` do cache saudável expirarem.
3. **Sem alteração no frontend (`JogosDoDiaPage.tsx`):** O frontend de produção continua consumindo o fluxo legado até que a etapa dedicada de migração para `/api/games/central` seja autorizada e executada.
4. **Limpeza dos 2 testes legados de `2B.2` / `2B.2.1`:** Os arquivos `tests/etapa2b2.test.ts:623` e `tests/etapa2b21.test.ts:416,427` continuam sensíveis à data do sistema por passarem `Date` em vez de `{ now: Date }` para `classifyGameForSets`.

---

## 10. Parecer Final & Próximos Passos Recomendados

**Parecer Técnico:** **APROVADO para revisão e publicação do backend.**  
O coletor Bzzoiro e sua integração com o `unifiedGamesCollector` e `centralGamesApi` não mascaram mais falhas frias nem paginações truncadas, protegem o cache saudável e o último snapshot completo com timestamp honesto, cobrem todos os endpoints auxiliares com timeouts finitos e passam em 100% dos 18 novos cenários e em todas as suítes da arquitetura Central (`52/52`).

**Próximos passos recomendados (mediante sua aprovação):**
1. Revisar as alterações pendentes do backend (Etapa 2B.5.2.1 + 2B.5.2.3) e publicar o backend atualizado no GitHub / Render.
2. Validar `/api/games/central` em produção no Render confirmando `degraded`, `partial`, `providerStatus.bzzoiro` e contagens compatíveis com a página de produção.
3. Iniciar a etapa de migração controlada do `JogosDoDiaPage.tsx` para consumir `GET /api/games/central` (com tratamento explícito de `degraded` / `partial` na UI se desejado).
