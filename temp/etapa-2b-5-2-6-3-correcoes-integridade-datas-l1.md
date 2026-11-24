# ÓRBITA ARENA — ETAPA 2B.5.2.6.3
## Correções de Integridade e Resolução de Datas do Cache L1

**Natureza:** Implementação corretiva local e validação automatizada (sem commit, push ou deploy).  
**Escopo alterado:** `src/server/collectors/unifiedGamesCollector.ts`, `src/server/cache/prewarmedGamesCache.ts` e suíte `tests/etapa2b5263.test.ts`.

---

## 1. Contexto e Problemas Identificados na Etapa 2B.5.2.6.2

A auditoria complementar da Etapa 2B.5.2.6.2 (`temp/etapa-2b-5-2-6-2-auditoria-complementar-l1.md`) apontou cinco pontos críticos pendentes antes da publicação do Cache L1 em RAM:

1. **Coleta fria e uso indevido do LRU em `/api/games/central` e `/api/games/shadow` para ontem/amanhã:** `targetIso !== todayBrt` desviava `?date=<ontem>` e `?date=<amanhã>` para coleta fria on-demand (e inseria essas datas no `historicalCache` no Central), mesmo já estando na janela de 3 dias pré-aquecida em `this.activeState`.
2. **Desalinhamento das rotas legadas para `<ontem>`:** `unifiedGamesCollector.ts` não expunha os candidatos coletados de ontem (`enrichmentScope: "overnight_candidates"`) em mapa separado para `/api/football/today?date=<ontem>`, `/api/jogos-texto?date=<ontem>` e `/api/jogos-lite?date=<ontem>`, forçando nova coleta upstream quando os dados já existiam no L1.
3. **Verificação incompleta de integridade no refresh:** `fixturesTruncatedOrFailed` e a proteção de preservação de snapshot na Camada A olhavam apenas falhas/truncamento do Bzzoiro (`hasMaterialBzzPartial`), ignorando falhas de `espn` e `sportsblaze`, e renovando `fixturesCollectedAt` quando uma coleção preservada era reaproveitada.
4. **Lacuna de testes de inicialização fria (`activeState === null`) e grades vazias legítimas:** `tests/etapa2b5261.test.ts` validava apenas falhas após um boot saudável, sem testar cold-boot total falho, cold-boot parcial de cada provedor, grades vazias legítimas (HTTP 200 vazio ou SportsBlaze 404) e recuperação automática.
5. **Atualização do PLL Lacrosse no endpoint `/api/sports/lacrosse`:** deltas da Camada B atualizavam `espnLacrosseDeduped`, mas o cache separado `lacrosseEndpointCache` aguardava TTL de 45s/12min e não refletia imediatamente os placares/estados do tick de 30s em RAM.

---

## 2. Arquivos Modificados e Responsabilidades

| Arquivo | Tipo | Responsabilidades implementadas |
| :--- | :--- | :--- |
| `src/server/collectors/unifiedGamesCollector.ts` | Backend / Coletor | Adição de `bzzOvernightPayloadByDate: Map<string, BzzoiroTodayCollectorPayload>` a `UnifiedCollectionArtifacts`; gravação controlada quando `isYesterdayOvernightWindow && payload && !isUsingPreviousFallback && eventsCompleteForDate`; marcação explícita `bzzEventsCompleteByDate.set(dateStr, false)` em falha/exceção do Bzzoiro. |
| `src/server/cache/prewarmedGamesCache.ts` | Backend / Cache L1 | Janela explícita de 3 dias no Central e Shadow com `yesterdaySnapshotId`/`tomorrowSnapshotId` estáveis; reaproveitamento de `bzzOvernightPayloadByDate` nas rotas legadas de ontem; integridade multiprincipal (`ProviderIntegrityFlags`: Bzzoiro, ESPN, SportsBlaze); preservação granular por provedor e liga sem renovar timestamps preservados; reconciliação de deltas PLL da Camada B no `lacrosseEndpointCache`. |
| `tests/etapa2b5263.test.ts` | Testes automatizados | Cobertura integral dos 20 cenários exigidos pela Etapa 2B.5.2.6.3 em 5 blocos de teste (`node:test` + Express HTTP real). |

