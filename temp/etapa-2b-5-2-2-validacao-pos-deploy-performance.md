# ÓRBITA ARENA — ETAPA 2B.5.2.2
## Validação Pós-Deploy, Desempenho, Cache e Resiliência da API Central

**Data da Auditoria:** domingo, 22/11/2026  
**Janela da Validação em Produção:** `11h12` – `11h16 BRT` (`14h12` – `14h16 UTC`)  
**Backend Auditado:** `https://orbitaarena-v2.onrender.com`  
**Frontend de Produção Auditado:** `https://orbitaarena.com.br/jogosdodia`  
**Escopo Preservado:** Auditoria exclusivamente de leitura e testes HTTP não destrutivos em produção. Nenhuma alteração foi realizada em `src/**`, `server.ts`, `tests/**`, contratos, TTLs, timeouts ou configurações do Render/Vercel.

---

## 1. Validação Funcional da Central (`https://orbitaarena-v2.onrender.com`)

### 1.1. Disponibilidade, Contrato e Métodos HTTP
A API Central da Etapa 2B.4.3 encontra-se **publicada e operacional** no Render (`x-render-origin-server: Render`):

| Verificação | Método / Rota | Status HTTP | Tempo | Evidência Observada | Resultado |
|---|---|---:|---:|---|---|
| Health Check | `GET /api/health` | `200` | `238 ms` | `{"status":"ok","uptime":1142.87,"environment":"production"}` | **OK** |
| Leitura padrão (`TODAY` + `OVERNIGHT` + `TOMORROW`) | `GET /api/games/central?date=2026-11-22` | `200` | `44.353 ms` (frio) / `128–169 ms` (HIT) | `Content-Type: application/json; charset=utf-8`, `contractVersion: "1.0"`, `timezone: "America/Sao_Paulo"`, estrutura completa (`window`, `cache`, `filters`, `counts`, `providerStatus`, `warnings`, `errors`, `sets`) | **OK** |
| Revalidação condicional (`ETag`) | `GET /api/games/central?date=2026-11-22` + `If-None-Match` | `304` | `84 ms` | Corpo `0 B`, preservando `ETag` (`W/"ug:2026-11-22:1795356790783:1:e3118e78"`) e `Cache-Control: public, max-age=15, must-revalidate` | **OK** |
| Cabeçalhos sem corpo | `HEAD /api/games/central?date=2026-11-22` | `200` | `114 ms` | Corpo `0 B`, `ETag` e `Cache-Control` presentes *(obs.: proxy Cloudflare/Brotli reescreve `Content-Length` para `"1"` no `HEAD`)* | **OK** |
| Projeção e filtros em memória | `GET ...&includeStats=none&includeCompat=true` / `&sport=basketball&includeStats=none` | `200` | `503 ms` (HIT) / `17.357 ms` (pós-TTL 180s) | `ETag` distinto por assinatura de filtros (`:f87a5623`, `:97c89b50`), `counts.returned` vs `counts.snapshotTotal` consistentes (`16` de `212` ao filtrar `basketball`) | **OK** |
| Data inválida | `GET /api/games/central?date=2026-99-99` | `400` | `1.573 ms` | JSON `{"contractVersion":"1.0","ok":false,"error":{"code":"INVALID_DATE",...,"param":"date"}}`, `Cache-Control: no-store` | **OK** |
| Método não permitido | `POST /api/games/central` | `405` | `1.764 ms` | JSON `METHOD_NOT_ALLOWED`, `Allow: GET, HEAD, OPTIONS`, `Cache-Control: no-store` | **OK** |
| Preflight CORS | `OPTIONS /api/games/central` | `200` | `1.680 ms` | Retornou `text/plain` (`"OK"`, `2 B`) sem header `Allow` *(o middleware global de CORS em `server.ts:50-51` intercepta `OPTIONS` antes da rota dedicada em `registerCollectorRoutes.ts:717-724`, que previa `204` + `Allow`)* | **Divergência menor** |
| Paridade e snapshot compartilhado com Shadow | `GET /api/games/shadow?date=2026-11-22` | `200` | `211 ms` | `fromCache: true`, `generatedAt: "2026-11-22T14:13:10.783Z"` **idêntico** ao da Central, mesmos totais (`124` today, `0` overnight, `42` tomorrow, `166` total) | **OK** |

