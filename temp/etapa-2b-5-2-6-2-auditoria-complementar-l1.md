# ÓRBITA ARENA — ETAPA 2B.5.2.6.2
## Auditoria Complementar do Cache Antecipado L1

**Natureza:** Auditoria exclusivamente de leitura (nenhum arquivo de código, teste, configuração ou contrato foi alterado).  
**Escopo auditado:** `PrewarmedGamesCache` (`src/server/cache/prewarmedGamesCache.ts`), coletores (`bzzoiroCollector.ts`, `espnAmericanCollector.ts`, `sportsblazeAmericanCollector.ts`, `espnLacrosseCollector.ts`, `unifiedGamesCollector.ts`), rotas (`src/server/centralGamesApi.ts`, `src/server/registerCollectorRoutes.ts`, `server.ts`) e suíte de testes (`tests/etapa2b5261.test.ts`).

---

## 1. Resolução de Datas e LRU

### 1.1 Comportamento verificado por rota

#### A) `/api/games/central` (`handleCentralGamesRequest` -> `PrewarmedGamesCache.getCentralSnapshotResolution`)
- **Fluxo:** `src/server/registerCollectorRoutes.ts:777-806` delega para `prewarmedCache.getCentralSnapshotResolution(reqOpts)`.
- **Regra implementada em `src/server/cache/prewarmedGamesCache.ts:1749-1767`:**
  ```ts
  if (!targetIso) {
    return this.collectSnapshotFn(options);
  }
  if (targetIso !== todayBrt) {
    return this.getHistoricalSnapshotResolution({ ...options, date: targetIso });
  }
  ```
- **Diagnóstico — `PROBLEMA IDENTIFICADO`:**
  1. **Hoje BRT (`?date` ausente ou `?date=<hoje>`):** atendido 100% pelo L1 principal (`this.activeState` via `ensureActiveTodayState`, `:1768-1799`), contendo internamente os conjuntos `sets.today`, `sets.overnight` e `sets.tomorrow` para o snapshot ancorado em `todayBrt`.
  2. **Ontem e Amanhã BRT explícitos (`?date=<ontem>` ou `?date=<amanha>`):** como `targetIso !== todayBrt`, a condição `:1764` **desvia imediatamente para `getHistoricalSnapshotResolution()` (`:1694-1747`)**, ignorando `this.activeState` mesmo que ontem e amanhã já tenham sido coletados pela Camada A. Isso **provoca coleta fria on-demand** via `this.collectSnapshotFn(...)` e insere `yesterdayBrt` / `tomorrowBrt` no `historicalCache` (LRU com TTL de 5 min e limite de 5 entradas).
  3. *(Nota contratual)* Para o próprio dia `hoje`, o snapshot central já entrega jogos de ontem no conjunto `overnight` e de amanhã em `tomorrow` dentro do payload padrão (sem `?date=`). Porém, uma consulta direta com `?date=<ontem>` ou `?date=<amanha>` falha em reutilizar o L1 e dispara coleta fria.

#### B) `/api/games/shadow` (`PrewarmedGamesCache.getShadowResult`)
- **Fluxo:** `src/server/registerCollectorRoutes.ts:736-759` delega para `prewarmedCache.getShadowResult(...)`.
- **Regra implementada em `src/server/cache/prewarmedGamesCache.ts:1802-1817`:**
  ```ts
  if (!targetIso || targetIso !== todayBrt) {
    const hist = await this.collectSnapshotFn({
      date: options?.date,
      allowStaleFallback: false,
      useColdFailureCooldown: false,
      now: nowFn
    });
    return hist.shadowResult;
  }
  ```
- **Diagnóstico — `PROBLEMA IDENTIFICADO`:**
  - Apenas `todayBrt` (ou sem `date`) usa `this.activeState`.
  - Qualquer consulta com `?date=<ontem>` ou `?date=<amanha>` **dispara coleta fria completa sempre** (nem sequer passa pelo LRU histórico).