> **Preservação de escopo:** Frontend (`src/App.tsx`, `src/components/**`, `src/services/**`), contrato Central 1.0, `brtDate.ts`, `gameSets.ts`, `MatchStatsService`, autenticação/CORS de `/api/internal/cache/refresh`, `/api/sportsblaze/schedule?startDate=...&endDate=...` e `tests/etapa2b5261.test.ts` permaneceram intactos.

---

## 3. Resolução de Datas no Central e Shadow (`yesterdayBrt`, `referenceDateBrt`, `tomorrowBrt` e LRU)

Em `src/server/cache/prewarmedGamesCache.ts`:

- **`ActiveCombinedState` (`:240-264`):** passou a armazenar `yesterdaySnapshotId` e `tomorrowSnapshotId` ao lado de `snapshotId`, garantindo que projeções de ontem e amanhã mantenham `snapshotId` e `ETag` determinísticos entre leituras e ticks no-op (`304 Not Modified`), sem depender de chamadas dinâmicas a `createUnifiedSnapshotId(...)` por requisição.
- **`getCentralSnapshotResolution()` (`:2367-2476`):**
  1. Resolve explicitamente a janela `[yesterdayWindowBrt, todayBrt, tomorrowWindowBrt]`.
  2. Se `!targetIso`, delega a `collectSnapshotFn` (mantendo o erro `400` para data inválida).
  3. Se `!isInsidePrimaryWindow` (qualquer data fora dos 3 dias), roteia **exclusivamente** para `getHistoricalSnapshotResolution()`, que grava somente em `this.historicalCache` (LRU de 5 entradas / TTL 5 min) e nunca altera `this.activeState`.
  4. Dentro da janela de 3 dias:
     - `targetIso === ready.referenceDateBrt`: serve `ready.shadowResult` clonado com `ready.snapshotId` e zero chamadas upstream.
     - `targetIso === yesterdayWindowBrt` ou `tomorrowWindowBrt`: projeta a partir de `ready.artifacts` via `composeUnifiedShadowResultFromArtifacts(...)` com `referenceDateBrt: targetIso`, **sem** mutar `ready.artifacts` nem `ready.shadowResult`, **sem** inserir nada no `historicalCache`, **sem** fabricar D-2 em `overnight` para ontem ou D+2 em `tomorrow` para amanhã, e mantendo o snapshot de `today` com `sets.overnight` preservado intacto.
     - Sinaliza claramente os warnings de escopo (`l1_window_projection_yesterday_overnight_scope` / `l1_window_projection_next_day_outside_window`) e herda estado degradado quando a coleta para a data alvo não estiver completa.
- **`getShadowResult()` (`:2478-2544`):** aplica a mesma resolução da janela de 3 dias `[yesterdayWindowBrt, todayBrt, tomorrowWindowBrt]` em RAM, disparando coleta fria apenas para datas fora dessa janela ou quando não houver snapshot ativo.

---

## 4. Reaproveitamento das Rotas Legadas para Ontem (e Preservação de Range)

- **`unifiedGamesCollector.ts` (`:2657, :2890, :3035, :3482`):** `bzzOvernightPayloadByDate` armazena o payload de ontem coletado pela Camada A sob `enrichmentScope: "overnight_candidates"` quando `eventsCompleteForDate && !isUsingPreviousFallback`.
- **`executeLiveTick()` (`prewarmedGamesCache.ts:2094-2095`):** deltas ao vivo de placar/status/tempo da Camada B também são propagados para `bzzOvernightPayloadByDate`, mantendo partidas overnight em andamento consistentes nas rotas legadas.
- **`getPrewarmedFootballToday()` (`prewarmedGamesCache.ts:2546-2590`):** distingue três estados:
  1. Payload completo (`bzzPayloadByDate` + `bzzEventsCompleteByDate === true`) -> hit em RAM.
  2. Ontem preparado sob `overnight_candidates` (`bzzOvernightPayloadByDate` + `bzzEventsCompleteByDate === true`) -> hit em RAM sem chamada upstream duplicada, preservando `enrichmentScope: "overnight_candidates"` no payload.
  3. Data indisponível ou incompleta (`bzzEventsCompleteByDate !== true`) -> retorna `{ hit: false, payload: null }` para fallback seguro on-demand.