### 1.2. Comparação Contemporânea com o Baseline de Produção (`22/11/2026`)
Baseline informado pelo projeto às **`10h57 BRT`**: **Total `169`** | **Futebol `146`** | **Esportes Americanos `23`** | **Ao vivo `31`**.

Durante a captura no Render entre `11h13` e `11h16 BRT`, dois comportamentos distintos do Bzzoiro ocorreram na produção:

| Fonte / Snapshot (Horário BRT) | `TODAY` Total | Futebol (`TODAY`) | Americanos (`TODAY`) | Lacrosse | Ao Vivo (`TODAY`) | `TOMORROW` | Observação Objetiva |
|---|---:|---:|---:|---:|---:|---:|---|
| **Baseline informado (`10h57 BRT`)** | **169** | **146** | **23** | `0` | **31** | — | Fotografia de referência do fluxo atual |
| **Central Snapshot `:1` (`11h13:10 BRT`)** | **124** | **100** | **24** | `0` | **13** | `42` | **Truncamento silencioso na página 2 do Bzzoiro** (ver Seção 5): apenas `100` jogos a partir das `11h00` BRT; `0` jogos `FINAL` (`00h00–10h59` perdidos), mas `degraded: false`! |
| **Central Snapshot `:2` (`11h14:20 BRT`)** | **124** | **100** | **24** | `0` | **13** | `42` | Revalidou Central após `60s`, mas reutilizou o mesmo cache de `180s` truncado do Bzzoiro |
| **Legado `/api/football/today` & `/api/jogos-texto` (`11h14:32 BRT`)** | **123** | **100** | **23** | `0` | **13** | — | Mesmos `100` jogos do `footballCache`; Americanos = `23` (`11` NFL, `7` NBA, `5` NHL) |
| **Central Snapshot `:3` (`11h15:48 BRT`, pós-TTL 180s)** | **170** | **146** | **24** | `0` | **42** | `42` | Coleta completa do Bzzoiro (`402` brutos vs `326` anteriores): **146/146 jogos de futebol** (`42` ao vivo, `86` agendados, `17` encerrados, `1` adiado). `24` americanos (`11` NFL, `8` NBA, `5` NHL — o `+1` é `Sacramento Kings vs Boston Celtics` `00:00 FINAL` via `sportsblaze_only`). Aumento de `31 -> 42` ao vivo reflete a entrada dos jogos das `11h00 BRT`. |

---

## 2. Tabela de Tempos das Requisições (Sequência Controlada em Produção)

Sequência executada contra `https://orbitaarena-v2.onrender.com/api/games/central?date=2026-11-22` (além da verificação complementar após expiração dos `180s` do Bzzoiro):

| # | Marco / Condição | Horário Início (`BRT`) | Duração Total | HTTP | Payload (`Bytes`) | `snapshotId` | `cache.hit` | `cache.stale` | `cache.ageMs` | `cache.ttlMs` | `degraded` | `counts.returned.today` (`live`) |
|---|---|---|---:|---:|---:|---|---|---|---:|---:|---|---:|
| 1 | **`t = 0s`** (Central fria + Bzzoiro frio) | `11:12:26.660` | **`44.353 ms`** | `200` | `247.438` | `ug:2026-11-22:1795356790783:1` | `false` | `false` | `79` | `60.000` | `false` | `124` (`13`) |
| 2 | **`+5s`** após Req 1 | `11:13:16.033` | **`128 ms`** | `200` | `247.439` | `ug:2026-11-22:1795356790783:1` | `true` | `false` | `5.332` | `60.000` | `false` | `124` (`13`) |
| 3 | **`+15s`** após Req 1 | `11:13:26.022` | **`167 ms`** | `200` | `247.440` | `ug:2026-11-22:1795356790783:1` | `true` | `false` | `15.329` | `60.000` | `false` | `124` (`13`) |
| 4 | **`+30s`** após Req 1 | `11:13:41.022` | **`169 ms`** | `200` | `247.440` | `ug:2026-11-22:1795356790783:1` | `true` | `false` | `30.336` | `60.000` | `false` | `124` (`13`) |
| 4b | `If-None-Match` (`+30.2s`) | `11:13:41.20` | **`84 ms`** | `304` | `0` | *(mesmo ETag `:1`)* | — | — | — | — | — | — |
| 4c | `HEAD /api/games/central` (`+30.3s`) | `11:13:41.28` | **`114 ms`** | `200` | `0` | *(mesmo ETag `:1`)* | — | — | — | — | — | — |
| 4d | `GET` com `includeStats=none&includeCompat=true` (`+30.8s`) | `11:13:41.392` | **`503 ms`** | `200` | `269.425` | `ug:2026-11-22:1795356790783:1` | `true` | `false` | `30.772` | `60.000` | `false` | `124` (`13`) |
| 4e | `GET /api/games/shadow` (`+31.1s`) | `11:13:41.897` | **`211 ms`** | `200` | `755.557` | *(generatedAt idêntico)* | `true` (`fromCache`) | — | — | — | `false` | `124` |
| 5 | **`+65s`** após Req 1 (TTL `60s` Central expirado; Bzzoiro `180s` ainda quente) | `11:14:16.022` | **`5.072 ms`** | `200` | `247.437` | `ug:2026-11-22:1795356860862:2` | `false` | `false` | `97` | `60.000` | `false` | `124` (`13`) |
| 6 | **Pós-TTL `180s` do Bzzoiro** (`sport=basketball&includeStats=none`) | `11:15:31` | **`17.357 ms`** | `200` | `24.932` | `ug:2026-11-22:1795356948665:3` | `false` | `false` | `100` | `60.000` | `false` | `8` ret. / **`170` (`42`)** snap. |

