# ÓRBITA ARENA — ETAPA 2B.5.2
## Publicação e Validação da API Central no Render (`https://orbitaarena-v2.onrender.com`)

**Data da Auditoria:** sábado, 21/11/2026  
**Horário da Validação Contemporânea:** `15h40` – `15h43 BRT` (`18h40` – `18h43 UTC`)  
**Backend Alvo:** `https://orbitaarena-v2.onrender.com`  
**Endpoint Alvo:** `GET /api/games/central`  
**Escopo Preservado:** Nenhuma alteração realizada em frontend (`JogosDoDiaPage.tsx`), coletores, rotas, contratos, regras BRT ou configurações de infraestrutura.

---

## 1. Estado Inicial do Repositório

### 1.1. Presença dos arquivos da Etapa 2B.4.3 no workspace
Todos os arquivos previstos para a API Central estão presentes e íntegros no workspace (`/app/applet`):
- `src/server/collectors/unifiedGamesCollector.ts` (`122 KB`)
- `src/server/centralGamesApi.ts` (`41 KB`)
- `src/server/registerCollectorRoutes.ts` (`28 KB`, registrando `HEAD`, `GET`, `OPTIONS` e `.all -> 405` em `/api/games/central` antes do fallback SPA)
- `server.ts` (`112 KB`, importando e invocando `registerCollectorRoutes(app, { cbAPI })` na linha `1050` antes de `app.get("*", ...)` nas linhas `2369–2442`)

### 1.2. Estado Git e credenciais de publicação no ambiente
- O workspace `/app/applet` não possui diretório `.git` (`git status` retorna `fatal: not a git repository`), nem possui `GITHUB_TOKEN`, `GH_TOKEN` ou `RENDER_API_KEY` configurados no ambiente.
- Também não existe `.env` local (`.env: DOES NOT EXIST`), e `BZZOIRO_API_KEY` está `UNSET` no sandbox.

### 1.3. Validação local de build, tipagem e testes da Etapa 2B.4.3
- **Typecheck (`npm run lint` / `tsc --noEmit`):** **0 erros**.
- **Build de servidor (`package.json` -> `build` / `build:server`):** `npx esbuild server.ts --bundle --platform=node --format=cjs --packages=external --sourcemap --outfile=...` compilou em `76 ms` gerando bundle único de `409.6 kb` (apenas os 5 avisos já conhecidos de `import.meta` em CJS com fallback seguro para `__filename`/`__dirname`).
- **Start script (`package.json`):** `"start": "node dist/server.cjs"` (`build`: `node copy-assets.cjs && vite build && npx esbuild server.ts ...`).
- **Testes automatizados (`npx tsx --test tests/etapa2b43.test.ts`):** **7/7 testes passaram** (`1282 ms`), validando `GET`, `HEAD`, `OPTIONS`, `405`, CORS, snapshot compartilhado, `ETag`/`304`, filtros em memória, `includeStats` e `503`/stale fallback.

---

## 2. Estado do Deploy

**Status atual:** **NÃO PUBLICADO** no backend de produção (`https://orbitaarena-v2.onrender.com`).

Evidências objetivas obtidas em `21/11/2026` às `18:40 UTC` (`15h40 BRT`):
1. Ao consultar `https://orbitaarena-v2.onrender.com/api/health`, o serviço acabou de iniciar (`"uptime": 7.700057614`, `"environment": "production"`, cabeçalho `x-render-origin-server: Render`).
2. `GET /api/games/central?date=2026-11-21` retornou **HTTP `200` com `Content-Type: text/html; charset=utf-8`** e corpo HTML do fallback SPA (`<!DOCTYPE html> ... <title>Órbita Arena Backend</title> ... Órbita Arena API is ONLINE`).
3. `GET /api/games/shadow?date=2026-11-21` também retornou **HTTP `200` com `text/html`** do fallback SPA, comprovando que nem o Shadow Mode (Etapa 2B.3) nem a API Central (Etapa 2B.4.3) foram ainda publicados nesse serviço Render.
4. `GET /api/sportsblaze/boxscores?league=NBA&date=20261121` retornou `{"success":true,"games":[...]}` **sem** o campo `"date": "20261121"` presente em `src/server/registerCollectorRoutes.ts:142`, confirmando que o código em execução no Render é anterior à extração modular de rotas dos coletores.

