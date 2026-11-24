# ÓRBITA ARENA — ETAPA 2B.5.2.6.4
## Correção do Contrato Bzzoiro Live e Isolamento da Integridade do Overlay

**Natureza:** Correção local do contrato `/api/v2/events/live/`, normalização de campos dinâmicos e isolamento de integridade entre Camada A (`/events/`) e Camada B (`/events/live/`) — **sem commit, push ou deploy**.  
**Escopo alterado:** `src/server/collectors/bzzoiroCollector.ts`, `src/server/collectors/unifiedGamesCollector.ts`, `src/server/cache/prewarmedGamesCache.ts`, `src/server/centralGamesApi.ts` e suíte `tests/etapa2b5264.test.ts`.

---

## 1. Causa Raiz Confirmada

A auditoria direta contra os endpoints de produção da Bzzoiro API v2 revelou a causa raiz das falhas recorrentes no overlay ao vivo e da falsa degradação da grade de futebol:

1. **Divergência de envelope em `/api/v2/events/live/`:** O coletor (`bzzoiroCollector.ts`) exigia `results` (ou array raiz) tanto na coleta de fixtures (`/events/`) quanto na coleta de live overlay (`collectBzzoiroFootballToday` e `collectBzzoiroLiveOverlayDelta`), rejeitando como `invalid_payload` o payload oficial de `/api/v2/events/live/`, cujo formato real é `{ "count": N, "events": [...] }`.
2. **Acoplamento indevido de integridade entre Camada A e Camada B:** Falhas no endpoint `/events/live/` contaminavam `bzzoiroComplete` / `fixturesTruncatedOrFailed` em `PrewarmedGamesCache` e `unifiedGamesCollector`, gerando falso alarme de truncamento da grade (`/events/`), bloqueando o avanço de `fixturesCollectedAt` quando a grade do dia estava íntegra e acionando preservação desnecessária do snapshot completo ou erro espúrio no ciclo de 30s.
3. **Riscos de normalização e sobrescrita:** Campos reais de `/events/live/` (`period`, `home_score_ht`, `away_score_ht`, `extra_time_score`, `penalty_shootout`, `last_updated`) precisavam de preservação segura contra `null`/string vazia e isolamento de `last_updated` para nunca contaminar horários de término (`endDate`/`endTime`/`finishedAt`/`completedAt`) nem invalidar `ETag` em ticks no-op.

---

## 2. Contrato Real de `/api/v2/events/` vs `/api/v2/events/live/`

| Aspecto | `GET /api/v2/events/` (Calendário / Camada A) | `GET /api/v2/events/live/` (Ao Vivo / Camada B) |
| :--- | :--- | :--- |
| **Envelope JSON oficial** | `{ count, next, previous, results: [...] }` | `{ count, events: [...] }` |
| **Paginação** | Paginado via `next` / `page` | Resposta única não paginada |
| **Estado vazio válido** | `{ count: 0, next: null, previous: null, results: [] }` | `{ count: 0, events: [] }` |
| **Parser aplicado** | `parseBzzoiroPaginatedEventsEnvelope()` (rejeita `{ events: [...] }`) | `parseBzzoiroLiveEventsEnvelope()` (aceita `{ count, events: [...] }` e preserva compatibilidade controlada com mocks legados `{ results: [...] }` / array) |
| **Validação de elementos** | Exige array válido e objetos de evento reconhecíveis (`id` numérico/string + identificadores de evento) | Exige lista array válida, `count` numérico não negativo quando presente e validação estrutural de cada item (rejeita `{ count: 0 }` sem lista, `events` não-array ou itens malformados como `invalid_payload`) |

Campos reais tratados em eventos Bzzoiro: `id`, `league_id`, `league_name`, `home_team_id`, `home_team`, `away_team_id`, `away_team`, `event_date` (UTC com sufixo `Z`), `status`, `period`, `current_minute`, `home_score`, `away_score`, `home_score_ht`, `away_score_ht`, `penalty_shootout`, `extra_time_score` e `last_updated`.

---

## 3. Alterações no Parser Bzzoiro e Normalização de Campos

Em `src/server/collectors/bzzoiroCollector.ts` e `src/server/centralGamesApi.ts`:

