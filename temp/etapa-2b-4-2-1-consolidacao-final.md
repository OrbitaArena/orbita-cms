# ÓRBITA ARENA — ETAPA 2B.4.2.1
## Consolidação Final do Contrato da API Central de Jogos (`GET /api/games/central`)

**Versão do Contrato:** `1.0`  
**Documento Base Revisado:** `temp/etapa-2b-4-2-contrato-aprovado.md`  
**Status:** Consolidação Final — Aguardando Aprovação para Etapa 2B.4.3  
**Data:** 2026-11-21  
**Escopo:** Auditoria somente leitura e consolidação documental (nenhum arquivo de produção foi modificado).

---

## A. Resumo Executivo

### Decisões finais consolidadas
1. **Snapshot único compartilhado (Shadow + Central):** `GET /api/games/shadow` e `GET /api/games/central` consumirão o **mesmo snapshot** produzido por `unifiedGamesCollector.ts` (por `referenceDate` + ligas), sem segundo agregador e sem coletas adicionais disparadas por filtros HTTP.
2. **Ciclo de TTL unificado no snapshot compartilhado:**
   - Snapshot saudável (`ok: true, degraded: false`): **60 segundos** (`60_000 ms`).
   - Snapshot degradado (`ok: true, degraded: true`): **20 segundos** (`20_000 ms`).
   - Falha total (`ok: false`): **nunca armazenado como snapshot válido**.
   - **Fallback *stale* acotado (`MAX_STALE_AGE_MS = 180_000 ms`):** caso exista snapshot válido recente e uma revalidação posterior falhe totalmente, a API Central poderá servir o último snapshot válido sinalizando explicitamente `stale: true`, `degraded: true` e idade real, evitando derrubar o site em oscilações momentâneas sem mascarar dados antigos como novos.
3. **Semântica HTTP, CORS e `OPTIONS`/`HEAD`/`405`:**
   - `OPTIONS` já é respondido globalmente pelo middleware CORS em `server.ts:45-54` antes das rotas; a rota `/api/games/central` não deve quebrar preflight.
   - `GET` e `HEAD` compartilham o mesmo pipeline sem duplicar coleta (`HEAD` sem corpo).
   - `POST`, `PUT`, `PATCH`, `DELETE` retornam **`HTTP 405 Method Not Allowed`** com `Allow: GET, HEAD`.
   - Identificada lacuna real no middleware CORS de `server.ts:47-48`: `If-None-Match` não está em `Access-Control-Allow-Headers` e `Access-Control-Expose-Headers` não expõe `ETag` (correção pontual mapeada para a Etapa 2B.4.3).
4. **`ETag` fraco e comparação HTTP (RFC 9110):**
   - `ETag` passa a contemplar **todos** os parâmetros que alteram o corpo (`snapshotId`, `stale`, `degraded`, `sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat`, `includeProvenance`).
   - Suporte completo a `If-None-Match` (tag única, lista separada por vírgula respeitando aspas, `*`, comparação fraca `W/` e tratamento seguro de header malformado).
5. **`includeStats=full` estritamente sem I/O adicional:** projeta exclusivamente estatísticas já presentes no snapshot compartilhado (atualmente apenas boxscores inline da SportsBlaze para NFL/MLB/NBA/WNBA), mantendo `MatchStatsService` + `/api/match-stats` como fluxo independente sob demanda.

### Resultado da revisão cruzada (2B.4.2 × código atual)
Foram identificadas 5 lacunas objetivas no código atual que exigirão ajustes mínimos e cirúrgicos na **Etapa 2B.4.3**:
1. `unifiedShadowCache` (`unifiedGamesCollector.ts:336`) ainda usa TTL fixo de `60_000 ms` para saudáveis e degradados e não gera `snapshotId` nem controla `expiresAtMs` / `stale`.
2. `unifiedGamesCollector.ts:3125-3133` marca `fromCache = true` apenas no acerto direto de cache, mas não armazena identidade única do snapshot compartilhado.
3. `server.ts:47-48` precisa liberar `HEAD`, `If-None-Match` e `Access-Control-Expose-Headers: ETag` no CORS para clientes cross-origin (ex.: Vercel -> Render).
4. A composição de `ETag` na Seção C.6 do documento `2B.4.2` citava `snapshotId + filtersHash`, porém no texto introdutório anterior havia omissão de `sport`, `league`, `status`, `liveOnly` e do estado `stale` — agora todos incluídos explicitamente.
5. `CentralGamesResponse` em `2B.4.2` não possuía o sinalizador booleano `cache.stale` para suportar transparência de fallback *stale*.

---

## B. Cache Compartilhado

### 1. Arquitetura definitiva e fluxo Shadow ↔ Central

