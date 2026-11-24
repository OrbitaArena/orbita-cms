# ÓRBITA ARENA — ETAPA 2B.5.2.6.1
## Relatório de Implementação: Cache Antecipado em RAM (L1) — Fase 1

**Data:** 2026-11-23  
**Escopo autorizado:** Fase 1 exclusivamente (Cache Antecipado em RAM L1, sem persistência Firestore L2, sem alterações de frontend e sem commit/push/deploy).

---

## 1. Resumo Executivo

Foi implementado o orquestrador de Cache Antecipado em RAM (`L1`) para eliminar a coleta fria (~30s em produção) no primeiro acesso de usuários ao `/api/games/central`, `/api/games/shadow` e às rotas legadas consumidas atualmente pelo frontend em produção.

A solução mantém o servidor respondendo imediatamente no boot (warmup assíncrono não bloqueante logo após o bind da porta `3000`), sustenta duas cadências independentes de atualização em memória (**Camada A — Grade/Fixtures** e **Camada B — Overlay Ao Vivo Adaptativo**) e realiza troca atômica de snapshot combinado (**Camada C**), preservando integralmente o contrato Central `1.0` (`ETag`/`304`, `stats.matchStatsId`, códigos HTTP e headers), o Shadow e todas as rotas legadas.

---

## 2. Arquitetura Implementada (`PrewarmedGamesCache`)

Arquivo principal: `src/server/cache/prewarmedGamesCache.ts`

### 2.1 Camada A — Preparação de Fixtures / Grade
- **Intervalo configurável:** `CACHE_FIXTURES_INTERVAL_MS` (padrão `720000` ms / `12 minutos`).
- **Coleta base unificada:** executa `collectUnifiedCollectionArtifacts` cobrindo ontem/hoje/amanhã BRT (Bzzoiro, ESPN American, ESPN Lacrosse e SportsBlaze), preservando deduplicação, reconciliação americana e `stats.matchStatsId`.
- **Virada de data BRT (`00:00 America/Sao_Paulo`):** detectada automaticamente tanto no agendador quanto no atendimento de requisições, acionando `triggerFixturesRefresh({ reason: "brt_rollover", force: true })` e descartando o dia anterior como data primária.
- **Integridade / Anti-truncamento:** se uma atualização de fixtures falhar ou retornar truncada (ex.: falha na página 2 do Bzzoiro), o snapshot íntegro existente da mesma data BRT **não é substituído por dados parciais**; os metadados de degradação (`degraded: true`, `errors`, `warnings`, `providerMeta`) são atualizados de forma transparente.
- **Lock independente (`fixturesInFlight`):** impede sobreposição de coletas pesadas e realiza coalescing automático entre boot warmup, timer periódico, cronjob autenticado e requisições simultâneas.

### 2.2 Camada B — Overlay Ao Vivo Incremental e Adaptativo
- **Intervalo adaptativo:**
  - **Modo ativo (`active`):** `CACHE_LIVE_ACTIVE_INTERVAL_MS` (padrão `30000` ms / `30s`) quando há partidas `AO VIVO`/`LIVE` ou dentro da janela `±15 minutos` do horário de início.
  - **Modo ocioso (`idle`):** `CACHE_LIVE_IDLE_INTERVAL_MS` (padrão `120000` ms / `120s`) quando `liveCount === 0` e nenhuma partida está próxima do kickoff.
- **Delta Bzzoiro (`collectBzzoiroLiveOverlayDelta` em `src/server/collectors/bzzoiroCollector.ts`):**
  - Consulta leve em `/events/live/` + confirmação pontual limitada de saídas (`LIVE -> FINAL`) e buscas de `/incidents/` restritas a delta de gols / partidas recém-saídas (evitando N+1 em jogos já finalizados ou com eventos sincronizados).
  - Caso >= 10 partidas saiam simultaneamente de `live` sem confirmação (`MISSING_LIVE_BULK_REVALIDATE_THRESHOLD`), agenda revalidação de fixtures em background.
- **Overlay Americano & Lacrosse:** atualiza apenas ligas/datas com partidas ativas ou próximas do kickoff (e boxscores SportsBlaze somente quando há partida ao vivo, próxima do início ou encerrada há menos de 10 minutos).
- **Preservação de `ETag` / `304 Not Modified`:** quando `computeOverlayHash` e flags de degradação permanecem inalterados após um tick live (*no-op*), `snapshotId` e `generatedAt` são preservados sem troca desnecessária de ETag.