Não foi possível realizar o push/deploy diretamente a partir deste ambiente por ausência de repositório Git remoto local e de credenciais de acesso ao GitHub/Render, além da restrição explícita de não criar outra instância ou alterar infraestrutura fora do fluxo estabelecido.

---

## 3. Serviço Render e Branch Utilizados

- **Serviço Render utilizado pelo frontend de produção:** `https://orbitaarena-v2.onrender.com`
  - Comprovado diretamente pela inspeção do bundle ativo em `https://orbitaarena.com.br/jogosdodia` (`/assets/index-CfmzBsp0.js`), que contém `https://orbitaarena-v2.onrender.com` como URL de API (`VITE_API_URL`), e pelo cabeçalho HTTP `x-render-origin-server: Render` retornado nas requisições.
- **Branch e repositório de origem:**
  - Na organização pública `OrbitaArena` no GitHub, apenas o repositório de conteúdo `orbita-cms` é público (branch padrão `main`, último push em `2026-11-21T14:10:16Z`).
  - O repositório do backend conectado ao serviço Render `orbitaarena-v2` é privado (ou está sob conta pessoal do proprietário) e não possui metadata `.git` vinculada neste sandbox, impossibilitando inspecionar remotamente a branch configurada no painel do Render (`main` por convenção usual) ou fila de deploys.

---

## 4. Confirmação da Presença da Variável Secreta (`BZZOIRO_API_KEY`)

- **No backend de produção (`https://orbitaarena-v2.onrender.com`):** **PRESENTE E FUNCIONAL** (comprovação indireta sem exposição de valor).
  - Evidência: `GET https://orbitaarena-v2.onrender.com/api/football/today?date=2026-11-21` retornou `HTTP 200`, `Content-Type: application/json`, `"source": "bzzoiro_v2_live_api"`, `"totalMatches": 229`, `"liveCount": 25` e `56` ligas às `15h40–15h41 BRT`. Sem `BZZOIRO_API_KEY` configurada no Render, essa rota não conseguiria retornar `"source": "bzzoiro_v2_live_api"` com os jogos reais do dia.
- **No ambiente sandbox local:** `BZZOIRO_API_KEY` está **AUSENTE** (`UNSET`), o que fez a execução local do coletor unificado registrar falha controlada `3/3` no provedor `bzzoiro` e `degraded: true`.

---

## 5. Versão Efetivamente Publicada

A versão atualmente rodando em `https://orbitaarena-v2.onrender.com`:
- **Contém:** `/api/health`, `/api/football/today`, `/api/sportsblaze/boxscores` (formato anterior sem campo `date`), `/api/jogos-texto` (`301` jogos, `43` ao vivo às `15h41 BRT`), CORS global respondendo `200 OK` para `OPTIONS`, e fallback `app.get("*")` servindo a página HTML `"Órbita Arena Backend"`.
- **Não contém:** `src/server/registerCollectorRoutes.ts` atualizada, `/api/games/shadow` (Etapa 2B.3) nem `GET / HEAD /api/games/central` (Etapa 2B.4.3).

---

## 6. Resultados dos Testes HTTP em Produção (`https://orbitaarena-v2.onrender.com`)

Executados em `21/11/2026` às `15h40–15h41 BRT` (`18h40–18h41 UTC`), sem provocar indisponibilidade ou alteração de dados:

| Teste | Resultado Real em Produção | Esperado pelo Contrato (`2B.4.3`) | Status |
|---|---|---|---|
| `GET /api/health` | `HTTP 200`, `application/json`, `{"status":"ok","environment":"production"}` | `HTTP 200 JSON` | OK |
| `GET /api/games/central?date=2026-11-21` | `HTTP 200`, **`text/html; charset=utf-8`**, HTML `"Órbita Arena Backend"` | `HTTP 200`, `application/json`, JSON contrato `1.0` | **FALHOU (SPA fallback)** |
| `HEAD /api/games/central?date=2026-11-21` | `HTTP 200`, **`text/html; charset=utf-8`**, sem `ETag` de snapshot | `HTTP 200`, `application/json`, `ETag` e `Cache-Control` | **FALHOU (SPA fallback)** |
| `OPTIONS /api/games/central` | `HTTP 200`, `text/plain`, corpo `"OK"`, `Allow: null` | `HTTP 204` (ou CORS `Allow: GET, HEAD, OPTIONS` com headers `If-None-Match` expostos) | **FALHOU** |
| `POST /api/games/central` | **`HTTP 404`**, `text/html` (`Cannot POST /api/games/central`), `Allow: null` | **`HTTP 405`**, JSON, `Allow: GET, HEAD, OPTIONS` | **FALHOU** |
| `GET /api/games/central` + `If-None-Match` (`304`) | *Não aplicável enquanto retorna fallback HTML* | `HTTP 304 Not Modified` | **BLOQUEADO** |
| `GET /api/games/central?date=invalida` (`400`) | *Não aplicável enquanto cai no wildcard `app.get("*")` HTML* | `HTTP 400`, `INVALID_DATE` em JSON | **BLOQUEADO** |
| `GET /api/games/shadow?date=2026-11-21` | `HTTP 200`, **`text/html; charset=utf-8`**, HTML `"Órbita Arena Backend"` | `HTTP 200 JSON` | **FALHOU (não publicado)** |

*(Nota: os requisitos de campos do contrato — `contractVersion`, `snapshotId`, `referenceDate`, `generatedAt`, `degraded`, `cache.stale`, `sets`, `providerStatus`, `304` e `400` — já estão verificados localmente via `tests/etapa2b43.test.ts` e express local, mas permanecem bloqueados para certificação em produção até o deploy no Render.)*

---

## 7. Estado dos Provedores (Verificação Contemporânea `15h41–15h43 BRT`)

| Provedor | Estado Real Observado | Evidência |
|---|---|---|
| **Bzzoiro (Futebol)** | **Saudável em produção** / **Ausência controlada no sandbox** | Produção (`/api/football/today`) entregou `229` jogos (`25` ao vivo, `56` ligas). No sandbox sem `BZZOIRO_API_KEY`, falhou `3/3` chamadas e marcou `degraded: true`, sem mascarar como saudável. |
| **ESPN American** | **Saudável** | `21/21` chamadas OK (`108` itens deduplicados), fornecendo `72` jogos `TODAY` (`55` NCAAF, `12` NHL, `5` NBA) e `12` americanos `TOMORROW`, além de todos os `18` placares ao vivo americanos no momento da captura. |
| **ESPN Lacrosse (PLL)** | **Saudável (slate vazio fora de temporada)** | Consultas concluídas com `HTTP 200`, `0` eventos para `2026-11-21`. |
| **SportsBlaze** | **Saudável sem chave ou Bearer Token** | `16/16` chamadas OK aos feeds públicos (`cache.sportsblaze.com`), `21` itens deduplicados, `feedUpdatedAt: "2026-11-21T18:33:36.998Z"`, sem expor chaves nem metadados sensíveis. |

---

## 8. Resultado da Comparação Contemporânea (`21/11/2026`, `15h41–15h43 BRT`)

Como `GET /api/games/central` ainda não está publicado em `orbitaarena-v2.onrender.com`, a comparação contemporânea foi feita entre:
1. O fluxo atual real em produção (`orbitaarena-v2.onrender.com/api/football/today` + `/api/sportsblaze/boxscores` + `/api/jogos-texto`, usado por `/jogosdodia`);
2. A execução ao vivo do coletor da API Central local no mesmo instante (`15h43m17s BRT`).

| Métrica (`2026-11-21` às `15h41–15h43 BRT`) | Fluxo Atual Produção (`/jogosdodia` & `/api/jogos-texto`) | API Central Local (sem `BZZOIRO_API_KEY` no sandbox) | Análise de Divergência |
|---|---:|---:|---|
| **Total `TODAY`** | **301** | **74** (`degraded: true`) | `-229` de futebol (chave Bzzoiro ausente no sandbox) e `+2` de NBA `00:00 BRT` via `sportsblaze_only`. |
| Futebol `TODAY` | **229** (`25` ao vivo) | `0` (`bzzoiro.ok: false`) | Falta exclusivamente publicação no Render onde `BZZOIRO_API_KEY` está ativa. |
| Esportes Americanos `TODAY` | **72** (`55` NCAAF, `12` NHL, `5` NBA) | **74** (`55` NCAAF, `12` NHL, `7` NBA) | Paridade `72/72` + `2` jogos NBA da virada `00:00 BRT` (`sportsblaze_only`). |
| Lacrosse `TODAY` | **0** | **0** | Paridade `0/0`. |
| **Ao Vivo Total (`TODAY`)** | **43** (`25` Futebol + `18` Americanos) | **18** (`0` Futebol + `18` Americanos) | Nos esportes americanos: **`18 / 18` jogos ao vivo (`16` NCAAF, `1` NHL, `1` NBA) com correspondência exata de ID, status `LIVE` e placares (`0` divergências)** — ex.: `#24 Florida Gators 17-24 Vanderbilt Commodores (Halftime)`, `#25 Ole Miss Rebels 49-3 Wofford Terriers (Halftime)`. |
| **`TOMORROW` (`2026-11-22`)** | — | **23** americanos (`11` NFL `matched`, `7` NBA, `5` NHL) | Coerente (sem futebol no sandbox pela ausência da chave Bzzoiro). |