```text
Provedores Externos (Bzzoiro / ESPN / SportsBlaze)
   │
   ▼
Coletores Independentes (caches próprios + in-flight coalescing)
  ├── bzzoiroCollector.ts          (TTL 180s)
  ├── espnAmericanCollector.ts     (TTL 60s)
  ├── espnLacrosseCollector.ts     (TTL 60s)
  └── sportsblazeAmericanCollector (TTL 60s boxscores / 300s schedule)
   │
   ▼
Unified Games Collector (src/server/collectors/unifiedGamesCollector.ts)
  ├── Cache Único: unifiedSharedCache (chave: referenceDateBrt + leaguesSig)
  ├── Coalescing Único: unifiedSharedInFlight
  └── Registro armazenado: SharedUnifiedSnapshotEntry
        ├── snapshotId
        ├── createdAtMs / generatedAt
        ├── expiresAtMs (createdAtMs + 60s saudável | + 20s degradado)
        ├── staleUntilMs (createdAtMs + 180s)
        ├── nextRetryAfterMs (anti-loop em falha)
        └── payloadBase (UnifiedGamesShadowResult)
   │
   ├────────────────────────────────────────┐
   ▼                                        ▼
GET /api/games/shadow               GET /api/games/central (e HEAD)
  │                                   │
  ▼                                   ▼
Preserva payload Shadow atual       Projeta e filtra em memória:
(sem breaking change)               sets, sport, league, status, liveOnly,
                                    includeStats, includeCompat, includeProvenance
                                      │
                                      ▼
                                    Validação ETag / If-None-Match -> 200 ou 304
```

### 2. Verificação frente ao código atual (`unifiedGamesCollector.ts`)
- **O que já existe no código:**
  - `unifiedShadowCache` (`Map<string, { timestamp: number; data: UnifiedGamesShadowResult }>`) e `unifiedShadowInFlight` (`Map<string, Promise<UnifiedGamesShadowResult>>`), com chave `unified:shadow:${referenceDateBrt}:${leaguesSig}` e clonagem via `structuredClone`.
  - Coalescing funcional para chamadas simultâneas antes da resolução da promise.
- **O que falta para suportar a arquitetura consolidada (menor alteração possível na Etapa 2B.4.3):**
  1. Evoluir a entrada de `unifiedShadowCache` de `{ timestamp, data }` para `SharedUnifiedSnapshotEntry` contendo `snapshotId`, `createdAtMs`, `expiresAtMs`, `staleUntilMs` e `nextRetryAfterMs`.
  2. Expor uma função interna ou estender o núcleo do agregador para retornar `{ snapshotMeta, shadowResult }` (ou `collectUnifiedGamesSnapshot`), onde:
     - `collectUnifiedGamesShadow()` devolve exatamente `UnifiedGamesShadowResult` sem quebrar consumidores do `/api/games/shadow`.
     - O futuro serviço/handler da API Central consome o **mesmo** `SharedUnifiedSnapshotEntry` e projeta `CentralGamesResponse` estritamente em memória.

### 3. Identidade do snapshot (`snapshotId`)
O snapshot compartilhado é identificado por:
- `snapshotId`: string determinística gerada **somente quando uma nova coleta produz um resultado válido**, no formato:
  `ug:${referenceDate}:${createdAtMs}:${snapshotSeq}` (ex.: `ug:2026-11-20:1795224900000:1`).
- `referenceDate`: data esportiva `YYYY-MM-DD` em `America/Sao_Paulo`.
- `generatedAt`: `new Date(createdAtMs).toISOString()`.
- `degraded`: booleano (`true` se qualquer chamada de provedor falhou).
- `providerStatus`: estado consolidado de execução de `bzzoiro`, `espn` e `sportsblaze`.

**Regras de mutação do `snapshotId`:**
1. **Muda** exclusivamente quando uma nova execução efetiva do agregador produz um novo snapshot (por expiração de TTL ou `bypassCache`).
2. **NÃO muda** quando requisições aplicam filtros diferentes (`sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat`, `includeProvenance`), nem quando um acerto de cache retorna o snapshot existente.

### 4. Atualização, expiração e coalescing
1. **Hit fresco (`nowMs < entry.expiresAtMs`):** retorna imediatamente um clone do `entry` existente (`cache.hit = true`, `cache.stale = false`). Zero chamadas aos coletores.
2. **Miss ou expirado (`nowMs >= entry.expiresAtMs`):**
   - Verifica `unifiedSharedInFlight.get(cacheKey)`. Se existir coleta em andamento, **todas** as requisições simultâneas (sejam de Shadow ou Central, `GET` ou `HEAD`, com quaisquer filtros) aguardam a **mesma Promise**.
   - Somente uma coleta é executada; ao concluir, se `ok === true`, substitui o `entry` no cache e gera novo `snapshotId`.
   - Filtros e projeções HTTP de cada requisição aguardando a Promise são aplicados após sua resolução, de forma independente por requisição, **em memória**.

---

## C. Política de TTL e Fallback *Stale*

### 1. Tabela de TTL no cache compartilhado