*(Nota técnica: mesmo nos misses `#1`, `#5` e `#6`, `cache.ageMs` aparece entre `79–100 ms` porque `createdAtMs` é gravado ao concluir a coleta e `finishNowMs` é lido após o `cloneShadowResult` via `structuredClone` em `unifiedGamesCollector.ts:3471-3545`.)*

---

## 3. Análise do Cache

### 3.1. O cache realmente reduz o tempo de carregamento?
**Sim, de forma expressiva (redução de até 346x):**
- **Cold hit (`Req 1`):** `44.353 ms` (44,35s).
- **Revalidação sem Bzzoiro frio (`Req 5`):** `5.072 ms` (5,07s).
- **Revalidação com Bzzoiro frio (`Req 6`):** `17.357 ms` (17,36s).
- **Cache HIT na Central (`Req 2–4`):** **`128–169 ms`** (`~0,13–0,17s`).
- **Cache HIT com `ETag` (`304 Not Modified`):** **`84 ms`** (`0 B` trafegados).

### 3.2. Arquitetura multi-camada observada e efeitos colaterais
No código (`unifiedGamesCollector.ts:407-416, 3322-3584` e coletores), existem **dois níveis de cache em memória** independentes no mesmo processo Node:

1. **Camada 1 — Snapshot Compartilhado (`unifiedSharedCache` + `unifiedSharedInFlight`):**
   - TTL saudável: **`60.000 ms` (`60s`)**.
   - TTL degradado: **`20.000 ms` (`20s`)**.
   - Janela *stale fallback*: **`180.000 ms` (`180s`)** — **somente usada se todos os provedores falharem (`!result.ok`)** ou durante cooldown de falha (`20.000 ms`). Não há *stale-while-revalidate* assíncrono em revalidação normal.
   - Coalescência de concorrência (`unifiedSharedInFlight`): múltiplos requests simultâneos (inclusive entre `/api/games/central` e `/api/games/shadow`) compartilham a mesma execução em voo.
2. **Camada 2 — Caches internos de cada coletor:**
   - **Bzzoiro (`bzzoiroCollector.ts:13, 1117, 758, 565`):** `footballCache` = **`180.000 ms` (`3 min`)**; `EVENT_BROADCASTS_CACHE` = **`300.000 ms` (`5 min`)**; `BSD_LEAGUES_CACHE` / `BSD_VENUES_CACHE` = **`7.200.000 ms` (`2h`)**.
   - **ESPN American (`espnAmericanCollector.ts:84`):** **`60.000 ms`**.
   - **SportsBlaze (`sportsblazeAmericanCollector.ts:122-123`):** boxscores = **`60.000 ms`**, schedule = **`300.000 ms`**.