- **Parsers dedicados (`bzzoiroCollector.ts`):**
  - `parseBzzoiroLiveEventsEnvelope()` utilizado tanto no merge de `collectBzzoiroFootballToday()` quanto em `collectBzzoiroLiveOverlayDelta()`.
  - `parseBzzoiroPaginatedEventsEnvelope()` utilizado em `fetchBzzoiroEventsForLeague()` e na paginação de `collectBzzoiroFootballToday()`.
- **Normalização de campos dinâmicos (`normalizeBzzPeriod`, `normalizeBzzHalftimeScore`, `normalizeBzzOptionalScoreDetail`, `normalizeBzzLastUpdated`):**
  - `period` numérico normalizado junto a `rawPeriod` textual (ex.: `1`, `2`, `"EXTRA_TIME"` -> `period: 3`, `"PENALTIES"`).
  - `tempo` trata intervalo (`HT`/`HALFTIME`) como `"INT"` quando `current_minute` está ausente/0 e preserva minutos válidos (`"68'"`, `"105'"`).
  - `halftimeScore` (`{ home, away }`), `extraTimeScore` e `penaltyShootout` preservados e propagados em `BzzoiroFootballMatchItem`, `UnifiedLiveOverlay` (`unifiedGamesCollector.ts`) e `CentralLiveOverlay` (`centralGamesApi.ts`, omitidos quando `SCHEDULED`/`PP`).
  - `lastUpdated` normalizado em ISO UTC (`eventUpdatedAt` apenas para futebol em `/api/games/central`; nunca copiado para `fixture.endTime.endDate`, `endTime`, `finishedAt` ou `completedAt`).
- **Bloqueio de sobrescrita por `null`/vazio:** `mergeBzzRawEventWithLiveOverlay()` e `applyBzzoiroDeltaToMatchItem()` (`prewarmedGamesCache.ts`) nunca sobrescrevem nomes de times/ligas, datas de fixture, placares válidos, `tempo`, `period`, `halftimeScore`, `extraTimeScore` ou `penaltyShootout` existentes com `null` ou string vazia vindos de um payload parcial de `/events/live/`.

---

## 4. Separação de Integridade entre `/events/` e `/events/live/`

Em `src/server/collectors/unifiedGamesCollector.ts` e `src/server/cache/prewarmedGamesCache.ts`:

- **`UnifiedCollectionArtifacts` (`unifiedGamesCollector.ts`):**
  - `bzzEventsCompleteByDate` e `isFixturesTruncatedOrFailed` refletem **exclusivamente** a integridade do endpoint de calendário `/events/` (`eventsComplete` de `yesterday`, `today` e `tomorrow`).
  - Falhas apenas em `/events/live/` são contabilizadas em `bzzLiveFailedCount` / `liveFailed`, emitindo aviso específico `"live_overlay_bzzoiro_unavailable"` sem rebaixar `eventsComplete` e sem adicionar `"provider_bzzoiro_partial"` enganoso (regra também alinhada em `centralGamesApi.ts`).
- **Flag independente em `PrewarmedGamesCache`:**
  - `PrewarmedCacheObservability.activeSnapshot` agora expõe `liveOverlayDegraded: boolean` ao lado de `fixturesTruncatedOrFailed: boolean` e `providerIntegrity` (`{ bzzoiroComplete, espnComplete, sportsblazeComplete }`).
  - `bzzoiroComplete` e `fixturesTruncatedOrFailed` representam a completude da grade de calendário (`/events/`); `liveOverlayDegraded` representa exclusivamente a saúde da camada de overlay ao vivo (`/events/live/`).

---

## 5. Comportamento nas 5 Combinações (Matriz de Integridade)