| Estado do snapshot coletado | `ok` | `degraded` | TTL no cache compartilhado | Armazenado como snapshot principal? |
|---|---|---|---|---|
| **Saudável** (incluindo *empty slate* válido) | `true` | `false` | **60 segundos** (`60_000 ms`) | Sim |
| **Degradado** (1 ou mais provedores falharam, ao menos 1 saudável) | `true` | `true` | **20 segundos** (`20_000 ms`) | Sim (TTL menor acelera recuperação) |
| **Falha total** (`anyProviderSucceeded === false`) | `false` | `true` | **0 ms (não armazenado)** | **Não** — nunca substitui snapshot anterior válido |

### 2. Avaliação e política oficial de Fallback *Stale*

#### Análise de segurança
- **Quando NÃO há snapshot anterior válido** (cold start) ou o último snapshot válido ultrapassou `MAX_STALE_AGE_MS` (**180 segundos / 3 minutos** desde `createdAtMs`) ou pertence a outra `referenceDate`: **não é seguro servir dados stale** -> a resposta deve ser **`HTTP 503 Service Unavailable`** (`ok: false`, `degraded: true`).
- **Quando EXISTE snapshot anterior válido dentro da janela de 180 segundos** (`nowMs - previousEntry.createdAtMs <= 180_000 ms`) e uma nova tentativa de atualização falha totalmente (`ok: false` em todos os provedores):
  - **É seguro servir o último snapshot válido como fallback *stale***, desde que sejam cumpridas 4 garantias estritas:
    1. **Nunca substituir um snapshot válido por um vazio/inválido:** o `previousEntry` permanece preservado (não sobrescrito pelo objeto falho).
    2. **Sinalização explícita e transparente (proibido mascarar como dado novo):**
       - `ok: true`, `degraded: true`
       - `cache.hit: true`, `cache.stale: true`
       - `cache.ageMs: nowMs - previousEntry.createdAtMs` (idade real preservada, > TTL normal)
       - `generatedAt` preservado como o instante real daquela geração anterior
       - `warnings` acrescido de `"stale_snapshot_fallback"`
       - `errors` acrescido de `{ code: "REVALIDATION_FAILED_USING_STALE", message: "..." }`
       - Header HTTP `Warning: 110 - "Response is Stale"` e `Cache-Control: no-store` (ou `max-age=5, must-revalidate` — preferível `no-store` para evitar que browser/CDN estendam ainda mais um dado *stale*).
    3. **Proteção anti-loop (cooldown de revalidação):** ao falhar a atualização e ativar o modo *stale*, gravar `previousEntry.nextRetryAfterMs = nowMs + 20_000` (20s). Requisições que chegarem dentro desses 20s recebem o fallback *stale* diretamente sem disparar rajadas de chamadas aos provedores caídos; após 20s, uma única requisição (via coalescing) tenta recuperar os provedores novamente.
    4. **Preservação do Shadow (`/api/games/shadow`):**
       - Por padrão, `GET /api/games/shadow` continua executando sua semântica de auditoria atual (`collectUnifiedGamesShadow` preserva os campos esperados; se todos os provedores falharem após expiração, pode manter `502`/`ok: false` via opção `{ allowStaleFallback: false }` ou aceitar stale se explicitado — **recomendado manter `allowStaleFallback: false` no Shadow** para que a rota de auditoria reflita falhas totais imediatamente em monitoramento, enquanto `GET /api/games/central` usa `{ allowStaleFallback: true }` para resiliência ao usuário final!).

### 3. Comportamento em expiração durante coleta em andamento
- Se o snapshot expirou (`nowMs >= expiresAtMs`) e uma coleta de atualização já está em `unifiedSharedInFlight`:
  - Novas requisições aguardam a Promise `inFlight` em andamento (timeout máximo limitado pelos timeouts dos coletores: ~6s).
  - Se a coleta retornar `ok: true` (saudável ou degradada), todas recebem o novo snapshot.
  - Se a coleta retornar `ok: false` e existir `previousEntry` dentro de `180_000 ms`, a rotina devolve o snapshot *stale* sinalizado (`stale: true`) e agenda `nextRetryAfterMs = nowMs + 20_000`. Caso contrário, devolve `503`.

---

## D. Contrato HTTP (`GET`, `HEAD`, `OPTIONS`, `405` e CORS)

