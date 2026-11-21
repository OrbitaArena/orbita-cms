# ÓRBITA ARENA — ETAPA 2B.4.3
## Relatório Técnico da Implementação da API Central de Jogos (`GET / HEAD /api/games/central`)

**Data:** 2026-11-21  
**Versão do Contrato Implementado:** `1.0`  
**Documentos Base:** `temp/etapa-2b-4-2-contrato-aprovado.md` e `temp/etapa-2b-4-2-1-consolidacao-final.md`  
**Escopo:** Implementação do snapshot compartilhado, rota pública somente leitura (`/api/games/central`), `ETag`/`If-None-Match`, políticas de cache/TTL/stale, validação de query params, suíte de testes automatizados e ajustes pontuais de CORS — **sem migrar nenhum consumidor do frontend**.

---

## 1. Diagnóstico Inicial Confirmado

Antes da implementação, a auditoria do repositório confirmou:
1. Os arquivos `temp/etapa-2b-4-2-contrato-aprovado.md` e `temp/etapa-2b-4-2-1-consolidacao-final.md` estavam presentes como especificação consolidada (`temp/etapa-2b-4-1-contrato-aprovado.md` não existia como arquivo separado, estando incorporado nos documentos `2B.4.2` e `2B.4.2.1`).
2. `unifiedGamesCollector.ts` mantinha `unifiedShadowCache` com TTL fixo de `60_000 ms` mesmo quando `degraded === true`, sem `snapshotId`, sem expiração diferenciada (`20s` degradado), sem janela *stale* acotada (`180s`) e sem proteção anti-loop (`20s`).
3. `server.ts:45-54` tinha middleware global CORS interceptando `OPTIONS` (`200`), porém sem `HEAD` em `Access-Control-Allow-Methods`, sem `If-None-Match` em `Access-Control-Allow-Headers` e sem `Access-Control-Expose-Headers`.
4. `/api/games/central` ainda não existia em `src/server/registerCollectorRoutes.ts`.

---

## 2. Arquivos Criados e Modificados

| Arquivo | Tipo | Finalidade |
|---|---|---|
| `src/server/collectors/unifiedGamesCollector.ts` | Modificado | Evolução do cache para `SharedUnifiedSnapshotEntry` compartilhado entre Shadow e Central (`60s` saudável / `20s` degradado / `180s` janela *stale* / `20s` cooldown anti-loop), geração de `snapshotId`, metadados de frescor da SportsBlaze e exportação de `collectUnifiedGamesSnapshot`. |
| `src/server/centralGamesApi.ts` | Criado | Módulo dedicado da API Central: parser/validador rigoroso (`400`), projeção `CentralGamesResponse`/`CentralGameMatch` 100% em memória, modos `includeStats=none\|summary\|full`, proteção `stats.available`, `ETag` fraco, comparação RFC 9110 `If-None-Match`, políticas `Cache-Control`/`Warning` e handler unificado `GET`/`HEAD`. |
| `src/server/registerCollectorRoutes.ts` | Modificado | Registro explícito de `HEAD`, `GET`, `OPTIONS` (`204` em modo isolado) e `.all` (`405` com `Allow: GET, HEAD, OPTIONS`) para `/api/games/central`, antes do catch-all SPA. Mantida integralmente a rota `/api/games/shadow` e demais rotas legadas. |
| `server.ts` | Modificado | Atualização pontual do middleware CORS global (`HEAD` em `Access-Control-Allow-Methods`, `If-None-Match` em `Access-Control-Allow-Headers` e `Access-Control-Expose-Headers: ETag, Cache-Control, Allow, Warning`). |
| `tests/etapa2b43.test.ts` | Criado | Suíte automatizada cobrindo HTTP (`GET`/`HEAD`/`OPTIONS`/`405`/CORS), snapshot compartilhado Shadow+Central, coalescing, TTL `60s`/`20s`, fallback *stale* `180s`, cooldown `20s`, isolamento do Shadow (`502` sem stale), filtros em memória, `includeStats`, `stats.available`, `ETag`/`If-None-Match` (`304` vs stale `200`) e segurança. |

**Arquivos explicitamente preservados sem alterações:**
- `src/utils/brtDate.ts` e `src/utils/gameSets.ts`
- `src/server/collectors/bzzoiroCollector.ts`, `espnAmericanCollector.ts`, `espnLacrosseCollector.ts`, `sportsblazeAmericanCollector.ts`
- Todos os consumidores do frontend (`JogosDoDiaPage.tsx`, `Ticker.tsx`, `HeroCommandCenter.tsx`, `HeroCopa.tsx`, `HomeLite.tsx`, `MeusTimesPage.tsx`, `MatchStatsService`, `sportsData.json`) e todas as rotas legadas.

---

## 3. Como Funciona o Snapshot Compartilhado Entre Shadow e Central