#### C) `/api/football/today` (`PrewarmedGamesCache.getPrewarmedFootballToday`)
- **Fluxo:** `src/server/registerCollectorRoutes.ts:431-445` -> `prewarmedCache.getPrewarmedFootballToday(dateParam)` (`prewarmedGamesCache.ts:1836-1865`).
- **Armazenamento na Camada A (`src/server/collectors/unifiedGamesCollector.ts:2964, 3029-3032`):**
  ```ts
  const isYesterdayOvernightWindow = dateStr === yesterdayBrt && dateStr !== referenceDateBrt;
  ...
  bzzEventsCompleteByDate.set(dateStr, eventsCompleteForDate && !isUsingPreviousFallback);
  if (!isYesterdayOvernightWindow && payload && !isUsingPreviousFallback && eventsCompleteForDate) {
    bzzPayloadByDate.set(dateStr, structuredClone(payload));
  }
  ```
- **Diagnóstico:**
  - **Hoje e Amanhã BRT:** `CONFIRMADO` — servidos diretamente da RAM (`bzzPayloadByDate`) sem coleta fria, e com deltas ao vivo propagados em `prewarmedGamesCache.ts:1490-1504`.
  - **Ontem BRT:** `PROBLEMA IDENTIFICADO` — para ontem (`isYesterdayOvernightWindow === true`), a Camada A coleta com `enrichmentScope: "overnight_candidates"` (`unifiedGamesCollector.ts:2972-2973`) e **propositalmente não grava** em `bzzPayloadByDate` (`:3030`). Portanto, `GET /api/football/today?date=<ontem>` recebe `hit: false` e executa `collectBzzoiroFootballToday({ date })` on-demand (`registerCollectorRoutes.ts:440`).

#### D) `/api/sportsblaze/boxscores` e `/api/sportsblaze/schedule` (`PrewarmedGamesCache.getPrewarmedSportsBlazeGames`)
- **Fluxo:** `src/server/registerCollectorRoutes.ts:156-254` -> `prewarmedCache.getPrewarmedSportsBlazeGames(...)` (`prewarmedGamesCache.ts:1867-1897`).
- **Armazenamento na Camada A (`unifiedGamesCollector.ts:3099-3125`):** grava `espnAmericanByLeagueDate` para todas as ligas americanas (`MLB`, `WNBA`, `NFL`, `NCAAF`, `NHL`, `NBA`) nas 3 datas: `yesterdayBrt:boxscores`, `referenceDateBrt:boxscores` e `tomorrowBrt:schedule`. `getPrewarmedSportsBlazeGames` verifica a chave exata e faz fallback entre `:boxscores` e `:schedule` (`prewarmedGamesCache.ts:1881-1889`).
- **Diagnóstico:** `CONFIRMADO` para consultas por data única (`yesterdayBrt`, `todayBrt`, `tomorrowBrt`) sem coleta fria. *(Observação: consultas com `startDate` + `endDate` em `/api/sportsblaze/schedule`, linhas `223-228` de `registerCollectorRoutes.ts`, continuam indo direto para `collectEspnAmericanGames`).*

#### E) `/api/jogos-texto` e `/api/jogos-lite` (`PrewarmedGamesCache.getPrewarmedJogosTexto`)
- **Fluxo:** `src/server/registerCollectorRoutes.ts:449-459` -> `prewarmedCache.getPrewarmedJogosTexto(dateParam)` (`prewarmedGamesCache.ts:1899-1941`).
- **Regra implementada em `prewarmedGamesCache.ts:1909-1913`:**
  ```ts
  const isSupportedDate =
    targetDateStr === ready.referenceDateBrt || targetDateStr === ready.artifacts.tomorrowBrt;
  if (!isSupportedDate) {
    return null;
  }
  ```
- **Diagnóstico:** `CONFIRMADO` para **Hoje e Amanhã BRT**; porém, para **Ontem BRT**, retorna `null` e aciona coleta on-demand nas linhas `465-673` de `registerCollectorRoutes.ts` (o que é coerente com o fato de `bzzPayloadByDate` também não conter ontem).

### 1.2 Isolamento e escopo do LRU histórico (`this.historicalCache`)
- **Isolamento (`CONFIRMADO`):** Em `getHistoricalSnapshotResolution()` (`prewarmedGamesCache.ts:1694-1747`), a escrita ocorre exclusivamente no `Map` `this.historicalCache` (máx. 5 entradas, TTL 5 min, `:60-61`, `:1729-1744`), clonando resultados com `structuredClone`. Nunca substitui nem altera `this.activeState`, `activeState.artifacts` ou `activeState.shadowResult`. Na virada BRT (`:1224`), `this.historicalCache.delete(targetDate)` remove eventual entrada obsoleta da nova data corrente. Validado no `Cenário 10` (`tests/etapa2b5261.test.ts:1107-1156`).
- **Escopo efetivo (`PROBLEMA IDENTIFICADO`):** Devido à condição `targetIso !== todayBrt` em `getCentralSnapshotResolution()` (`:1764`), **o LRU histórico também está sendo usado para `yesterdayBrt` e `tomorrowBrt`**, consumindo slots do LRU em vez de ficar restrito a datas fora da janela principal `[yesterdayBrt, todayBrt, tomorrowBrt]`.