**Consequências críticas do desalinhamento de TTLs:**
- **Envenenamento por `180s`:** como `footballCache` (`180s`) dura `3x` mais que o cache da Central (`60s`), quando Bzzoiro devolve lista vazia (`fallback_resilience`) ou lista truncada (`100` jogos como ocorreu na `Req 1`), o erro fica congelado no `footballCache` durante 3 minutos. A `Req 5` revalidou a Central após `60s`, gastou `5,07s` em ESPN+SportsBlaze e voltou a servir os mesmos `100` jogos incompletos.
- **Bloqueio síncrono na expiração:** qualquer usuário cuja requisição coincida com `age >= 60s` paga `~5s` síncronos, e quem coincidir com `age >= 180s` do Bzzoiro (ou frio total) paga **`17s–44s`** bloqueado aguardando resposta.
- **CDN Cloudflare na frente do Render:** todas as respostas do endpoint central retornaram `cf-cache-status: DYNAMIC`; ou seja, `Cache-Control: public, max-age=15, must-revalidate` atua no cache do navegador/cliente, não no edge do Cloudflare.

---

## 4. Diagnóstico dos Provedores

### 4.1. Orquestração (`unifiedGamesCollector.ts:2733-3030`)
Na camada superior do agregador, os coletores são disparados **em paralelo** via `Promise.all([bzzTask, ...espnAmTasks, ...espnLaxTasks, ...sbTasks])` (`linha 3030`), totalizando **40 chamadas lógicas** por coleta para 1 data de referência (`[yesterday, today, tomorrow]`):
- **Bzzoiro:** `3` chamadas (1 por data).
- **ESPN:** `21` chamadas (`6` ligas americanas × `3` datas = `18` + `3` datas de Lacrosse PLL).
- **SportsBlaze:** `16` chamadas (`4` ligas × `3` datas de boxscores = `12` + `4` schedules).

Entretanto, `Promise.all` só conclui quando a **tarefa mais lenta** termina. Comparação empírica:

| Provedor | Chamadas na Coleta | Latência Individual Medida (HTTPS direto) | Latência Agregada Inferida no Render | Padrão de Execução Interno |
|---|---:|---|---|---|
| **ESPN** | `21` | `159–278 ms` (NBA `278 ms`, NFL `195 ms`, NHL `159 ms`, PLL `199 ms`) | Somado ao SportsBlaze + montagem = **`~5,0s`** (`Req 5`) | Paralelo (`1` request por liga/data, timeout padrão `10s`) |
| **SportsBlaze** | `16` | `145–248 ms` (NFL box `145 ms`, NBA sched `158 ms`, 404s rápidos `152–248 ms`) | Integrado aos **`~5,0s`** (`Req 5`) | Paralelo (`cache.sportsblaze.com`, `404` em slates sem jogos tratado como `emptySlateCalls`) |
| **Bzzoiro** | `3` datas (que explodem em **centenas de sub-requisições**) | Não mensurável isoladamente sem logs do Render | **`~39s`** na `Req 1` (`44,35s - 5,07s`) e **`~12s`** na `Req 6` (`17,36s - 5,07s`, já com `EVENT_BROADCASTS_CACHE` e ligas/venues pré-aquecidos!) | **Sequencial em cascata com N+1 massivo** (ver abaixo) |

### 4.2. Gargalo interno do `bzzoiroCollector.ts` (`linhas 1147-1528`)
Para **cada uma das 3 datas** (`yesterday`, `today`, `tomorrow`) executando concorrentemente:
1. **Passo 1 (sequencial):** `events/live/?page_size=100` (`timeout 3.500 ms`, somente para `today`).
2. **Passo 2 (sequencial por página):** `while (currentUrl && pageCount < 10)` sobre `/events/?page_size=100` (`timeout 4.000 ms` por página). Em dias cheios (`2026-11-21` = 199 jogos; `2026-11-22` = 146 jogos), são 2 páginas por dia.
3. **Passo 3 (sequencial após eventos):** `resolveMissingVenues` (`linha 624`) e `resolveMissingLeagues` (`linha 659`) — disparam 1 requisição por ID faltante **SEM nenhum `timeout`**.
4. **Passo 4 (sequencial após Passo 3):** `getBsdBroadcastsForEvents` (`linhas 768-835`) — processa eventos não cacheados em **lotes sequenciais de 25**, fazendo **1 chamada HTTP `/api/v2/broadcasts/?event_id=...` por jogo** (`timeout 2.000 ms`). Com ~372 eventos nas 3 datas (`199` ontem + `146` hoje + `27` amanhã) em cache frio, são até **8 lotes sequenciais para ontem + 6 para hoje + 2 para amanhã**.
5. **Passo 5 (sequencial após Passo 4):** `matchesWithGoals` (`linhas 1431-1499`) — processa todos os jogos com placar > 0 em **lotes sequenciais de 10**, fazendo **1 chamada HTTP `/api/v2/events/${m.id}/incidents/` por jogo SEM nenhum `timeout`**. Como **ontem (`2026-11-21`) teve 199 jogos quase todos `FINAL` com gols**, o agregador ainda buscou dezenas/centenas de `incidents` e `broadcasts` de jogos de ontem que foram **100% descartados logo depois** por `partitionGamesIntoSets` (`excluded: 273`, sendo `199` de futebol)!