- **`getPrewarmedSportsBlazeGames()` (`:2592-2622`):** já atende consultas de data única (`yesterdayBrt`, `todayBrt`, `tomorrowBrt`) diretamente de `espnAmericanByLeagueDate`; consultas com `startDate`/`endDate` em `/api/sportsblaze/schedule` (`registerCollectorRoutes.ts`) continuam preservadas no coletor dedicado sem impacto.
- **`getPrewarmedJogosTexto()` (`:2624-2677`):** passa a aceitar `yesterdayBrt` quando `bzzEventsCompleteByDate === true`, `bzzOvernightPayloadByDate` (ou `bzzPayloadByDate`), `espnAmericanByLeagueDate` (`boxscores`) e `espnLacrosseByDate` estão completos, beneficiando tanto `/api/jogos-texto` quanto `/api/jogos-lite` sem nova coleta upstream, e retornando `null` (fallback controlado) quando qualquer fonte da data estiver incompleta.

---

## 5. Correções de Integridade no Refresh (Bzzoiro, ESPN, SportsBlaze) e Timestamps

Em `PrewarmedGamesCache.executeFixturesRefresh()` (`prewarmedGamesCache.ts:1287-1820`):

- **Integridade de todos os provedores (`ProviderIntegrityFlags`, `:234-238, :1316-1354`):**
  - `bzzFailed`: verifica `!collectedArtifacts`, `!bzzReport.ok`, `bzzReport.failedCalls > 0`, `!bzzCompleteToday`, `!bzzCompleteYesterday`, `!bzzCompleteTomorrow`, `bzzMaterialPartialCount > 0` ou `bzzStaleFallbackCount > 0`.
  - `espnFailed`: verifica `(espnReport.totalCalls > 0 && !espnReport.ok)` ou `espnReport.failedCalls > 0`.
  - `sbFailed`: verifica `(sbReport.totalCalls > 0 && !sbReport.ok)` ou `sbReport.failedCalls > 0`.
  - `anyProviderMaterialFailure = !collected.result.ok || !collectedArtifacts || bzzFailed || espnFailed || sbFailed`.
- **Preservação granular por provedor e por liga (`:1362-1752`):**
  - Quando existe `sameDateExisting` e `anyProviderMaterialFailure === true`:
    - Se a coleta global falhou (`!collected.result.ok || !collectedArtifacts`), preserva todo o último snapshot válido com `degraded: true`, `fixturesTruncatedOrFailed: true`, `providerIntegrity` falso para todos e os `providerStatus` falhos atualizados.
    - Se houve falha parcial ou total em um ou mais provedores:
      - **Provedores/ligas saudáveis** (ex.: ESPN/SportsBlaze quando só Bzzoiro trunca, ou Bzzoiro quando só uma liga ESPN ou SportsBlaze falha) **recebem seus dados novos normalmente**.
      - **Provedor/data/liga com falha** preserva exclusivamente os dados válidos anteriores do `sameDateExisting` (incluindo `bzzPayloadByDate`, `bzzOvernightPayloadByDate`, `bzzEventsCompleteByDate`, `bzzDeduped`, `espnAmericanByLeague`, `espnAmericanByLeagueDate`, `espnLacrosseDeduped`, `espnLacrosseByDate`, `sbAmericanByLeague` por liga falhada), nunca substituindo uma coleção válida existente por array vazio de erro.
- **Preservação real de timestamps:**
  - Quando uma coleção de provedor é preservada de `sameDateExisting`, `fixturesCollectedAt` / `fixturesCollectedAtMs` mantêm o timestamp da última coleta preservada (`anyPreserved ? sameDateExisting.fixturesCollectedAt : generatedIso`), `providerMeta.bzzoiro.collectorFetchedAt` preserva o `collectorFetchedAt` original (via `preserveBzzoiroFreshnessMeta`) e `providerMeta.sportsblaze.feedUpdatedAt` preserva o timestamp anterior caso o SportsBlaze tenha sido preservado.