---

## 2. Frequência Efetiva dos Overlays (Camada B)

A Camada B executa em `executeLiveTick()` (`src/server/cache/prewarmedGamesCache.ts:1272-1670`) sob cadência adaptativa de **30s ativo** (`DEFAULT_CACHE_LIVE_ACTIVE_INTERVAL_MS = 30_000`, `:55`) quando há jogo `LIVE` ou dentro da janela `±15 min` do kickoff, ou **120s ocioso** (`DEFAULT_CACHE_LIVE_IDLE_INTERVAL_MS = 120_000`, `:56`).

### 2.1 `espnAmericanCollector`
- **Cache interno:** `ESPN_AMERICAN_CACHE_TTL_MS = 60_000` (60s, `src/server/collectors/espnAmericanCollector.ts:84`), `espnAmericanCache` e mapa `espnAmericanInFlight` (`:619-650`). Só armazena respostas `result.ok` (`:636`).
- **Chamada da Camada B:** `prewarmedGamesCache.ts:1411-1419` passa explicitamente **`bypassCache: true`** (`:1417`) para ligas/datas com jogo `LIVE` ou `±15 min` do kickoff (`:1338-1360`).
- **Efeito:** em `collectEspnAmericanGames` (`espnAmericanCollector.ts:619-630`), `bypassCache: true` pula a verificação de TTL de 60s, **preserva o coalescing por `espnAmericanInFlight` (`:626-630`)** e ainda atualiza `espnAmericanCache` (`:636-640`).
- **Conclusão:** **`CONFIRMADO`** — frequência efetiva acompanha o tick de **30s**, sem bloqueio pelo TTL de 60s.

### 2.2 `sportsblazeAmericanCollector`
- **Cache interno:** `SPORTSBLAZE_BOXSCORES_CACHE_TTL_MS = 60_000` (60s, `src/server/collectors/sportsblazeAmericanCollector.ts:122`), `SPORTSBLAZE_SCHEDULE_CACHE_TTL_MS = 300_000` (5 min, `:123`), `sportsblazeCache` (`:165`) e `sportsblazeInFlight` (`:166`).
- **Chamada da Camada B:** `prewarmedGamesCache.ts:1435-1443` consulta apenas `collectSportsBlazeBoxscoresFn` (nunca `schedule`) com **`bypassCache: true`** (`:1441`) para jogos `LIVE`, `±15 min` do kickoff ou `FINAL` dentro de `RECENT_FINISH_WINDOW_MS = 10 min` (`:1374-1398`).
- **Efeito:** `collectSportsBlazeAmericanGames` (`sportsblazeAmericanCollector.ts:788-819`) ignora o cache de 60s quando `bypassCache: true`, preserva `sportsblazeInFlight` e atualiza o cache interno ao concluir.
- **Conclusão:** **`CONFIRMADO`** no backend/coletor (atualiza no tick de **30s**, limitado apenas pela cadência externa de publicação do próprio CDN `cache.sportsblaze.com`).

### 2.3 `espnLacrosseCollector` e endpoint `/api/sports/lacrosse`
- **No agregador central / Shadow / `jogos-texto` (`espnLacrosseCollector.ts`):**
  - Cache interno de 60s (`ESPN_LACROSSE_CACHE_TTL_MS = 60_000`, `src/server/collectors/espnLacrosseCollector.ts:53`) com `espnLacrosseInFlight` (`:56, 243-247`).
  - A Camada B (`prewarmedGamesCache.ts:1424-1430`) passa **`bypassCache: true`** (`:1428`) quando há jogo PLL `LIVE` ou próximo do kickoff (`:1363-1371`), ignorando o TTL de 60s e mantendo coalescing (`espnLacrosseCollector.ts:236-247`).