### 2.3 Camada C — Snapshot Combinado Atômico & LRU Histórico
- Troca atômica da referência `activeState` em memória (`fixturesHash`, `overlayHash`, `artifacts`, `shadowResult`, `providerMeta`, `errors`, `warnings`).
- Consultas a datas históricas (`date !== hoje BRT`) utilizam cache LRU isolado (`maxHistoricalCacheEntries = 5`, TTL `180s`), sem poluir o snapshot principal do dia corrente.

---

## 3. Compatibilidade com Rotas Legadas do Frontend Atual

O frontend (`src/App.tsx`, `src/components/*`, `src/services/*`) **não foi modificado**. Todas as rotas utilizadas pelo fluxo atual passaram a consultar o `PrewarmedGamesCache` em RAM antes de recorrer a coletas diretas:

| Rota consumida pelo frontend | Integração L1 aplicada | Comportamento em RAM |
|---|---|---|
| `GET /api/football/today` | `prewarmedCache.getPrewarmedFootballToday(date)` | Lê payload completo do dia em RAM sem disparar Bzzoiro |
| `GET /api/sportsblaze/boxscores` | `prewarmedCache.getPrewarmedSportsBlazeGames(..., "boxscores")` | Lê jogos americanos já coletados e atualizados via overlay |
| `GET /api/sportsblaze/schedule` | `prewarmedCache.getPrewarmedSportsBlazeGames(..., "schedule")` | Lê agenda do dia em RAM para consulta de data única |
| `GET /api/jogos-texto` e `GET /api/jogos-lite` | `prewarmedCache.getPrewarmedJogosTexto(date)` | Projeção determinística (`buildJogosTextoResponseFromRaw`) direta dos artefatos L1 |
| `GET /api/sports/lacrosse` | `prewarmedCache.getUnifiedLacrosseEndpointPayload()` | Cache dedicado em RAM com coalescing e TTL adaptativo (live/idle) |
| `GET/HEAD /api/games/central` e `GET /api/games/shadow` | `getCentralSnapshotResolution` / `getShadowResult` | Leituras 100% em RAM com contrato `1.0`, `ETag`/`304` e `matchStatsId` preservados |

---

## 4. Segurança, Endpoint Interno e Graceful Shutdown

- **`POST /api/internal/cache/refresh` (`src/server/registerCollectorRoutes.ts`):**
  - Suporta modos `auto`, `live` e `fixtures` (`400` para modo inválido; `405` com `Allow: POST` para outros métodos).
  - Autenticação via `Authorization: Bearer <secret>` (ou `x-cache-refresh-secret`) validada com `SHA-256` + `crypto.timingSafeEqual` contra `CACHE_REFRESH_SECRET` (env exclusivamente server-side, nunca prefixada com `VITE_*`).
  - *Fail-closed*: retorna `503` se `CACHE_REFRESH_SECRET` não estiver configurada, `401` se credencial ausente/malformada, `403` se inválida.
  - Middleware CORS em `server.ts` bloqueia explicitamente wildcard (`Access-Control-Allow-Origin: *`) em `/api/internal/*` (`OPTIONS` retorna `204` sem headers permissivos).
  - Aplica coalescing com ciclos `inFlight` e cooldown configurável (`CACHE_REFRESH_COOLDOWN_MS`, padrão `15000` ms).
- **`GET /api/health` (`server.ts`):**
  - 100% passivo e somente leitura (nunca dispara coleta externa).
  - Preserva os campos existentes (`status`, `uptime`, `timestamp`, `service`, `environment`) e adiciona `cache` sanitizado (`warmupStatus`, `warmupSource`, `referenceDateBrt`, `snapshotId`, `snapshotAgeMs`, `degraded`, `counts`, `liveMode`, `liveIntervalMs`, `fixturesInFlight`, `liveInFlight`) sem expor segredos ou stack traces.
- **Boot Warmup & Graceful Shutdown (`server.ts`):**
  - `app.listen(PORT, "0.0.0.0", ...)` inicia `void prewarmedCache.start()` sem bloquear a abertura da porta HTTP.
  - Handlers `SIGTERM` e `SIGINT` chamam `prewarmedCache.shutdown()`, encerram timers (`legacySyncInterval` e schedulers L1) e fecham `httpServer` com segurança.

---

## 5. Inventário de Arquivos

### Criados
- `src/server/cache/prewarmedGamesCache.ts` — orquestrador de Cache Antecipado L1 (Camadas A/B/C, projeções legadas, LRU histórico e observabilidade).
- `tests/etapa2b5261.test.ts` — suíte automatizada cobrindo todos os 14 cenários e benchmark comparativo frio vs pré-aquecido.
- `temp/etapa-2b-5-2-6-1-cache-antecipado-ram.md` — este relatório técnico.