1. Tanto `GET /api/games/shadow` (`collectUnifiedGamesShadow`) quanto `GET / HEAD /api/games/central` (`collectUnifiedGamesSnapshot`) operam sobre o mesmo mapa `unifiedSharedCache` e a mesma fila de *coalescing* `unifiedSharedInFlight` indexados por `unified:shadow:${referenceDateBrt}:${leaguesSig}`.
2. **Coalescing único:** requisições simultâneas misturando Shadow e Central (com quaisquer filtros) na mesma data aguardam a **mesma Promise** em execução e recebem o mesmo `snapshotId` e `generatedAt`, com **apenas 1 coleta** nos provedores.
3. **Identidade determinística:** `snapshotId` segue o formato `ug:${referenceDateBrt}:${createdAtMs}:${seq}`, mudando apenas quando uma nova execução válida (`ok: true`) substitui o registro em cache. Filtros HTTP e *cache hits* nunca alteram o `snapshotId`.
4. **TTL compartilhado diferenciado:**
   - Snapshot saudável (`ok: true, degraded: false`): `60_000 ms` (`60s`).
   - Snapshot degradado (`ok: true, degraded: true`): `20_000 ms` (`20s`), acelerando recuperação automática.
   - Falha total (`ok: false`): **nunca** sobrescreve `previousEntry` e nunca é gravada como snapshot válido.

---

## 4. Funcionamento do Fallback *Stale*, Cooldown e Isolamento do Shadow

Quando um snapshot válido já existe em `unifiedSharedCache` e uma revalidação após expiração do TTL falha totalmente (`ok: false`):
- **Preservação + Cooldown (`20s`):** o registro válido anterior é preservado intacto e recebe `nextRetryAfterMs = nowMs + 20_000` junto ao último erro (`lastRevalidationFailure`).
- **Central (`allowStaleFallback: true`):**
  - Se `nowMs <= staleUntilMs` (`createdAtMs + 180_000 ms`): responde `HTTP 200`, `ok: true`, `degraded: true`, `cache.stale: true`, `cache.hit: true`, preservando `snapshotId`, `generatedAt` e `ageMs` reais, adicionando `warnings: ["stale_snapshot_fallback"]`, `errors` (`REVALIDATION_FAILED_USING_STALE` + falhas dos provedores), `Cache-Control: no-store` e `Warning: 110 - "Response is Stale"`.
  - Durante o cooldown de `20s`, requisições subsequentes retornam esse fallback *stale* sem martelar provedores caídos.
  - Expirado o limite de `180_000 ms` (ou em *cold start* sem snapshot prévio), a Central devolve **`HTTP 503`** com `Cache-Control: no-store`.
- **Shadow (`allowStaleFallback: false`):**
  - Nunca recebe o snapshot *stale* como coleta válida.
  - Em falha total (incluindo durante o cooldown de `20s` após falha), retorna `ok: false` (`HTTP 502` na rota `/api/games/shadow`), mantendo fidelidade imediata para auditoria e sem disparar coleta duplicada durante o cooldown ativo.

---

## 5. Como `ETag`, `If-None-Match` e `Cache-Control` Foram Implementados

1. **`ETag` fraco determinístico:** `W/"${snapshotId}:${sha256_8chars(signature)}"`, onde `signature` inclui `contractVersion`, `snapshotId`, `referenceDate`, `degraded`, `stale` e todos os filtros normalizados (`sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat`, `includeProvenance`). A progressão de `cache.ageMs` e `cache.hit: false -> true` dentro do mesmo snapshot preserva o mesmo `ETag`.
2. **Avaliação RFC 9110 §13.1.2 (`matchesIfNoneMatch`):**
   - Suporta tag única, lista separada por vírgulas respeitando aspas, `*` e comparação fraca (`W/"tag"` ↔ `"tag"`).
   - Rejeita/ignora com segurança prefixo `w/` minúsculo, tokens sem aspas e aspas não fechadas sem lançar exceções ou retornar `400`/`500`.
   - Só avalia `304` após validar/resolver o snapshot em cache, sem disparos adicionais aos provedores.
   - **Proteção *stale*:** quando `cache.stale === true`, a Central **nunca** retorna `304` (mesmo que `If-None-Match` coincida), devolvendo sempre `200` com corpo, `Cache-Control: no-store` e `Warning: 110 - "Response is Stale"`.
3. **Política de `Cache-Control`:**
   - `200` saudável (`degraded: false, stale: false`): `public, max-age=15, must-revalidate`
   - `200` degradado (`degraded: true, stale: false`): `public, max-age=10, must-revalidate`
   - `200` *stale* (`stale: true`): `no-store` (+ `Warning: 110 - "Response is Stale"`)
   - `304 Not Modified`: repete `Cache-Control` (`max-age=15` ou `max-age=10`), `ETag` e `Vary: Accept-Encoding`
   - `400`, `405`, `503`: `no-store`

---

## 6. Filtros e Projeções em Memória (Sem Coletas Adicionais)