- **Ressalva em `/api/sports/lacrosse` (`server.ts:855-862` -> `prewarmedGamesCache.ts:1973-2012`):**
  - Esse endpoint usa `lacrosseEndpointCache` separado com `DEFAULT_LACROSSE_ENDPOINT_LIVE_TTL_MS = 45_000` (45s) e `DEFAULT_LACROSSE_ENDPOINT_IDLE_TTL_MS = 12 * 60_000` (12 min) (`prewarmedGamesCache.ts:62-63, 1998-2008`).
  - `refreshUnifiedLacrosseEndpoint()` só é chamado no boot (`:936`), no timer de fixtures a cada 12 min (`:947`) ou sob demanda quando o TTL de 45s/12m expira em `getUnifiedLacrosseEndpointPayload()` (`:2004-2010`). A Camada B de 30s **não** atualiza `lacrosseEndpointCache`.
  - Ademais, na suíte `tests/etapa2b5261.test.ts`, o `Cenário 4` testa deltas de Bzzoiro, ESPN NBA e SportsBlaze NBA, mas não exercita explicitamente delta de placar do Lacrosse PLL na Camada B.
- **Conclusão:** **`CONFIRMADO`** para o overlay no Central/Shadow/`jogos-texto` (30s com `bypassCache: true`), com a **ressalva** documentada sobre o TTL de 45s (e refresh reativo) do endpoint legado `/api/sports/lacrosse` e lacuna de asserção explícita de delta PLL no `Cenário 4`.

### 2.4 `collectBzzoiroLiveOverlayDelta` (`bzzoiroCollector.ts:3222-3563`)
- **Ausência do cache de 180s:** ao contrário de `collectBzzoiroFootballToday` (`FOOTBALL_CACHE_TTL_MS = 180_000`, `:13`), `collectBzzoiroLiveOverlayDelta` (`:3222`) **não consulta `footballCache`** nem `footballLastCompleteCache`; consulta diretamente `https://sports.bzzoiro.com/api/v2/events/live/?page_size=100` (`:3267`) a cada tick da Camada B.
- **Coalescing:** embora `collectBzzoiroLiveOverlayDelta` não possua `inFlight` próprio, todas as chamadas passam por `PrewarmedGamesCache.triggerLiveTick()` (`prewarmedGamesCache.ts:1257-1270`), protegido por `this.liveInFlight`.
- **Confirmação `LIVE -> FINAL` sem rebuild de grade (`bzzoiroCollector.ts:3366-3432`):** identifica partidas que estavam `AO VIVO` (`knownLiveMatches`) ou agendadas há 5–120 min (`missingLiveIdsToConfirm`, limitado a 5 em `prewarmedGamesCache.ts:1326-1334`) que saíram de `/events/live/`, e confirma o status individualmente em `/api/v2/events/{id}/` (até 25 por tick, somente se `liveOk === true`, `:3388-3390`). Se `>= 10` (`MISSING_LIVE_BULK_REVALIDATE_THRESHOLD`) continuarem não confirmadas após aplicar deltas, agenda revalidação de Layer A em background (`prewarmedGamesCache.ts:1515-1524`).
- **Incidents sob delta (`bzzoiroCollector.ts:3434-3514`):** só consulta `/events/{id}/incidents/` quando `totalGoals > known.totalGoals`, `!known` ou `!known.hasMatchingGoalEvents`, usando cache de 5 min (`BZZOIRO_INCIDENT_CACHE_TTL_MS = 5 * 60_000`, `:16, 3461-3469`) apenas para partidas finalizadas (`isFin`).
- **Preservação de `snapshotId` e `ETag` em no-op:** em `prewarmedGamesCache.ts:1610-1631`, `computeOverlayHash(recomposedShadow)` verifica somente mudança material; se `!changed`, mantém `snapshotId` e `generatedAt` idênticos, preservando `ETag` e `304 Not Modified` (`centralGamesApi.ts:1097-1122`). Validado em `Cenário 4` e `Cenário 5` (`tests/etapa2b5261.test.ts:616-850`).
- **Observação de cache HTTP:** em `centralGamesApi.ts:1201-1213`, respostas saudáveis retornam `Cache-Control: public, max-age=15, must-revalidate` (clientes/CDNs intermediários podem reutilizar resposta por até 15s entre ticks de 30s).
- **Conclusão:** **`CONFIRMADO`**.

---

## 3. Warmup Inicial com Falha Parcial (`activeState === null`)