### 1. Diagnóstico do middleware CORS real (`server.ts:44-54`)
No código atual (`server.ts:44-54`), existe um middleware global aplicado antes de `registerCollectorRoutes(app, { cbAPI })`:
```ts
app.use((req, res, next) => {
  res.setHeader("Access-Control-Allow-Origin", "*");
  res.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
  res.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
  if (req.method === "OPTIONS") {
    res.sendStatus(200);
  } else {
    next();
  }
});
```
**Conclusões técnicas verificadas:**
1. **`OPTIONS` é tratado globalmente antes das rotas:** ao acessar `OPTIONS /api/games/central` no servidor completo (`server.ts`), o middleware intercepta a requisição e responde `HTTP 200` com os headers CORS antes de chegar a `registerCollectorRoutes`.
2. **Proteção para testes isolados (`registerCollectorRoutes` sem `server.ts`):** ao registrar a rota `/api/games/central` na Etapa 2B.4.3, `OPTIONS` **não** deve cair no handler `405`. A rota deve permitir que `OPTIONS` passe (ou responder `204`/`200` com `Allow: GET, HEAD, OPTIONS` quando chamado diretamente sem middleware anterior), enquanto apenas `POST`, `PUT`, `PATCH`, `DELETE` (e outros métodos não suportados) retornam **`HTTP 405 Method Not Allowed`**.
3. **Ajuste necessário no CORS de `server.ts:47-48` na Etapa 2B.4.3:**
   - Incluir `HEAD` em `Access-Control-Allow-Methods`: `"GET, HEAD, POST, PUT, DELETE, OPTIONS"`.
   - Incluir `If-None-Match` em `Access-Control-Allow-Headers`: `"Content-Type, Authorization, If-None-Match"` (necessário quando um frontend externo envia `If-None-Match` manualmente via `fetch`, evitando bloqueio de preflight).
   - Adicionar `Access-Control-Expose-Headers`: `"ETag, Cache-Control, Allow, Warning"` para que clientes JavaScript cross-origin (ex.: Vercel -> Render) possam ler `response.headers.get("ETag")`.

### 2. Matriz definitiva de métodos HTTP em `/api/games/central`

| Método | Comportamento | Status possíveis | Headers obrigatórios | Corpo |
|---|---|---|---|---|
| **`GET`** | Valida query params -> resolve snapshot compartilhado -> aplica filtros/projeção em memória -> compara `ETag` (`If-None-Match`). | `200`, `304`, `400`, `503` | `Content-Type: application/json; charset=utf-8` (exceto `304`), `ETag` (em `200`/`304`), `Cache-Control`, `Vary: Accept-Encoding` | JSON (`CentralGamesResponse` ou `CentralErrorResponse`), vazio em `304` |
| **`HEAD`** | Executa exatamente o mesmo fluxo de `GET` (mesmo snapshot, mesmos filtros, mesmo status, mesma lógica de `ETag`/`304`), **sem duplicar coleta** e **sem enviar corpo**. | `200`, `304`, `400`, `503` | Idênticos ao `GET`, incluindo `ETag`, `Cache-Control`, `Vary: Accept-Encoding` e `Content-Type` (`Content-Length` calculado quando aplicável em `200`) | **Vazio** |
| **`OPTIONS`** | Tratado pelo middleware global CORS (`server.ts:49-50`) retornando `200` com headers `Access-Control-*`. Se rota for testada isoladamente sem middleware, responde `204` preservando preflight. | `200` / `204` | `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Expose-Headers` | Conforme middleware (`OK` / vazio) |
| **`POST` / `PUT` / `PATCH` / `DELETE`** (e demais) | Rejeitado explicitamente antes do catch-all SPA (`app.get("*")`). | **`405`** | `Allow: GET, HEAD`, `Content-Type: application/json; charset=utf-8`, `Cache-Control: no-store` | `{ "contractVersion": "1.0", "ok": false, "error": { "code": "METHOD_NOT_ALLOWED", "message": "Método não permitido. Utilize GET ou HEAD." } }` |

### 3. Regras de `Cache-Control` por estado
- `200 OK` saudável (`degraded: false`, `stale: false`): `Cache-Control: public, max-age=15, must-revalidate`
- `200 OK` degradado (`degraded: true`, `stale: false`): `Cache-Control: public, max-age=10, must-revalidate`
- `200 OK` fallback *stale* (`stale: true`): `Cache-Control: no-store` + `Warning: 110 - "Response is Stale"`
- `304 Not Modified`: repete o `Cache-Control` correspondente ao estado do snapshot atual (`max-age=15` ou `max-age=10`, nunca superior ao TTL do snapshot)
- `400`, `405`, `503`: `Cache-Control: no-store`

---

## E. ETag, `If-None-Match` e Invalidação

### 1. Composição do `ETag`
A API Central emitirá **ETag fraco** (`W/"..."`), pois campos operacionais do envelope (`cache.hit`, `cache.ageMs`) podem variar entre requisições enquanto os dados esportivos e filtros da representação são semanticamente equivalentes.

**Entrada canônica para o hash da representação (`representationSignature`):**
1. `contractVersion`: `"1.0"`
2. `snapshotId`: identificador único do snapshot compartilhado
3. Estado do snapshot que altera o corpo: `degraded` (`true`/`false`), `stale` (`true`/`false`)
4. **Todos** os parâmetros normalizados que alteram o payload retornado:
   - `referenceDate` (implícito em `snapshotId`, incluído por clareza)
   - `sets` (lista ordenada deduplicada, ex.: `overnight,today,tomorrow`)
   - `sport` (lista ordenada deduplicada ou `all`)
   - `league` (lista ordenada deduplicada em lowercase ou `all`)
   - `status` (lista ordenada deduplicada ou `all`)
   - `liveOnly` (`true` | `false`)
   - `includeStats` (`none` | `summary` | `full`)
   - `includeCompat` (`true` | `false`)
   - `includeProvenance` (`true` | `false`)