Esse pipeline explica com precisão matemática a diferença entre **`44,35s`** (`Req 1`, tudo frio) e **`17,36s`** (`Req 6`, quando `EVENT_BROADCASTS_CACHE` de 5 minutos, `BSD_LEAGUES_CACHE` e `BSD_VENUES_CACHE` já estavam quentes da `Req 1`, restando apenas `/events/` + `incidents/`).

---

## 5. Diagnóstico Específico do Bzzoiro (Falhas Silenciosas)

**Pergunta 4: Quando o Bzzoiro falha, a API consegue identificar corretamente que houve degradação?**  
**Resposta: NÃO.** A investigação combinada do código e do incidente real observado hoje em produção comprova **dois mecanismos de mascaramento silencioso**:

### 5.1. Caso 1 — Falha Total (0 jogos retornados)
Em `src/server/collectors/bzzoiroCollector.ts`:
- `collectBzzoiroFootballToday` só retorna `{ ok: false, statusCode: 503 }` quando `!apiKey` (`linhas 1122-1135`).
- Em **qualquer erro HTTP (401, 403, 429, 500, 502, 503), timeout ou falha de rede** nas chamadas `/events/live/` (`linha 1183`) ou `/events/` (`linhas 1207, 1224-1231`), o erro é capturado/ignorado e `rawResults` fica vazio:
  - Se houver `footballCache[cacheKey]?.data` anterior (mesmo vencido!), a função retorna **o cache antigo com `{ ok: true, statusCode: 200 }`** (`linhas 1236-1239`).
  - Se o cache estiver frio, a função monta `responsePayload` com `totalMatches: 0`, `leagues: []`, **`source: "fallback_resilience"`** (`linha 1523`), **grava essa resposta vazia em `footballCache[cacheKey]` por `180s`** (`linha 1527`) e retorna **`{ ok: true, statusCode: 200 }`** (`linha 1528`).
- No `unifiedGamesCollector.ts`:
  - Como `res.ok === true`, a linha `2753` executa `bzzReport.succeededCalls++` (`3/3`), mantendo `failedCalls: 0`, `errors: []`, **`providerStatus.bzzoiro.ok: true`** (`linha 3036`) e **`degraded: false`** (`linhas 3145-3146`).
  - A linha `2783` (`if (dateMatchesCount === 0 && res.payload?.source !== "fallback_resilience") bzzReport.emptySlateCalls++`) apenas suprime o contador de `emptySlateCalls`, **mas não contabiliza falha nem degradação**.
  - Consequência: se o Bzzoiro cair em cold-start, a API Central responde `ok: true, degraded: false` com `0` jogos de futebol por até 3 minutos.

### 5.2. Caso 2 — Truncamento Parcial por Timeout de Paginação (Comprovado ao vivo hoje!)
- Na `Req 1` (`11h12:26 -> 11h13:11 BRT`), o Bzzoiro entregou a página 1 (`100` jogos das `11h00–22h30 BRT`), porém falhou/excedeu o timeout de `4.000 ms` (`linha 1204`) na página 2.
- O `break` silencioso nas linhas `1207` e `1230` retornou **apenas `100` jogos** com `source: "bzzoiro_v2_live_api"`, **`ok: true`**, `degraded: false`, `partial: false`, `failedCalls: 0` e gravou o recorte incompleto no `footballCache` por `180s`.
- Como consequência objetiva medida no Render, **46 jogos reais de futebol (`00h00–10h59 BRT`: 29 ao vivo + 17 encerrados) desapareceram do Snapshot `:1` e `:2`** sem nenhum alerta em `warnings`, `errors` ou `providerStatus.bzzoiro`, reaparecendo automaticamente no Snapshot `:3` (`146` jogos) assim que os `180s` expiraram.

*(No frontend atual `JogosDoDiaPage.tsx:269-272, 905-945`, o comportamento também mascara falhas: exceções retornam `[]` e o painel exibe o badge verde `"Online"` incondicionalmente.)*

---