Inspecionado `PrewarmedGamesCache.executeFixturesRefresh()` (`src/server/cache/prewarmedGamesCache.ts:1072-1237`), `executeUnifiedShadowCollection()` (`src/server/collectors/unifiedGamesCollector.ts:2798-3499`) e `collectBzzoiroFootballToday()` (`src/server/collectors/bzzoiroCollector.ts:1749-2565`).

### 3.1 Comportamento por cenário de falha no boot (sem snapshot anterior)

1. **Falha total de todos os provedores (`!anyProviderSucceeded`):**
   - `unifiedGamesCollector.ts:3419-3464` retorna `result.ok = false` e `artifacts = undefined`.
   - `prewarmedGamesCache.ts:1168-1177` incrementa `fixturesFailuresTotal`, define `bootStatus = "failed"`, `bootSource = "none"`, **não grava `activeState`** (`activeState` permanece `null`) e retorna `{ ok: false, snapshotId: null }`.
   - **Recuperação automática e coalescing:** `start()` não bloqueia o `app.listen` (`server.ts:2442-2445`) e agenda `scheduleNextLiveTick()` no `.finally()` (`prewarmedGamesCache.ts:955-959`). Como `activeState === null`, `evaluateLiveCadence()` escolhe modo `"active"` (30s, `:977-979`), e `executeLiveTick()` (`:1280-1295`) dispara `triggerFixturesRefresh({ reason: "boot", force: true })` após 30s. Requisições concorrentes antes disso passam por `ensureActiveTodayState()` (`:1672-1692`) com coalescing em `this.fixturesInFlight`.

2. **Paginação truncada do Bzzoiro (ex.: página 2 falha) ou falha em `/events/live/` no boot (`activeState === null`):**
   - Em `bzzoiroCollector.ts:2452-2459, 2550-2571`, `collectBzzoiroFootballToday` retorna `ok: true` (HTTP 200) com `complete: false`, `eventsComplete: false`, `truncated: true` (se página >= 2 falhou) ou `liveEndpoint.ok: false` (se `/events/live/` falhou), e **não salva** esse resultado em `footballCache` nem `footballLastCompleteCache` (`:2550`).
   - Em `unifiedGamesCollector.ts:3023-3032, 3057-3060`:
     - `bzzEventsCompleteByDate.set(dateStr, false)`
     - `bzzPayloadByDate` **não** é gravado para a data afetada (`eventsCompleteForDate === false`)
     - `state.materialPartial = true` -> `bzzMaterialPartialCount > 0`
     - `result.ok = true` e `result.degraded = true` (`:2742-2749`).
   - Em `prewarmedGamesCache.ts:1114, 1179-1221`:
     - Como `sameDateExisting` é `null` (sem snapshot anterior), a regra de proteção `:1114` não se aplica e o fluxo cai na aceitação inicial (`:1179-1221`):
       - `this.activeState` **é preenchido** com o snapshot parcial, marcado com `degraded: true` e `fixturesTruncatedOrFailed: true` (`:1200-1202`), e `bootStatus = "degraded"` (`:1217`).
       - **Proteção exposta aos consumidores:** `/api/games/central` e `/api/games/shadow` expõem `degraded: true`, `providerStatus.bzzoiro.partial: true`, `errors` e `Cache-Control: public, max-age=10, must-revalidate` (`centralGamesApi.ts:1209-1210`) — **não** mascara coleta truncada como íntegra.
       - **Rotas legadas (`/api/football/today`, `/api/jogos-texto`, `/api/jogos-lite`):** como `bzzEventsCompleteByDate.get(today) === false` e `bzzPayloadByDate` está vazio para `today`, `getPrewarmedFootballToday` (`:1853-1864`) e `getPrewarmedJogosTexto` (`:1915-1919`) recusam servir L1 truncado e fazem fallback controlado.
       - **Recuperação no próximo refresh:** como `fixturesTruncatedOrFailed` foi salvo como `true`, na próxima coleta saudável (ou se já houver snapshot com `fixturesTruncatedOrFailed: true`), o snapshot parcial será substituído assim que a coleta recuperar.