- Parâmetros validados com limites rígidos (`<= 20` tokens em listas gerais, `<= 10` em `status`, `<= 64` chars/token, rejeição de caracteres de controle e arrays de query): `date`, `sets`, `sport` (canônicos + aliases `futebol`, `nba`, `wnba`, `nfl`, `ncaaf`, `mlb`, `nhl`, `pll`), `league` (código, `leagueId` numérico de futebol ou nome normalizado sem acentos), `status` (`LIVE`, `UPCOMING`, `FINAL`, `POSTPONED` + aliases PT-BR), `liveOnly`, `includeStats`, `includeCompat` e `includeProvenance`.
- Parâmetros desconhecidos (ex.: `_t`) são ignorados sem alterar cache nem `ETag`.
- **Proteções aplicadas:**
  - `TODAY`, `OVERNIGHT` e `TOMORROW` seguem exatamente `brtDate.ts` e `gameSets.ts` (sem misturar `OVERNIGHT` em `counts.*.today`).
  - `counts.returned` reflete exatamente os arrays retornados; `counts.snapshotTotal` reflete o snapshot completo.
  - `providerSnapshots` e `excluded` do Shadow nunca vazam.
  - `providerStatus[p].freshness` nunca fabrica `collectorFetchedAt` ou `sourceCacheAgeMs` (`null` quando os coletores não expõem) e `liveOverlay.eventUpdatedAt` nunca copia `sb.updated`, `generatedAt` ou `startTime`.

---

## 7. Fronteira de `includeStats` vs `/api/match-stats`

- **`includeStats=none`:** omite `keyPlayers`, `teamStatistics` e `players`; mantém sempre os 5 campos planos (`available`, `matchStatsId`, `endpoint`, `hasTeamStats`, `hasPlayerStats`) sem criar `stats.ref` e preservando `liveOverlay.linescores` e `incidents`.
- **`includeStats=summary` (padrão):** inclui `keyPlayers` resumidos (até 5 atletas/lado) apenas quando `players` boxscores já existem em memória (SportsBlaze); omite `teamStatistics` e `players`.
- **`includeStats=full`:** omite `keyPlayers` e projeta `teamStatistics` e `players` exclusivamente a partir dos dados já presentes no snapshot compartilhado (SportsBlaze), retornando `null` explícito nos demais casos (Futebol, ESPN-only, Lacrosse), com **zero** chamadas externas adicionais.
- **Proteção `stats.available`:** `true` apenas quando existe endpoint funcional no `/api/match-stats` (`bsd-<id>` no futebol; `sb-<lg>-<espnId>` em americanos `matched`/`espn_only`); `sportsblaze_only` (UUID) e `lacrosse` retornam `available: false`, `matchStatsId: null` e `endpoint: null`.

---

## 8. Resultados dos Testes, Build e Verificação

- **Testes Etapa 2B.4.3 + Regressão Core (`npx tsx --test tests/etapa2b43.test.ts tests/etapa2b3.test.ts tests/etapa2b32.test.ts tests/etapa2a.test.ts`):** **38/38 testes passaram** (0 falhas).
- **Observação transparente sobre suítes antigas:** ao rodar `tests/*.test.ts` completo antes das edições, havia 2 falhas pré-existentes dependentes da data corrente (`etapa2b2.test.ts:625` e `etapa2b21.test.ts:413`), em arquivos não modificados nesta etapa (`brtDate.ts`, `gameSets.ts` e testes legados permaneceram intocados conforme escopo).
- **Typecheck (`npm run lint` / `tsc --noEmit`):** **0 erros**.
- **Build de produção (`npm run build` / `compile_applet`):** **sucesso** (`dist/` e `dist-server/server.js` compilados).
- **Smoke check HTTP:** servidor local não estava em execução na porta 3000 do ambiente sandbox; toda a pilha Express completa (incluindo middleware CORS, `GET`, `HEAD`, `OPTIONS`, `405`, `400`, `503`, `304`, fallback SPA abaixo da rota e stale fallback) foi verificada em servidor Express real na porta efêmera dentro de `tests/etapa2b43.test.ts`.
- **Revisão de segurança:** confirmada ausência de segredos hardcoded, validação estrita de todos os query params e headers condicionais, ausência de vazamento de stack traces/tokens internos (`bzzoiroApiKey`, `providerSnapshots`) e proteção contra fallback HTML SPA nas rotas da API.

---

## 9. Confirmação de Preservação dos Consumidores e Próximo Passo

- **Nenhum consumidor do frontend** (`JogosDoDiaPage.tsx`, `Ticker.tsx`, `HeroCommandCenter.tsx`, `HeroCopa.tsx`, `HomeLite.tsx`, `MeusTimesPage.tsx`, `MatchStatsService`) foi modificado ou migrado nesta etapa.
- **Próximo passo recomendado (Etapa 2B.5):** iniciar a migração gradual controlada dos consumidores começando por `JogosDoDiaPage.tsx` (e `/api/jogos-texto` consumindo o snapshot compartilhado internamente), mantendo paridade visual e funcional integral.