## 6. Análise do Recarregamento da Página (`/jogosdodia`)

Inspeção direta do bundle de produção em `https://orbitaarena.com.br/jogosdodia` (`/assets/index-CFZLfs72.js`, `2.19 MB`) confirma que a página ainda usa o fluxo legado apontando para `VITE_API_URL = https://orbitaarena-v2.onrender.com` (sem qualquer referência a `/api/games/central`).

**Pergunta 5: Ao recarregar a página `/jogosdodia`, as chamadas aos provedores estão sendo repetidas ou apenas um snapshot armazenado é consultado?**  
**Resposta:** **No navegador, TODAS as chamadas HTTP são repetidas (zero cache persistente); no Render, elas batem nos caches legados em memória (`180s` Bzzoiro / `60s` ESPN), mas timeouts curtos no frontend podem disparar uma tempestade extra de chamadas ao provedor.**

Fluxo exato executado a cada `F5` / recarregamento de `/jogosdodia`:
1. **Perda de cache no cliente:** `JogosDoDiaPage.tsx` mantém partidas somente em estado React e `CMSService.cache` apenas em RAM da aba; nada é salvo em `localStorage`/`sessionStorage` para partidas do dia.
2. **Duplicação automática entre `App.tsx` e `JogosDoDiaPage.tsx`:**
   - Mesmo na rota `/jogosdodia`, `App.tsx:449-625` executa `fetchFreshSportsData()`, disparando `/api/sports-data`, `/api/football/today` (**sem timeout**), `/api/sports/lacrosse`, `CMSService.getBroadcastData()` e **15 chamadas** via `SportsBlazeService` (`today` + `yesterday` boxscores + `tomorrow` schedule para `nfl, mlb, nba, nhl, wnba`). Se o futebol vier vazio, `App.tsx:621-623` ainda chama `CMSService.getFootballToday()` uma segunda vez.
   - Em paralelo, `JogosDoDiaPage.tsx:194-374` dispara `/api/health` (`timeout 2.5s`), `/api/football/today` (**timeout `6.000 ms`**, `cmsService.ts:1501`), **6 chamadas** `SportsBlazeService.getBoxscores` (**timeout `4.000 ms`**, `sportsblazeService.ts:498` — duplicando 5 ligas de `App.tsx`), `CMSService.getBroadcastData(true)` (`bypassCache = true`), `CMSService.getAoVivoStreams(true)` (`bypassCache = true`), chamada direta do navegador ao endpoint ESPN Lacrosse PLL e `<ESPNBottomLineTicker />` (`newsTicker.json` a cada `60s`).
3. **Efeito em cascata quando o Render está frio (`> 4s / > 6s`):**
   - Com cache quente no Render, `/api/football/today` (`134 ms`) e `/api/sportsblaze/boxscores` (`125 ms`) respondem rápido dos caches legados (`180s`/`60s`).
   - Porém, sempre que o `footballCache` de `180s` expira (levando `12s–39s` para coletar), o cliente **aborta `/api/football/today` aos `6.000 ms`** (`cmsService.ts:1501`) e ativa **`fetchBzzoiroTodayDirect`** (`cmsService.ts:2400`), fazendo o navegador disparar **dezenas/centenas de requisições via `/api/bsd-proxy/*`** (cada uma com timeout de 5s no servidor em `server.ts:1646` e 4s no cliente), competindo diretamente com a coleta que o próprio backend já está executando! Do mesmo modo, `SportsBlazeService` aborta aos `4.000 ms` e recorre a fetches diretos na ESPN pelo browser.

---

## 7. Causas Identificadas para a Lentidão (30s+)

Ordem de impacto comprovada pelas medições e análise de código:

1. **Causa Principal (~88% dos 44,35s medidos na `Req 1`) — Coleta fria N+1 sequencial no `bzzoiroCollector.ts` multiplicada por 3 datas:**
   - Lotes sequenciais de `25` chamadas HTTP para `broadcasts/` + lotes sequenciais de `10` chamadas HTTP **sem timeout** para `incidents/` + chamadas **sem timeout** para `venues/` e `leagues/`, executadas para `yesterday`, `today` e `tomorrow` — incluindo **~199 partidas `FINAL` de ontem** que serão descartadas pelo particionamento logo em seguida.
   - O `Promise.all` em `unifiedGamesCollector.ts:3030` retém a resposta da API Central até a conclusão total do Bzzoiro.