| Cenário (`/events/` × `/events/live/`) | `bzzoiroComplete` | `fixturesTruncatedOrFailed` | `liveOverlayDegraded` | `degraded` | Comportamento do Cache L1 |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **1. `/events/` completo + `/events/live/` `{ count: 0, events: [] }`** | `true` | `false` | `false` | `false` | Boot/refresh saudável sem jogos ao vivo; zero erros ou warnings. |
| **2. `/events/` completo + `/events/live/` `{ count: N, events: [...] }`** | `true` | `false` | `false` | `false` | Grade e placares/tempos/períodos ao vivo atualizados em todas as rotas. |
| **3. `/events/` completo + `/events/live/` falhando** | `true` | `false` | `true` | `true` | Nova grade de `/events/` e novo `fixturesCollectedAt` são aceitos; último overlay ao vivo válido anterior é preservado via `preservePriorBzzOverlayAcrossArtifacts` e warning `live_overlay_bzzoiro_unavailable` é emitido. |
| **4. `/events/` truncado/falho + `/events/live/` funcionando** | `false` | `true` | `false` | `true` | Grade válida anterior e `fixturesCollectedAt` original são preservados; deltas ao vivo de `/events/live/` continuam sendo aplicados nas partidas conhecidas via `applyFreshBzzLiveOntoPreservedArtifacts`. |
| **5. `/events/` falho + `/events/live/` falho** | `false` | `true` | `true` | `true` | Preserva último snapshot válido completo e último overlay sem renovar `fixturesCollectedAt`, `collectorFetchedAt` ou `liveUpdatedAt`. |

---

## 6. Ciclo de Atualização ao Vivo de 30s (`UPCOMING` -> `LIVE` -> `FINAL`)

- **Falha isolada no tick de 30s (`executeLiveTick`):** Quando `collectBzzoiroLiveOverlayDelta()` falha durante a cadência de 30s, o snapshot preserva placares/estados válidos anteriores, mantém `liveUpdatedAt` inalterado, marca `liveOverlayDegraded = true` + `degraded = true` com warning `"live_overlay_bzzoiro_unavailable"` e **não** altera `fixturesTruncatedOrFailed` nem `providerIntegrity.bzzoiroComplete`.
- **Recuperação automática no próximo tick:** Quando `/events/live/` volta a responder HTTP 200 `{ count, events }`, `executeLiveTick` aplica os deltas novos, zera `bzzLiveFailedCount`, remove erros `live:*` de `providerStatus.bzzoiro.errors`, remove `"live_overlay_bzzoiro_unavailable"` e limpa `liveOverlayDegraded`/`degraded` (caso não haja outra falha de fixture).
- **Ciclo completo `UPCOMING` -> `LIVE` -> `FINAL`:** Partida inicia agendada em `/events/`, entra ao vivo via `/events/live/` (`IN_PLAY` -> `"LIVE"`, placar, minuto, `halftimeScore`), e ao sair de `/events/live/` é confirmada via `/events/:id/` (`FINISHED` -> `"FINAL"` com `extraTimeScore`/`penaltyShootout` preservados).
- **Estabilidade de `ETag` / `304 Not Modified`:** `computeOverlayHash()` exclui metadados voláteis como `liveOverlay.updated` (`last_updated`). Ticks em que apenas `last_updated` avança sem mudança real de placar, status, tempo, período ou incidentes resultam em `changed: false` (no-op) e preservam `ETag` e respostas `304`.

---

## 7. Impacto e Consistência nas Rotas (`/api/*`)

| Rota | Impacto da Etapa 2B.5.2.6.4 |
| :--- | :--- |
| **`/api/games/central`** | Expõe `liveOverlay.rawPeriod`, `halftimeScore`, `extraTimeScore`, `penaltyShootout` (omitidos em `SCHEDULED`/`PP`) e `eventUpdatedAt` apenas para futebol; preserva `ETag`/`304` em no-ops; emite `live_overlay_bzzoiro_unavailable` isolado de `provider_bzzoiro_partial`. |
| **`/api/games/shadow`** | Recebe imediatamente os overlays normatizados de `/events/live/` e reflete o estado íntegro da grade separado do overlay ao vivo. |
| **`/api/football/today`** | Lido da camada L1 já mesclada e validada com proteção contra sobrescrita nula; não bloqueia grade íntegra quando apenas `/events/live/` falha. |
| **`/api/jogos-texto` & `/api/jogos-lite`** | Consomem exatamente os placares, status (`"AO VIVO"`/`"ENCERRADO"`) e tempos (`"68'"`, `"INT"`) atualizados pelos ciclos da Camada A e Camada B em RAM. |
| **`/api/health`** | Mantém formato passivo sem disparar coletas e adiciona `cache.liveOverlayDegraded` para distinguir saúde das fixtures (`fixturesTruncatedOrFailed`) da saúde do overlay live. |