3. **PROBLEMA IDENTIFICADO — Falha isolada de ESPN ou SportsBlaze no boot (com Bzzoiro completo):**
   - Em `prewarmedGamesCache.ts:1103-1114` e `:1202`:
     ```ts
     const hasMaterialBzzPartial = Boolean(
       !collected.artifacts ||
         !bzzCompleteToday ||
         collected.artifacts.bzzMaterialPartialCount > 0 ||
         collected.artifacts.bzzStaleFallbackCount > 0
     );
     ...
     if ((!collected.result.ok || !collected.artifacts || hasMaterialBzzPartial) && sameDateExisting && !sameDateExisting.fixturesTruncatedOrFailed) { ... }
     ...
     fixturesTruncatedOrFailed: hasMaterialBzzPartial,
     ```
   - `hasMaterialBzzPartial` e `fixturesTruncatedOrFailed` verificam **exclusivamente** a integridade do Bzzoiro, ignorando `espnReport.failedCalls > 0` e `sbReport.failedCalls > 0`.
   - **Consequência prática no boot sem snapshot anterior (`activeState === null`):**
     - Se no boot o Bzzoiro vier completo (`hasMaterialBzzPartial === false`) mas **ESPN ou SportsBlaze falharem 100%** (ex.: 0 jogos americanos carregados), o snapshot é salvo com `degraded: true` (correto em `composeUnifiedShadowResultFromArtifacts:2747-2748`), **porém com `fixturesTruncatedOrFailed: false` (`:1202`)**.
     - Se em um refresh subsequente o Bzzoiro falhar parcialmente, `:1114` enxergará `!sameDateExisting.fixturesTruncatedOrFailed === true` e preservará como se fosse um "snapshot íntegro" uma base que nunca teve os jogos da ESPN/SportsBlaze.
     - Além disso, se em um refresh posterior **somente a ESPN falhar** enquanto o Bzzoiro estiver íntegro, `:1114` não acionará a preservação do snapshot anterior completo porque só testa `hasMaterialBzzPartial`.

4. **PROBLEMA IDENTIFICADO — Lacuna de cobertura de testes automatizados para cold-boot parcial/falho:**
   - Em `tests/etapa2b5261.test.ts`, o `Cenário 6` (`:852-922`) cobre **exclusivamente** a preservação após um boot inicial 100% saudável (`Cenário 6` começa com `await cache.start()` bem-sucedido e só depois injeta truncamento na página 2 ou falha de live tick).
   - **Não existe nenhum teste automatizado** em `tests/etapa2b5261.test.ts` validando o comportamento quando **não há snapshot anterior (`activeState === null`)**:
     - boot com paginação truncada no Bzzoiro;
     - boot com falha em `/events/live/`;
     - boot com falha total de 1 provedor (Bzzoiro total down, ESPN total down ou SportsBlaze total down);
     - boot com falha total de todos os provedores (`bootStatus === "failed"`, `activeState === null`, coalescing e auto-recuperação após 30s).

---

## 4. Matriz de Evidências