**Formato resultante:**  
`W/"${snapshotId}:${sha256_8chars(representationSignature)}"`  
Exemplo: `W/"ug:2026-11-20:1795224900000:1:8f3a19c2"`

> **Observação sobre `cache.hit` e `cache.ageMs`:** conforme RFC 9110 §8.8.3, um ETag fraco (`W/"..."`) indica equivalência semântica da representação. A progressão de `cache.ageMs` e a transição `cache.hit: false -> true` dentro do mesmo snapshot **não invalidam** o ETag fraco.

### 2. Regras de comparação HTTP (`If-None-Match`)
A função de avaliação de `If-None-Match` na Etapa 2B.4.3 deverá seguir estritamente a semântica da RFC 9110 §13.1.2 (função pura dedicada, sem usar `===` simples com a string crua do header):

1. **Ausência do header `If-None-Match` (ou string vazia após trim):** não há correspondência -> retorna resposta normal (`200 OK` + `ETag`).
2. **Valor `*` (`If-None-Match: *`):** corresponde a qualquer representação existente para `GET` e `HEAD` -> retorna **`304 Not Modified`** (para respostas que seriam `200 OK`).
3. **Lista separada por vírgulas:** ex.: `If-None-Match: W/"tag1", "tag2", W/"tag3"`. Deve fazer *parsing* respeitando *quoted-strings* (dividir por vírgulas fora de aspas, fazendo `trim()` em cada item).
4. **Comparação fraca (`weakCompare`):** conforme RFC 9110 §8.8.3.2, duas entity-tags são equivalentes na comparação fraca se seus *opaque-tags* (conteúdo entre aspas, preservando as aspas) forem idênticos byte a byte, **ignorando a presença ou ausência do prefixo exato `W/`** (case-sensitive — `w/` minúsculo é inválido).
   - Ex.: `W/"ug:...:8f3a19c2"` casa com `W/"ug:...:8f3a19c2"` e também com `"ug:...:8f3a19c2"`.
5. **ETags inválidos ou malformados:** tokens sem aspas (ex.: `If-None-Match: abc`), prefixo `w/` minúsculo, ou aspas não fechadas são ignorados com segurança durante o parsing (não lançam exceção nem retornam `400` ou `500`); se nenhum token válido casar, segue fluxo normal `200 OK`.
6. **Não aplicar `304` em respostas de erro:** se a coleta resultou em `503` (sem fallback *stale*) ou query inválida (`400`), `If-None-Match` não retorna `304`.

### 3. Invalidação e relação com cache interno
- O `ETag` **não** substitui o cache interno do agregador nem gera snapshots próprios.
- A verificação de `If-None-Match` ocorre **somente após** resolver/validar o snapshot compartilhado para `referenceDate` (respeitando TTL de 60s/20s e coalescing). Nunca dispara coleta extra apenas para responder `304`.
- Como `snapshotId` muda a cada nova geração do snapshot e `representationSignature` inclui estado (`degraded`/`stale`) e todos os filtros, qualquer atualização de placar/status ou mudança de filtro invalida automaticamente o `ETag` anterior.

---

## F. Estatísticas (`includeStats`) e Fronteira On-Demand

### 1. Princípio de isolamento
A API Central **proíbe** qualquer chamada adicional de estatísticas durante a montagem ou filtragem da resposta. Ela **não** chama `/api/match-stats`, ESPN `summary`, SportsBlaze boxscore individual ou Bzzoiro `/stats/`.

### 2. Comportamento formal dos três modos de `includeStats`

| Modo | Metadados de referência (`available`, `matchStatsId`, `endpoint`, `hasTeamStats`, `hasPlayerStats`) | `stats.keyPlayers` | `stats.teamStatistics` e `stats.players` | `liveOverlay.linescores` e `liveOverlay.incidents` |
|---|---|---|---|---|
| **`none`** | **Sempre preservados** (exatos e funcionais) | Omitido (`undefined`) | Omitidos (`undefined`) | Preservados no `liveOverlay` (fazem parte do placar/lances básicos) |
| **`summary`** *(padrão)* | **Sempre preservados** | Incluído (até 5 atletas/lado) **somente** se `players` existir em memória (SportsBlaze) | Omitidos (`undefined`) | Preservados no `liveOverlay` |
| **`full`** | **Sempre preservados** | Omitido ou mantido? **Regra definida:** omitir `keyPlayers` redundante ou manter? -> **Omitir `keyPlayers` quando `players` completo já é entregue, OU manter `teamStatistics` e `players` com valor real (`object` quando presente em memória, `null` quando indisponível no provedor)** | **Incluídos** com dados já presentes no snapshot compartilhado; **`null` quando indisponível** | Preservados no `liveOverlay` (inclui `linescores.periodScores` quando houver) |