---

## 8. Cobertura de Testes Implementada (`tests/etapa2b5264.test.ts` e Suíte Completa)

Arquivo criado: `tests/etapa2b5264.test.ts` (12 blocos de teste cobrindo todos os 18 cenários especificados):

| Bloco | Cenários cobertos | Resultado |
| :--- | :--- | :---: |
| `2B.5.2.6.4 [1]` | `/events/live/` retornando `{ count, events: [...] }` aceito como `liveEndpoint.ok=true` e `liveDelta.ok=true` com placar e tempo atualizados. | **PASS** |
| `2B.5.2.6.4 [2]` | `/events/live/` retornando `{ count: 0, events: [] }` tratado como vazio válido sem `invalid_payload` nem `degraded`. | **PASS** |
| `2B.5.2.6.4 [3 & 4]` | `/events/live/` sem lista `events`/`results`, com `count` inválido ou itens malformados rejeitado como `invalid_payload`. | **PASS** |
| `2B.5.2.6.4 [5 & 6]` | `/events/` exige `results` (rejeita envelope `{ events }`) e `/events/live/` mantém compatibilidade com mocks legados `{ results }`. | **PASS** |
| `2B.5.2.6.4 [7 & 8]` | Boot saudável com `{ count: 0, events: [] }` e com `{ count: N, events }` propagando para Central, Shadow, `/api/football/today` e `/api/jogos-texto`. | **PASS** |
| `2B.5.2.6.4 [9]` | `/events/` completo + `/events/live/` falhando mantém `bzzoiroComplete=true`, `fixturesTruncatedOrFailed=false`, atualiza `fixturesCollectedAt` e preserva último overlay. | **PASS** |
| `2B.5.2.6.4 [10]` | `/events/` truncado/falho + `/events/live/` funcionando mantém `bzzoiroComplete=false`, `liveOverlayDegraded=false`, preserva fixtures e aplica deltas live. | **PASS** |
| `2B.5.2.6.4 [11]` | `/events/` falho + `/events/live/` falho preserva último snapshot válido e último overlay sem renovar `fixturesCollectedAt` nem `collectorFetchedAt`. | **PASS** |
| `2B.5.2.6.4 [12 & 13]` | Ciclo de 30s preserva overlay quando `/events/live/` falha e recupera automaticamente quando `/events/live/` volta a HTTP 200 `{ count, events }`. | **PASS** |
| `2B.5.2.6.4 [14 & 15]` | Ciclo completo `UPCOMING` -> `LIVE` -> `FINAL` (saída de `/events/live/` + confirmação `/events/:id/`) refletido em todas as rotas. | **PASS** |
| `2B.5.2.6.4 [16 & 17]` | Campos dinâmicos (`period`, `rawPeriod`, `halftimeScore`, `extraTimeScore`, `penaltyShootout`) preservados e sobrescrita por `null`/vazio bloqueada. | **PASS** |
| `2B.5.2.6.4 [18]` | `last_updated` exposto somente como `eventUpdatedAt` (nunca `endDate`/`endTime`/`finishedAt`/`completedAt`) com estabilidade de `ETag`/`304` em no-op. | **PASS** |

**Resultado da suíte completa de regressão (`npx tsx --test tests/*.test.ts`):** **105/105 testes aprovados (0 falhas)**.

---

## 9. Validação Técnica (`tsc --noEmit`, Build e Lint)

1. **Suíte completa de testes:**
   - Comando: `npx tsx --test tests/*.test.ts`
   - Resultado: **105/105 aprovados, 0 falhas** (duração ~11,0s).
2. **Verificação de tipos e lint (`lint_applet`):**
   - Comando: `npm run lint` (`tsc --noEmit`)
   - Resultado: **Sucesso sem erros**.
3. **Build completo (`compile_applet`):**
   - Comando: `npm run build`
   - Resultado: **Build compilado com sucesso**.

> **Status de entrega:** Correções e validações concluídas localmente. **Nenhum commit, push ou deploy foi executado.**