| Verificação | Classificação | Evidências principais | Lacunas / Observações |
| :--- | :--- | :--- | :--- |
| **Ontem/hoje/amanhã sem coleta fria** | **PROBLEMA IDENTIFICADO** | **Hoje:** `prewarmedGamesCache.ts:1768-1799` (`/api/games/central`), `:1819-1834` (`/api/games/shadow`), `:1853-1862` (`/api/football/today`), `:1881-1894` (`/api/sportsblaze/*`), `:1909-1940` (`/api/jogos-texto`, `/api/jogos-lite`). **Amanhã:** `bzzPayloadByDate` + `espnAmericanByLeagueDate` (`unifiedGamesCollector.ts:3030-3032, 3099-3125`) atendem rotas legadas sem coleta fria. **Ontem:** `espnAmericanByLeagueDate` atende `/api/sportsblaze/boxscores` e `schedule` por data única (`:3099-3125`).<br>**Testes:** `Cenário 1 & 2` (`tests/etapa2b5261.test.ts:519-565`) e `Cenário 12` (`:1190-1301`). | 1) `GET /api/games/central?date=<ontem\|amanha>` cai em `if (targetIso !== todayBrt)` (`prewarmedGamesCache.ts:1764-1766`) e executa coleta fria em `getHistoricalSnapshotResolution()`.<br>2) `GET /api/games/shadow?date=<ontem\|amanha>` executa `collectSnapshotFn` frio sempre (`:1809-1817`).<br>3) `GET /api/football/today?date=<ontem>` falha em acertar L1 porque `unifiedGamesCollector.ts:3030` exclui `!isYesterdayOvernightWindow` de `bzzPayloadByDate`.<br>4) `/api/jogos-texto` e `/api/jogos-lite` suportam em L1 apenas `today` e `tomorrow` (`:1909-1913`), indo para coleta on-demand em `yesterday`.<br>5) Consultas com range `startDate/endDate` em `/api/sportsblaze/schedule` (`registerCollectorRoutes.ts:223-228`) não usam L1. |
| **LRU restrito a datas históricas** | **PROBLEMA IDENTIFICADO** | **Isolamento intacto:** `getHistoricalSnapshotResolution()` (`prewarmedGamesCache.ts:1694-1747`) grava apenas em `this.historicalCache` (`maxHistoricalCacheEntries = 5`, `TTL = 5 min`), clona objetos, não muta `this.activeState` e limpa a nova data corrente na virada BRT (`:1224`).<br>**Teste:** `Cenário 10` (`tests/etapa2b5261.test.ts:1107-1156`). | Por causa de `if (targetIso !== todayBrt)` (`prewarmedGamesCache.ts:1764-1766`), consultas diretas a `/api/games/central?date=<ontem>` ou `?date=<amanha>` entram no `historicalCache` LRU em vez de serem restritas a datas fora da janela `[ontem, hoje, amanhã]`. |
| **Overlay americano efetivo** | **CONFIRMADO** | `PrewarmedGamesCache.executeLiveTick()` (`prewarmedGamesCache.ts:1338-1360, 1374-1444`) chama `collectEspnAmericanFn` (`:1417`) e `collectSportsBlazeBoxscoresFn` (`:1441`) com `bypassCache: true`, ignorando os caches de 60s em `espnAmericanCollector.ts:84, 619` e `sportsblazeAmericanCollector.ts:122, 788`, mantendo coalescing (`espnAmericanInFlight`, `sportsblazeInFlight`) e sem chamar `schedule`.<br>**Teste:** `Cenário 4` (`tests/etapa2b5261.test.ts:616-733`). | A efetividade em produção depende também da cadência upstream de atualização do feed público `cache.sportsblaze.com`; estatísticas pós-jogo SportsBlaze cessam após `RECENT_FINISH_WINDOW_MS = 10 min` (`prewarmedGamesCache.ts:65, 1382`). |
| **Overlay lacrosse efetivo** | **CONFIRMADO** | Na Camada B (`prewarmedGamesCache.ts:1362-1371, 1423-1432`), `collectEspnLacrosseFn` é chamado com **`bypassCache: true`** (`:1428`), ignorando `ESPN_LACROSSE_CACHE_TTL_MS = 60_000` (`espnLacrosseCollector.ts:53, 236-247`) e preservando `espnLacrosseInFlight`, atualizando `espnLacrosseByDate` e `espnLacrosseDeduped` (`prewarmedGamesCache.ts:1555-1567`). | **Ressalvas:** (1) A rota legada `/api/sports/lacrosse` (`server.ts:855-862`) lê `lacrosseEndpointCache` (`prewarmedGamesCache.ts:1973-2012`), cujo TTL ao vivo é **45s** (reativo no request + 12 min no timer de fixtures, sem atualização no tick de 30s); (2) `Cenário 4` não testa explicitamente delta de placar PLL na Camada B. |
| **Bzzoiro delta efetivo** | **CONFIRMADO** | `collectBzzoiroLiveOverlayDelta()` (`bzzoiroCollector.ts:3222-3563`) não usa o cache de 180s de `footballCache`; consulta `/events/live/` diretamente sob coalescing `this.liveInFlight` (`prewarmedGamesCache.ts:1257-1270`), confirma saída `LIVE -> FINAL` via `/events/{id}/` sem rebuild de fixtures (`:3366-3432`), consulta `/incidents/` apenas sob delta de gols (`:3445-3514`, cache de 5 min somente `FINAL`), propaga para `bzzPayloadByDate` (`prewarmedGamesCache.ts:1490-1504`) e preserva `snapshotId`/`ETag`/`304` em no-op (`:1623-1631`).<br>**Testes:** `Cenário 4`, `Cenário 5` e `Cenário 12` (`tests/etapa2b5261.test.ts:616-850, 1255-1291`). | `Cache-Control: public, max-age=15, must-revalidate` (`centralGamesApi.ts:1212`) permite reuso por até 15s em caches HTTP intermediários. |
| **Warmup parcial sem snapshot anterior** | **PROBLEMA IDENTIFICADO** | **Falha total:** `prewarmedGamesCache.ts:1168-1177` mantém `activeState = null`, `bootStatus = "failed"`, não bloqueia HTTP e tenta recuperar em 30s (`:955-979, 1280-1295`) ou via `ensureActiveTodayState` (`:1672-1692`).<br>**Parcial Bzzoiro / 1 provedor:** `:1179-1221` aceita snapshot inicial com `bootStatus = "degraded"`, `degraded: true`, `fixturesTruncatedOrFailed: hasMaterialBzzPartial`, e impede `/api/football/today` e `/api/jogos-texto` de servir L1 truncado via `bzzEventsCompleteByDate = false` (`unifiedGamesCollector.ts:3029-3032`; `prewarmedGamesCache.ts:1853-1864, 1915-1919`). | 1) **Bug lógico em `fixturesTruncatedOrFailed` (`prewarmedGamesCache.ts:1103-1114, 1202`):** avalia apenas `hasMaterialBzzPartial`, não considerando `espnReport.failedCalls > 0` nem `sbReport.failedCalls > 0`. Se no cold boot o Bzzoiro vier íntegro mas ESPN ou SportsBlaze falharem 100%, o estado fica com `fixturesTruncatedOrFailed: false` e passa a ser tratado como snapshot íntegro pela regra `:1114`.<br>2) **Lacuna de testes:** `tests/etapa2b5261.test.ts` não possui nenhum cenário de cold-boot parcial/truncado/falho (`activeState === null`). |
| **Preservação do snapshot íntegro** | **CONFIRMADO** | Com `sameDateExisting && !sameDateExisting.fixturesTruncatedOrFailed`, falhas ou truncamentos da Camada A preservam integralmente `sameDateExisting.artifacts` e os jogos válidos, expondo `degraded: true`, `providerStatus` atualizado e warning específico (`prewarmedGamesCache.ts:1114-1166`); falhas na Camada B não zeram placares nem partidas (`:1470-1481, 1529-1583`).<br>**Teste:** `Cenário 6` (`tests/etapa2b5261.test.ts:852-922`). | **Ressalva:** a guarda `:1114` só é acionada por `!collected.result.ok || !collected.artifacts || hasMaterialBzzPartial`. Falhas parciais/totais isoladas de ESPN ou SportsBlaze (quando o Bzzoiro retorna completo) substituem o `activeState` com `degraded: true` e contagens americanas reduzidas/zeradas em vez de preservar o snapshot americano anterior. |