*(Ajuste de clareza em `full`: manter `teamStatistics` e `players` explícitos — objeto completo ou `null` se não houver dados no snapshot — e não fabricar atletas nem métricas. `keyPlayers` fica opcional em `full` para evitar redundância com `players`.)*

### 3. Disponibilidade real por provedor e modalidade

| Origem / Modalidade | Estatísticas presentes no snapshot (`includeStats=full`) | `hasTeamStats` / `hasPlayerStats` | Consulta on-demand via `/api/match-stats` (`stats.available`) | Comportamento quando ausente |
|---|---|---|---|---|
| **SportsBlaze Boxscores** (`NFL`, `MLB`, `NBA`, `WNBA` — `matched` ou `sportsblaze_only`) | `teamStatistics` + `players` + `periodScores` inline | `true` (quando evento iniciou e feed trouxe boxscore) | `true` apenas se **`matched`** (`espnId` presente -> `sb-<lg>-<espnId>`). **`false`** se **`sportsblaze_only`** (`matchStatsId: null`). | Se `UPCOMING` ou apenas `schedule`, `teamStatistics: null` e `players: null`. |
| **ESPN-only** (`NCAAF`, `NHL`, ou `NFL`/`MLB`/`NBA`/`WNBA` sem par SB) | Apenas `linescores` no `liveOverlay` | `false` / `false` | **`true`** (`sb-<lg>-<espnId>`) | `teamStatistics: null`, `players: null`. Detalhes carregados sob demanda pelo frontend via `MatchStatsService`. |
| **Bzzoiro (Futebol)** | Apenas `incidents.homeGoals` / `awayGoals` no `liveOverlay` | `false` / `false` | **`true`** (`bsd-<bzzoiroId>`) | `teamStatistics: null`, `players: null`. Frontend chama `/api/match-stats?id=bsd-...` ao abrir detalhes. |
| **ESPN Lacrosse (PLL)** | Scoreboard básico (sem linescores/atletas) | `false` / `false` | **`false`** (`matchStatsId: null`, `endpoint: null`) | `teamStatistics: null`, `players: null`. Sem suporte em `/api/match-stats` atualmente. |

### 4. Responsabilidade do Frontend (`MatchStatsService`)
- A UI continua exibindo a lista/grade/placar a partir de `GET /api/games/central` (com `includeStats=summary` padrão).
- Ao abrir detalhes completos ou métricas avançadas de uma partida:
  - Se `stats.hasTeamStats === true` e o cliente já tiver obtido via `includeStats=full`, pode exibir diretamente.
  - Caso contrário, se `stats.available === true` e `stats.endpoint !== null`, o frontend aciona `MatchStatsService.getMatchStats(...)` -> `/api/match-stats` sob demanda.
  - Se `stats.available === false` (ex.: `sportsblaze_only` sem dados inline ou `lacrosse`), o frontend **não** chama `/api/match-stats`, evitando consultas quebradas ou fallback errado de futebol.

---

## G. Contrato Consolidado — Alterações Pontuais sobre o Documento `2B.4.2`

As alterações abaixo aplicam-se diretamente sobre `temp/etapa-2b-4-2-contrato-aprovado.md` para produzir a versão final de implementação da Etapa 2B.4.3:

1. **Seção C.1 (Endpoint e Métodos):**
   - Atualizar a linha de métodos para explicitar que `OPTIONS` é interceptado pelo middleware global CORS (`server.ts:45-54`) retornando `200` (e em testes isolados responde `204` sem cair em `405`).
   - Explicitar que `HEAD` usa o mesmo pipeline de `GET`, retornando mesmos headers/status sem corpo e sem duplicar coleta.
   - Registrar a necessidade de atualizar `server.ts:47-48` na Etapa 2B.4.3 para incluir `HEAD` em `Access-Control-Allow-Methods`, `If-None-Match` em `Access-Control-Allow-Headers` e adicionar `Access-Control-Expose-Headers: ETag, Cache-Control, Allow, Warning`.

2. **Seção C.4 (Semântica de Respostas HTTP e Falhas):**
   - Substituir a regra rígida de `503` em falha total quando houver snapshot recente válido em memória: quando `nowMs - previousEntry.createdAtMs <= 180_000 ms` (mesma `referenceDate`), aplicar fallback *stale* (`HTTP 200`, `ok: true`, `degraded: true`, `cache.stale: true`, `warnings: ["stale_snapshot_fallback"]`, `Cache-Control: no-store`, `Warning: 110 - "Response is Stale"`, cooldown de 20s antes da próxima tentativa). Retornar `503` quando não houver snapshot anterior válido ou quando ultrapassar `180_000 ms`.