### Modificados
- `src/server/collectors/bzzoiroCollector.ts` — adição de `parseBzzoiroGoalIncidents` e `collectBzzoiroLiveOverlayDelta`.
- `src/server/collectors/unifiedGamesCollector.ts` — extração de `collectUnifiedCollectionArtifacts`, `composeUnifiedShadowResultFromArtifacts`, `createUnifiedSnapshotId`, `extractUnifiedSnapshotErrors` e `UnifiedCollectionArtifacts`.
- `src/server/registerCollectorRoutes.ts` — integração do `prewarmedCache` nas rotas Central, Shadow e legadas + registro de `POST /api/internal/cache/refresh`.
- `server.ts` — boot warmup não bloqueante, restrição de CORS em `/api/internal/*`, resumo `cache` no `/api/health`, cache L1 em `/api/sports/lacrosse` e graceful shutdown `SIGTERM`/`SIGINT`.
- `.env.example` — documentação de `CACHE_REFRESH_SECRET` e intervalos L1.

### Garantias de Escopo
- **Frontend (`src/App.tsx`, `src/components/**`, `src/services/**`):** nenhuma alteração realizada.
- **Fase 2 (Firestore L2):** não implementada nesta etapa (`firebase-admin` / `games_cache_snapshots` ausentes por escopo).
- **Git / Deploy:** nenhum commit, push ou deploy executado.

---

## 6. Verificações Executadas e Medições

### 6.1 Suíte Automatizada (`npx tsx --test tests/*.test.ts`)
- **Total executado:** `88` testes (`76` testes das etapas anteriores 2A–2B.5.2.4 + `12` subtestes da Etapa 2B.5.2.6.1 cobrindo os 14 cenários exigidos).
- **Resultado:** **88 aprovados / 0 falhas** (`duration_ms: 18781.69`).

Cobertura validada na Etapa 2B.5.2.6.1:
1. Warmup não bloqueante no boot com resposta HTTP imediata.
2. Leituras `GET`/`HEAD /api/games/central`, `GET /api/games/shadow` e filtros executando 100% em RAM (0 chamadas upstream).
3. Refresh periódico de fixtures atualizando `fixturesHash`, grade e `snapshotId` com reconciliação americana preservada.
4. Tick live (`30s`) mantendo `ETag`/`304 Not Modified` em *no-op* e atualizando placar/gols/tempo + novo `ETag` sem reconsultar agenda pesada.
5. Backoff adaptativo (`120s` ocioso vs `30s` ativo) e confirmação segura de saída `LIVE -> FINAL` no Bzzoiro.
6. Preservação do último snapshot íntegro em falha parcial/truncamento (Bzzoiro página 2 / falha live) com `degraded: true` e exposição de `providerStatus`.
7. Coalescing entre boot warmup, refresh autenticado e múltiplas requisições concorrentes (1 única coleta executada).
8. Autenticação rigorosa (`503`/`401`/`403`/`400`/`405`) + CORS restrito em `POST /api/internal/cache/refresh` + `/api/health` passivo.
9. Virada de meia-noite BRT (`23:59 -> 00:00 America/Sao_Paulo`) acionando rollover imediato e descartando dia anterior como data primária.
10. Paridade integral Central `1.0` (`stats.matchStatsId`, `400`, `405`, `Cache-Control`) e LRU histórico limitado sem poluir RAM principal.
11. Graceful shutdown (`SIGTERM`/`SIGINT` / `shutdown()`) limpando timers e abortando ciclos pendentes sem handles presos.
12. Rotas legadas do frontend atual (`/api/football/today`, `/api/sportsblaze/*` incluindo consultas de amanhã via `boxscores`/`schedule`, `/api/jogos-texto`, `/api/jogos-lite`, `/api/sports/lacrosse`) servidas diretamente do L1 em RAM.
13. Benchmark comparativo de latência (Cenário 14).

### 6.2 Medição Comparativa de Latência (Cenário 14 — upstream simulado ~240ms)
| Cenário | Latência medida |
|---|---:|
| Coleta fria on-demand (`GET /api/games/central` sem warmup) | **268.8 ms** |
| Leitura pré-aquecida em RAM L1 (`GET /api/games/central`, 1ª leitura) | **6.3 ms** |
| Leitura pré-aquecida em RAM L1 (`GET /api/games/central`, P95 em série) | **6.3 ms** |
| Lote de rotas legadas do frontend (`/api/football/today`, `/api/sportsblaze/boxscores`, `/api/sportsblaze/schedule`, `/api/jogos-texto`, `/api/sports/lacrosse`) | **22.5 ms** |

### 6.3 Build e Typecheck / Lint
- `compile_applet` (`npm run build`): **sucesso** (`Build succeeded`).
- `lint_applet` (`tsc --noEmit`): **sucesso** (`0` erros).