---

## 5. Conclusão e Pontos de Atenção Antes de Commit/Deploy

A arquitetura de duas camadas (Layer A 12m + Layer B 30s/120s), o coalescing (`fixturesInFlight` / `liveInFlight`), o delta Bzzoiro, o bypass de caches internos nos overlays e a preservação de snapshot ante truncamento do Bzzoiro estão bem estruturados. Entretanto, antes de autorizar commit e deploy, foram identificados **3 desvios concretos** e **1 lacuna de testes**:

1. **Coleta fria para `?date=<ontem>` e `?date=<amanha>` no Central/Shadow (e `<ontem>` no `/api/football/today` e `/api/jogos-texto`):** `getCentralSnapshotResolution()` (`:1764`) e `getShadowResult()` (`:1809`) tratam apenas `targetIso === todayBrt` como L1 principal, enviando ontem/amanhã para coleta fria (e ocupando o LRU histórico no Central). Além disso, `unifiedGamesCollector.ts:3030` não popula `bzzPayloadByDate` para ontem.
2. **Controle incompleto de `fixturesTruncatedOrFailed` para ESPN/SportsBlaze (`prewarmedGamesCache.ts:1103-1114, 1202`):** falhas parciais ou totais de ESPN/SportsBlaze não ativam `fixturesTruncatedOrFailed` nem a proteção de preservação de snapshot íntegro quando o Bzzoiro está completo.
3. **Ausência de testes de falha parcial/total no boot (`activeState === null`) em `tests/etapa2b5261.test.ts`**, além de ausência de asserção específica de delta do Lacrosse PLL na Camada B e diferença de cadência (45s reativo vs 30s) no endpoint `/api/sports/lacrosse`.