3. **Seção C.6 (Cache, Coalescing, ETag e Limites):**
   - Consolidar o cache em memória único compartilhado entre `/api/games/shadow` e `/api/games/central` em `unifiedGamesCollector.ts` (`60s` saudável, `20s` degradado, `0s` falha total, janela *stale* `180s` exclusiva para fallback de disponibilidade na Central).
   - Substituir a composição incompleta do `ETag` para incluir obrigatoriamente: `contractVersion`, `snapshotId`, `degraded`, `stale`, `sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat` e `includeProvenance`.
   - Formalizar comparação fraca de `If-None-Match` conforme RFC 9110 §13.1.2 (tag única, lista separada por vírgula respeitando aspas, `*`, equivalência `W/"x"` ↔ `"x"`, ignorar tokens malformados).

4. **Seção D (Interfaces TypeScript Definitivas):**
   - Em `CentralGamesResponse.cache` (`2B.4.2`, linhas 228-232), adicionar o campo obrigatório:
     ```ts
     cache: {
       hit: boolean;
       stale: boolean; // true somente quando servindo último snapshot válido após falha de revalidação
       ageMs: number;
       ttlMs: number;  // 60000 saudável | 20000 degradado | 0 em 503
     };
     ```
   - Em `CentralMatchStatsBlock` (`2B.4.2`, linhas 392-421), clarificar na documentação JSDoc de `keyPlayers` que ele é emitido apenas em `includeStats="summary"` (em `"none"` é omitido; em `"full"` omitido ou opcional, priorizando `teamStatistics` e `players`).

5. **Seção E (Exemplos de Payload):**
   - Em todos os exemplos `1`, `2`, `3`, `4` e `7`, incluir `"stale": false` dentro do objeto `"cache"` para manter 100% de paridade com a interface atualizada.
   - No Exemplo `8` (`HTTP 304`), atualizar o exemplo de `ETag` para refletir o formato com `snapshotId` sequenciado e hash completo de filtros (`W/"ug:2026-11-20:1795224900000:1:8f3a19c2"`).

---

## H. Critérios de Aceite Atualizados (Etapa 2B.4.3)

A implementação na Etapa 2B.4.3 será aprovada somente quando **todos** os critérios abaixo forem verificados por testes automatizados:

### Cache compartilhado e TTL
- [ ] `GET /api/games/shadow` e `GET /api/games/central` utilizam exatamente o mesmo snapshot armazenado em `unifiedGamesCollector.ts` (mesmo `generatedAt` e mesma coleta subjacente).
- [ ] Chamadas simultâneas concorrentes (incluindo combinação Shadow + Central e diferentes filtros) para a mesma `referenceDate` executam apenas **1** coleta nos provedores (in-flight coalescing).
- [ ] Mudança de filtros (`sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat`, `includeProvenance`) no `/api/games/central` opera 100% em memória, mantém o mesmo `snapshotId` e não dispara nenhuma coleta extra.
- [ ] Snapshot saudável permanece válido por **60s** (`60_000 ms`); snapshot degradado permanece válido por **20s** (`20_000 ms`) no cache compartilhado.
- [ ] Falha total (`ok: false`) nunca substitui um snapshot anterior válido no cache nem grava resposta vazia como saudável.
- [ ] Fallback *stale* (quando habilitado na Central e dentro de `<= 180_000 ms`): retorna `200`, `cache.stale = true`, `degraded = true`, `generatedAt` original preservado, `warnings` com `"stale_snapshot_fallback"`, `Cache-Control: no-store`, `Warning: 110 - "Response is Stale"` e cooldown de `20s` anti-loop; fora da janela ou sem snapshot prévio, retorna `503`.
- [ ] `GET /api/games/shadow` mantém 100% de retrocompatibilidade em seu contrato JSON atual.

### Semântica HTTP, CORS e ETag
- [ ] `OPTIONS /api/games/central` retorna sucesso CORS (`200` via middleware ou `204` isolado), sem retornar `405`.
- [ ] `server.ts` expõe `HEAD` em `Access-Control-Allow-Methods`, `If-None-Match` em `Access-Control-Allow-Headers` e `ETag, Cache-Control, Allow, Warning` em `Access-Control-Expose-Headers`.
- [ ] `HEAD /api/games/central` retorna exatamente o mesmo status HTTP e os mesmos headers de `GET` (`ETag`, `Cache-Control`, `Vary`, `Content-Type`), sem corpo e sem coleta duplicada.
- [ ] `POST`, `PUT`, `PATCH` e `DELETE` em `/api/games/central` retornam `405` com header `Allow: GET, HEAD`.
- [ ] `ETag` fraco altera obrigatoriamente quando muda `snapshotId`, `degraded`, `stale`, `sets`, `sport`, `league`, `status`, `liveOnly`, `includeStats`, `includeCompat` ou `includeProvenance`.
- [ ] `If-None-Match` suporta tag única, lista separada por vírgulas, `*`, comparação fraca (`W/"x"` e `"x"`), e ignora valores malformados sem erro.
- [ ] Resposta `304 Not Modified` não dispara coleta adicional, omite corpo e retorna `ETag` e `Cache-Control` coerentes.