- **Limpeza de avisos após recuperação:** quando um refresh subsequente obtém coleta íntegra de todos os provedores (`anyProviderMaterialFailure === false`), substitui o estado degradado por `degraded: false`, `fixturesTruncatedOrFailed: false`, `providerIntegrity: { bzzoiroComplete: true, espnComplete: true, sportsblazeComplete: true }` e limpa avisos antigos de preservação/degradação.

---

## 6. Inicialização Fria Parcial/Falha (`activeState === null`), Grades Vazias Legítimas e Recuperação Automática

1. **Falha total no cold boot (`!collected.result.ok || !collectedArtifacts`, sem `activeState`):**
   - `activeState` permanece `null`, `bootStatus = "failed"`, `bootSource = "none"`, `fixturesFailuresTotal` é incrementado e o servidor HTTP **não** é bloqueado.
   - Requisições concorrentes durante/após o boot fazem coalescing em `ensureActiveTodayState()` (`fixturesInFlight`); a cadência ativa de 30s (`evaluateLiveCadence`) aciona recuperação automática no próximo tick até obter um snapshot e promover `bootStatus` para `"ready"` (ou `"degraded"` se parcial).
2. **Falha parcial no cold boot (`activeState === null`):**
   - Ex.: Bzzoiro truncado na página 2, falha em `/events/live/`, falha total da ESPN ou falha total do SportsBlaze.
   - O snapshot parcial inicial é aceito apenas como fallback operacional, marcado com `bootStatus = "degraded"`, `degraded: true`, `fixturesTruncatedOrFailed: true` e `providerIntegrity` apontando exatamente qual(is) provedor(es) está(ão) incompleto(s).
   - Para datas sem `bzzEventsCompleteByDate === true`, as rotas legadas `/api/football/today` e `/api/jogos-texto` não servem coleta truncada.
   - Na próxima coleta bem-sucedida, todos os jogos recuperados entram e os flags voltam a íntegros.
3. **Grades vazias legítimas:**
   - Respostas HTTP 200 sem partidas (`events: []`, `ok: true`) no Bzzoiro/ESPN ou `404` tratado como grade vazia legítima no SportsBlaze (`ok: true, empty: true, failedCalls: 0`) **não** são tratadas como erro, preservando `degraded: false`, `fixturesTruncatedOrFailed: false` e `providerIntegrity` totalmente `true`.

---

## 7. Overlay PLL Lacrosse na Camada B e `/api/sports/lacrosse`

Em `src/server/cache/prewarmedGamesCache.ts`:

- **`applyPllOverlayToLacrossePayload()` (`:936-1024`) e `syncPllOverlayIntoLacrosseEndpointCache()` (`:2679-2686`):**
  - Sempre que a Camada B (`executeLiveTick`, `:2280`) ou um refresh da Camada A (`:1744, :1812`) atualiza `workingArtifacts.espnLacrosseDeduped`, o cache de `/api/sports/lacrosse` (`lacrosseEndpointCache`) é reconciliado **em RAM com 0 chamadas HTTP extras**, atualizando `status`, `competitions[0].status`, placares e `hasLive` das partidas PLL (sem afetar eventos NLL ou NCAA já presentes).
- **`refreshUnifiedLacrosseEndpoint()` (`:2729`):** após buscar o payload unificado do endpoint (PLL/NLL/NCAA), aplica `applyPllOverlayToLacrossePayload` caso o estado ativo da Camada B contenha dados mais recentes da PLL, evitando regressão de placar se o refresh reativo de 45s/12min ocorrer logo após um tick de 30s.

---

## 8. Cobertura de Testes Implementada e Resultados

Arquivo criado: `tests/etapa2b5263.test.ts` (5 suítes cobrindo todos os 20 cenários especificados):