2. **Causa Secundária (~5s medidos na `Req 5`) — Revalidação síncrona sem *stale-while-revalidate*:**
   - Ao expirar o TTL de `60s` da Central, o próximo usuário aguarda de forma síncrona as `21` chamadas à ESPN + `16` à SportsBlaze (e, quando vencem os `180s` do Bzzoiro, `17s–44s`).
3. **Causa Agravante no Frontend Atual — Contenção por timeouts curtos (`4s`/`6s`) + duplicação `App.tsx` × `JogosDoDiaPage.tsx`:**
   - Cada F5 com backend frio aborta aos `6s` e dispara segunda onda N+1 pelo `/api/bsd-proxy/*`, além de ~25+ requisições paralelas redundantes.
4. **Causa Situacional — Cold start de instância no Render:**
   - Embora durante este teste a instância já estivesse ativa (`uptime: 1.142,87s` = 19 min), em períodos pós-hibernação no Render o tempo de spin-up da instância soma-se integralmente aos `17s–44s` da primeira coleta.

---

## 8. Limitações de Observabilidade

Durante esta auditoria foram identificadas lacunas objetivas que reduzem a visibilidade operacional em produção:

1. **Ausência de `durationMs` por provedor e por etapa:** nem a API Central nem os logs estruturados expõem quanto tempo foi gasto em `bzzoiro`, `espn`, `sportsblaze`, paginação, `broadcasts` ou `incidents`.
2. **Campos `freshness` nulos para Bzzoiro e ESPN:** em `unifiedGamesCollector.ts:419-440`, `createEmptyProviderFreshnessMeta()` inicializa `feedUpdatedAt`, `collectorFetchedAt`, `sourceCacheAgeMs` e `sourceCacheHit` como `null`, e apenas `providerMeta.sportsblaze` (`feedUpdatedAt` e `sourceCacheHit`) é preenchido nas linhas `3032-3033`. Para `bzzoiro` e `espn`, todos permanecem `null` em produção.
3. **Omissão de erros parciais do Bzzoiro:** timeouts/erros de página (`events/`), `events/live/`, `broadcasts/`, `incidents/`, `venues/` e `fallback_resilience` são tratados silenciosamente em `bzzoiroCollector.ts` (apenas `console.warn`/`console.log`), sem popular `providerStatus.bzzoiro.errors`, `failedCalls`, `partial` ou `degraded`.
4. **Acesso externo ao ambiente:** não há acesso direto aos logs de stdout do servidor Render nem métricas de CPU/memória/event-loop durante a coleta; toda comprovação em produção baseou-se em medições HTTP externas e auditoria estática do código-fonte publicado.

---

## 9. Recomendações Priorizadas (Sem Implementação Nesta Etapa)

### P0 — Confiabilidade (Pré-requisito de segurança para migração)
1. **Corrigir sinalização de falha e truncamento no `bzzoiroCollector.ts` + `unifiedGamesCollector.ts`:**
   - Retornar `ok: false` (ou flag explícita de falha/truncamento) quando todas as páginas de `/events/` falharem ou quando ocorrer `break` prematuro por timeout/erro HTTP em paginação (marcando `partial: true`, adicionando erro em `providerStatus.bzzoiro.errors` e ativando `degraded: true` na Central).
   - **Nunca gravar** respostas vazias (`fallback_resilience`) ou truncadas por timeout no `footballCache` com TTL longo de `180s` (usar TTL `0` ou curto para falhas e permitir que a Central sirva seu *stale fallback* íntegro de `180s`).
2. **Adicionar timeout explícito nas chamadas descobertas do `bzzoiroCollector.ts`:**
   - Aplicar `fetchWithTimeout` em `resolveMissingVenues` (`linha 624`), `resolveMissingLeagues` (`linha 659`) e `events/${m.id}/incidents/` (`linha 1438`) para evitar travamentos indefinidos da promise.
3. **Corrigir precedência do `OPTIONS /api/games/central`:**
   - Ajustar o middleware global em `server.ts:45-55` para delegar `OPTIONS /api/games/central` ao handler específico de `registerCollectorRoutes.ts:717-724` (`204` + `Allow: GET, HEAD, OPTIONS`).