### Estatísticas, dados e compatibilidade
- [ ] `includeStats=none` omite dados estatísticos detalhados e preserva `available`, `matchStatsId`, `endpoint`, `hasTeamStats`, `hasPlayerStats` (sem `stats.ref`).
- [ ] `includeStats=summary` inclui apenas `keyPlayers` resumidos quando já disponíveis em memória, sem chamadas extras.
- [ ] `includeStats=full` projeta `teamStatistics` e `players` somente quando já presentes no snapshot compartilhado (`null` nos demais), com **zero** chamadas a `/api/match-stats` ou endpoints individuais externos.
- [ ] `stats.available === true` apenas quando há suporte real no `/api/match-stats`; `sportsblaze_only` e `lacrosse` retornam `stats.available === false`, `stats.matchStatsId === null` e `stats.endpoint === null`.
- [ ] `providerStatus[p].freshness` nunca inventa timestamps indisponíveis e `liveOverlay.eventUpdatedAt` nunca reutiliza `sb.updated` ou `generatedAt`.
- [ ] `sets` e `counts` obedecem estritamente `brtDate.ts` e `gameSets.ts`; contadores sempre batem com os arrays retornados; `providerSnapshots` e `excluded` não vazam.

---

## I. Pendências Técnicas

### 1. Resolvidas documentalmente nesta etapa
- Arquitetura de snapshot compartilhado entre `/api/games/shadow` e `/api/games/central`.
- Política de TTL único compartilhado (`60s` saudável, `20s` degradado, `0s` falha total).
- Política de fallback *stale* acotada (`<= 180s`, cooldown `20s`, sinalização explícita e preservação de `503` em cold start / expiração máxima).
- Semântica completa de `OPTIONS` (CORS), `HEAD` e `405`.
- Composição completa e comparação RFC 9110 de `ETag` / `If-None-Match` / `304`.
- Fronteira estrita de `includeStats=none|summary|full` versus consulta on-demand via `MatchStatsService`.

### 2. Pontos que exigem alteração de código na Etapa 2B.4.3
1. **`src/server/collectors/unifiedGamesCollector.ts`:**
   - Evoluir estrutura do cache interno para `SharedUnifiedSnapshotEntry` com `snapshotId`, `expiresAtMs` (`60s` / `20s`), `staleUntilMs` (`180s`), `nextRetryAfterMs` (`20s`) e suporte à projeção compartilhada entre Shadow e Central sem perder compatibilidade de `collectUnifiedGamesShadow`.
2. **Coletores (`bzzoiroCollector.ts`, `espnAmericanCollector.ts`, `espnLacrosseCollector.ts`):**
   - Opcionalmente (sem quebra de contrato) expor `fetchedAtMs` e `fromCache` a partir do `timestamp` já existente em seus caches para popular `providerStatus[p].freshness.collectorFetchedAt`, `sourceCacheAgeMs` e `sourceCacheHit` fora de `null`.
3. **`src/server/registerCollectorRoutes.ts` (e novo módulo normalizador/filtro da API Central):**
   - Registrar `GET`, `HEAD` e handler `405` (preservando `OPTIONS`) para `/api/games/central`, com validação de query params (`400`), filtragem/projeção em memória, `ETag`/`If-None-Match` (`304`) e headers de cache.
4. **`server.ts` (linhas 47-48):**
   - Atualizar o middleware CORS global para incluir `HEAD` em `Access-Control-Allow-Methods`, `If-None-Match` em `Access-Control-Allow-Headers` e adicionar `Access-Control-Expose-Headers: ETag, Cache-Control, Allow, Warning`.
5. **Testes automatizados (`tests/`):**
   - Criar suíte específica validando todos os itens da Seção H (incluindo cache compartilhado Shadow+Central, TTL 20s degradado, stale fallback, `HEAD`, `OPTIONS`, `405`, `ETag` fraco / `*` / lista / malformado, e `includeStats=none|summary|full`).

### 3. Limitações que precisam ser preservadas
- Armazenamento em memória por processo Node.js (sem sincronização automática de `snapshotId`/`ETag` entre múltiplas instâncias no Render).
- `includeStats=full` restrito ao que os coletores já trazem na coleta diária (SportsBlaze boxscores); ESPN-only, Futebol e Lacrosse retornam `teamStatistics: null` e `players: null` na API Central.
- Ausência de suporte a Lacrosse e SportsBlaze UUID no endpoint `/api/match-stats` atual (`stats.available = false`).
- `liveOverlay.eventUpdatedAt` permanece `null` quando o provedor não envia timestamp individual confiável do evento.

### 4. Melhorias futuras fora do escopo (Etapas posteriores)
- Migração gradual dos consumidores (`JogosDoDiaPage`, `Ticker`, `HeroCommandCenter`, `HeroCopa`, `/api/jogos-texto`, `MeusTimesPage`) para o fluxo central (Etapa 2B.5+).
- Suporte de `/api/match-stats` a `?canonicalId=ug:...`, SportsBlaze UUID e Lacrosse PLL.
- Cache distribuído em Redis para ambientes multi-réplica.
- Endpoint dedicado `GET /api/v1/games` para o ORB Lite.