| Bloco | Cenários cobertos | Resultado |
| :--- | :--- | :--- |
| `2B.5.2.6.3 [1-5]` | `/api/games/central` e `/api/games/shadow` para `ontem`, `hoje` e `amanhã` via L1 (0 chamadas upstream); não contaminação de `sets.overnight` de hoje; nenhuma entrada de ontem/amanhã no `historicalCache`; datas fora da janela isoladas no LRU histórico sem contaminar snapshot principal nem virada BRT. | **PASS** |
| `2B.5.2.6.3 [6-7]` | Rotas legadas de ontem (`/api/football/today?date=<ontem>`, `/api/jogos-texto?date=<ontem>`, `/api/jogos-lite?date=<ontem>`) servidas de `bzzOvernightPayloadByDate` sem coleta duplicada quando completas, e fallback on-demand quando incompletas; preservação de `/api/sportsblaze/schedule?startDate=...&endDate=...`. | **PASS** |
| `2B.5.2.6.3 [8-14]` | Refresh com Bzzoiro truncado preservando apenas Bzzoiro e atualizando ESPN/SportsBlaze; falha parcial/total isolada de ESPN e por liga do SportsBlaze sem zerar coleções válidas nem mascarar timestamps preservados; `providerIntegrity` e `fixturesTruncatedOrFailed` consistentes; recuperação automática limpando degradação e warnings. | **PASS** |
| `2B.5.2.6.3 [15-17]` | Cold boot (`activeState === null`) com falha total (`bootStatus = "failed"`, sem bloqueio HTTP, coalescing e recuperação); cold boot parcial (Bzzoiro página 2, `/events/live/`, ESPN down, SportsBlaze down) com `bootStatus = "degraded"` e recuperação; grades vazias legítimas sem degradação falsa. | **PASS** |
| `2B.5.2.6.3 [18-20]` | Overlay PLL Lacrosse na Camada B atualizando Central, Shadow e `/api/sports/lacrosse` em RAM sem chamadas extras e mantendo NLL/NCAA; preservação de `snapshotId`, `ETag` e `304 Not Modified` em ticks no-op (para hoje, ontem e amanhã). | **PASS** |

---

## 9. Evidências de Validação Técnica

Todas as verificações foram executadas localmente após a última edição:

1. **Suíte específica da Etapa 2B.5.2.6.3:**
   - Comando: `npx tsx --test tests/etapa2b5263.test.ts`
   - Resultado: **5/5 subtestes aprovados (0 falhas)**.
2. **Suíte completa de regressão (todos os arquivos em `tests/*.test.ts`):**
   - Comando: `npx tsx --test tests/*.test.ts`
   - Resultado: **93/93 testes aprovados (0 falhas)** — incluindo os 88 testes pré-existentes (como `tests/etapa2b5261.test.ts`, `tests/etapa2b524.test.ts`, etc.) e os 5 blocos novos.
3. **Verificação de tipos / Lint:**
   - Comando (`lint_applet`): `npm run lint` (`tsc --noEmit`)
   - Resultado: **Sucesso sem erros**.
4. **Build completo da aplicação:**
   - Comando (`compile_applet`): `npm run build`
   - Resultado: **Build compilado com sucesso**.

---

## 10. Conclusão e Recomendação Antes de Commit/Deploy

As correções pendentes da auditoria Etapa 2B.5.2.6.2 foram integralmente implementadas e validadas localmente:

- A janela de 3 dias `[yesterdayBrt, referenceDateBrt, tomorrowBrt]` é servida diretamente do L1 em `/api/games/central`, `/api/games/shadow` e nas rotas legadas aplicáveis, mantendo o LRU histórico restrito a datas fora da janela e sem contaminar o snapshot principal.
- A integridade da Camada A agora cobre Bzzoiro, ESPN e SportsBlaze por provedor e por liga, preserva dados válidos e timestamps reais ante falhas, distingue grades vazias legítimas de erros, e trata corretamente inicializações frias parciais ou falhas com recuperação automática.
- Deltas ao vivo da PLL Lacrosse na Camada B refletem no `/api/sports/lacrosse` em RAM sem coletas extras e `ETag`/`304 Not Modified` permanecem estáveis em ticks no-op.

**Recomendação:** Aprovado tecnicamente no ambiente local para seguir para a etapa autorizada de consolidação de commit/deploy quando solicitado. **Nenhum commit, push ou deploy foi realizado nesta etapa.**