### P1 — Desempenho (Eliminação dos picos de 17s–44s)
1. **Cortar o N+1 desnecessário na coleta da API Central (`bzzoiroCollector` / `unifiedGamesCollector`):**
   - Não enriquecer `broadcasts` nem `incidents` para a data `yesterday` (ou filtrar antes do enriquecimento apenas partidas não finalizadas / virada `OVERNIGHT`, evitando ~199 jogos descartados).
   - Avaliar carregar `incidents` (artilheiros) sob demanda ou somente para `TODAY` com cache dedicado de longa duração para jogos `FINAL`, e revisar paralelismo/timeouts de paginação de `/events/` (`4s` mostrou-se apertado para página 2 hoje).
2. **Adotar *stale-while-revalidate* (SWR) e/ou refresh proativo em background no backend:**
   - Servir imediatamente o snapshot recente enquanto a renovação roda em background quando próximo da expiração (ou manter um aquecimento periódico enquanto houver tráfego), garantindo `< 200 ms` contínuos sem picos síncronos para o usuário final.
   - Alinhar/revisar o TTL de `180s` do `footballCache` com o ciclo de `60s` da Central.
3. **Instrumentar observabilidade mínima por provedor:**
   - Preencher `collectorFetchedAt`, `sourceCacheAgeMs` e `sourceCacheHit` para `bzzoiro` e `espn`, além de registrar `durationMs` de cada coletor nos logs/diagnóstico.
4. **Ajustar estratégia de chamada na futura migração do frontend (`JogosDoDiaPage.tsx` + `App.tsx`):**
   - Remover a execução redundante das chamadas de jogos no `App.tsx` quando o usuário estiver em `/jogosdodia` (ou compartilhar o mesmo consumidor de `/api/games/central`), evitando tempestade de fallback `/api/bsd-proxy/*` e definindo timeout compatível com o cold-path enquanto P1.1/P1.2 estiverem em rollout.

### P2 — Experiência do Usuário
1. **Cache curto no navegador + `If-None-Match` (`ETag`):**
   - Persistir último snapshot em `sessionStorage`/`localStorage` (com chave por data BRT) para renderização instantânea no F5 enquanto valida `ETag` (`304 Not Modified`, `84 ms`) em segundo plano.
2. **Status real no painel "Diagnóstico de Sincronização":**
   - Substituir o badge fixo `"Online"` em `JogosDoDiaPage.tsx:910-932` por leitura real de `providerStatus` (`ok`, `partial`), `degraded` e `cache.stale`.
3. **Payload enxuto na listagem inicial:**
   - Avaliar uso de `includeStats=none` na carga inicial da grade e carregar detalhes sob demanda via `/api/match-stats`.

---

## 10. Veredicto Sobre a Prontidão para Migrar o Jogos do Dia

> **Veredicto:** **`Parcialmente pronta — apta para migração condicionada à aplicação prévia (ou conjunta) dos itens P0.1, P0.2 e P1.1 / ajuste de timeout no cliente`**

### Justificativa Objetiva
- **O que já está aprovado em produção:** a API Central (`/api/games/central`, `HEAD`, `304`, filtros, `400`, `405` e cache compartilhado com `/api/games/shadow`) **funciona corretamente no Render**, reduz a latência em cache HIT para **`128–169 ms`** (`84 ms` em `304`) e entrega **paridade integral (`146/146` futebol, `24` americanos com `+1` jogo legítimo de virada)** quando a coleta do Bzzoiro conclui sem truncamento.
- **Por que NÃO migrar "às cegas" no estado atual:**
  1. **Risco de perda silenciosa de dados:** conforme comprovado ao vivo às `11h13 BRT`, um timeout na página 2 do Bzzoiro reduziu o futebol de `146` para `100` jogos (e `0` encerrados) mantendo **`degraded: false` e `ok: true`**, e congelando o erro por `180s` (o mesmo ocorre em falha total com `fallback_resilience`).
  2. **Risco de timeout/espera de 17s–44s:** em coleta fria (ou a cada expiração do `footballCache` de `180s`), a Central demorou **`17,4s–44,4s`** por conta do N+1 sequencial do Bzzoiro em 3 datas — acima dos timeouts legados do frontend (`4s`/`6s`).
- **Conclusão prática:** corrigindo a propagação de erros/truncamento e timeouts do Bzzoiro (**P0.1–P0.2**), cortando o N+1 desnecessário de `yesterday`/SWR (**P1.1–P1.2**) e eliminando a duplicação com `App.tsx` (**P1.4**), a API Central terá confiabilidade e velocidade superiores ao fluxo legado para assumir a produção do `/jogosdodia`.