---

## 9. Problemas Encontrados

1. **Bloqueio Principal — Código da Etapa 2B.4.3 não publicado no Render:**
   - `https://orbitaarena-v2.onrender.com` executa uma versão anterior do backend sem `/api/games/central`, sem `/api/games/shadow` e sem `registerCollectorRoutes.ts`. Todas as requisições a `/api/games/central` (`GET`/`HEAD`) caem no wildcard `app.get("*")` e devolvem `text/html`.
2. **Bloqueio Operacional de Publicação a partir deste Sandbox:**
   - O diretório `/app/applet` não possui histórico/remote `.git` nem variáveis `GITHUB_TOKEN` / `RENDER_API_KEY`, impedindo verificar o histórico de commits da branch do Render ou disparar o deploy automaticamente sem intervenção do responsável pelo projeto.
3. **Observação técnica sobre porta em `server.ts:57` (informativa, não bloqueante):**
   - `server.ts:57` mantém `const PORT = 3000;` hardcoded (restrição declarada do ambiente AI Studio). Como `orbitaarena-v2.onrender.com` já responde normalmente em produção usando a mesma base `server.ts`, o serviço Render já está configurado com `PORT=3000` ou detecção automática de porta; nenhuma alteração de infraestrutura é necessária.

### Procedimento exato para publicação pelo responsável (desbloqueio)
1. **Sincronizar os arquivos do backend** desta revisão para o repositório Git privado conectado ao serviço Render `orbitaarena-v2`, garantindo a inclusão de:
   - `server.ts`
   - `src/server/centralGamesApi.ts`
   - `src/server/registerCollectorRoutes.ts`
   - `src/server/collectors/unifiedGamesCollector.ts` (e demais coletores/utilitários `src/server/collectors/*`, `src/utils/brtDate.ts`, `src/utils/gameSets.ts`)
   - `package.json` (`build` e `start`)
2. **Conferir no painel do Render (`orbitaarena-v2`):**
   - **Build Command:** `npm run build` (ou `npm install && npm run build`)
   - **Start Command:** `npm run start` (`node dist/server.cjs`)
   - **Environment Variables:** confirmar `NODE_ENV=production` e manter `BZZOIRO_API_KEY` já existente (além de `PORT=3000` se já definida).
3. **Realizar commit e `git push`** na branch monitorada pelo Render (ou acionar **Manual Deploy → Deploy latest commit** no dashboard do Render).
4. **Reexecutar a bateria de validação HTTP e paridade em `https://orbitaarena-v2.onrender.com/api/games/central`** após o deploy concluir e retornar `HTTP 200 JSON` com `degraded: false`.

---

## 10. Veredicto

> **Veredicto:** **`Bloqueada`**

**Motivo objetivo:** Embora a implementação local da Etapa 2B.4.3 esteja íntegra, compile sem erros (`npm run lint` + `esbuild`) e passe em todos os testes (`7/7`), e embora o serviço Render correto (`https://orbitaarena-v2.onrender.com`) já possua `BZZOIRO_API_KEY` funcional em produção, **o endpoint `GET /api/games/central` ainda não está publicado no Render** (retorna HTML do fallback SPA) e este ambiente não dispõe de acesso Git/Render para publicar o deploy. A aprovação desta etapa fica condicionada à publicação do código no repositório conectado ao serviço `orbitaarena-v2` e à validação HTTP/paridade pós-deploy com resposta `JSON`, `degraded: false` e `stale: false`.
