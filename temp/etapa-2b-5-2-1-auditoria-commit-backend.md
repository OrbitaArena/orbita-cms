# ÓRBITA ARENA — ETAPA 2B.5.2.1
## Auditoria de Alterações Pendentes e Preparação do Commit do Backend

**Data da Auditoria:** sábado, 21/11/2026  
**Escopo:** Auditoria de dependências, separação frontend × backend, verificação de integridade (lint, build, testes e segurança) e preparação da lista exata de commit para publicação da API Central no Render (`https://orbitaarena-v2.onrender.com`).  
**Restrição cumprida:** Nenhum commit, push, deploy ou alteração de código/configuração foi executado.

---

## 1. Estado Atual do Workspace

1. **Ausência de metadados `.git` neste sandbox:** A raiz `/app/applet` não contém diretório `.git` (`git status` retorna `fatal: not a git repository`), pois o ambiente AI Studio exporta os arquivos sem o histórico de commits do GitHub. Consequentemente, a diferença exata contra o `origin/main` não pode ser obtida por `git diff` aqui dentro e foi reconstruída com rigor a partir:
   - do mapa completo de compilação gerado pelo `esbuild` sobre `server.ts`;
   - dos relatórios das etapas anteriores (`temp/etapa-2b-4-2-contrato-aprovado.md`, `temp/etapa-2b-4-2-1-consolidacao-final.md`, `temp/etapa-2b-4-3-implementacao-central.md`, `temp/etapa-2b-5-2-validacao-central-producao.md`);
   - da inspeção estática de `server.ts`, `src/server/**`, `src/utils/**`, `src/services/**`, `src/components/**`, `tests/**` e scripts na raiz.
2. **Defasagem comprovada entre workspace local e Render (`orbitaarena-v2.onrender.com`):**
   - Conforme auditado na Etapa 2B.5.2, a produção executa versão anterior às etapas **2B.2 / 2B.3 / 2B.4.3** (`GET /api/games/central` e `/api/games/shadow` devolvem o HTML fallback da SPA e `/api/sportsblaze/boxscores` ainda retorna payload sem o campo `"date"`).
3. **Integridade local confirmada:** `npm run lint` (`0` erros), `npm run build` (`vite build` + `dist/server.cjs` `409.6 kb`) e os testes da Etapa 2B.4.3 (`7/7`) executam com sucesso neste workspace.

---

## 2. Inventário de Alterações Pendentes (Classificação A / B / C)

### Categoria A — Necessárias para publicar a API Central no Render (Etapas 2A → 2B.4.3)
Arquivos criados ou evoluídos ao longo das etapas de centralização que compõem diretamente o runtime do servidor Express e a rota `GET / HEAD /api/games/central`:

| Arquivo | Origem / Etapa | Estado no Fluxo | Função na Central |
|---|---|---|---|
| `src/utils/brtDate.ts` | Etapa 2A *(compartilhado)* | Modificado/Padronizado | Parsing estrito de timestamps por provedor (`bzzoiro`, `espn`, `sportsblaze`) e conversões BRT (`America/Sao_Paulo`). |
| `src/utils/gameSets.ts` | Etapa 2A | Novo | Classificação determinística em `TODAY`, `OVERNIGHT`, `TOMORROW` e `NONE`. |
| `src/server/collectors/bzzoiroCollector.ts` | Etapa 2B.2 | Novo | Coletor independente de futebol (Bzzoiro API v2), cache `180s`, coalescing e leitura exclusiva de `BZZOIRO_API_KEY`. |
| `src/server/collectors/espnAmericanCollector.ts` | Etapa 2B.2 / 2B.3.2 | Novo | Coletor independente ESPN (`MLB`, `WNBA`, `NFL`, `NCAAF`, `NHL`, `NBA`), cache `60s`, coalescing e `espnLiveDetail`. |
| `src/server/collectors/espnLacrosseCollector.ts` | Etapa 2B.2 | Novo | Coletor independente ESPN Lacrosse (`PLL`). |
| `src/server/collectors/sportsblazeAmericanCollector.ts` | Etapa 2B.2.1 | Novo | Coletor independente SportsBlaze (`NFL`, `MLB`, `NBA`, `WNBA`) via `cache.sportsblaze.com` sem autenticação. |
| `src/server/collectors/unifiedGamesCollector.ts` | Etapas 2B.3 / 2B.3.2 / 2B.4.3 | Novo | Agregador unificado, reconciliação multi-provedor, proveniência ao vivo, cache compartilhado Shadow+Central (`60s`/`20s`), janela *stale* (`180s`) e cooldown (`20s`). |
| `src/server/centralGamesApi.ts` | Etapa 2B.4.3 | Novo | Contrato público `1.0` (`GET`/`HEAD /api/games/central`), validação `400`, filtros em memória, `includeStats`, `ETag`/`If-None-Match` (`304`), `Cache-Control` e `503`. |
| `src/server/registerCollectorRoutes.ts` | Etapas 2B.2 / 2B.3 / 2B.4.3 | Novo | Registro modular das rotas legadas refatoradas sobre coletores, `/api/games/shadow` e `/api/games/central` (`GET`/`HEAD`/`OPTIONS`/`405`). |
| `server.ts` | Etapas 2A → 2B.4.3 | Modificado | Integração de `registerCollectorRoutes(app, { cbAPI })`, `getBzzoiroApiKey()`, `brtDate` e cabeçalhos CORS (`HEAD`, `If-None-Match`, `Access-Control-Expose-Headers`). |
| `tests/etapa2a.test.ts`, `etapa2b2.test.ts`, `etapa2b21.test.ts`, `etapa2b3.test.ts`, `etapa2b32.test.ts`, `etapa2b43.test.ts` | Etapas 2A → 2B.4.3 | Novos | Suíte automatizada de verificação e regressão do backend (não entram no bundle `dist/server.cjs`, recomendadas no repositório para CI). |

### Categoria B — Relacionadas a etapas futuras do Frontend / Dependência compartilhada
1. **Dependência compartilhada indispensável:** `src/utils/brtDate.ts` é importado tanto pelo backend (`server.ts`, `src/server/**`) quanto pelo frontend (`src/App.tsx`, `JogosDoDiaPage.tsx`, `Ticker.tsx`, `HeroCommandCenter.tsx`, `MeusTimesPage.tsx`, `cmsService.ts`, `sportsblazeService.ts`, `favoritesService.ts`). Como o backend depende dele diretamente na compilação de `dist/server.cjs`, **precisa estar presente no commit do backend**. Validado com `tsc --noEmit` e `vite build` sem erros em nenhum consumidor frontend.
2. **Consumidores e serviços do frontend (`src/components/**`, `src/services/**`):**
   - Não há nenhum import de `src/server/**` no frontend (`grep -rn "server/" src/` retornou 0 ocorrências).
   - `JogosDoDiaPage.tsx`, `Ticker.tsx`, `HeroCommandCenter.tsx`, `HeroCopa.tsx`, `HomeLite.tsx`, `MeusTimesPage.tsx`, `JogosDoDiaTexto.tsx` e `MatchStatsService` **não devem ser migrados nem alterados neste commit**.
   - Caso `git diff` no repositório real do usuário aponte edições antigas em `src/components/**` ou `src/services/**` feitas antes/durante a 2A que não sejam necessárias ao backend, **devem ser separadas** (não incluídas neste commit focado no backend), mantendo no commit apenas `src/utils/brtDate.ts` como utilitário compartilhado.

### Categoria C — Não relacionadas à centralização dos jogos (EXCLUÍDAS)
- **Scripts avulsos de correção/teste manual na raiz:** `fix_bsd_match.cjs`, `fix_bsd_match_2.cjs`, `fix_central.cjs`, `fix_central_overrides.cjs`, `fix_espn_weeks.cjs`, `fix_fetchsched.cjs`, `fix_jogostexto.cjs`, `fix_jogostexto2.cjs`, `fix_lax.cjs`, `fix_loadschedule_syntax.cjs`, `fix_loadscheduledata.cjs`, `fix_nfl_week1.cjs`, `fix_nfl_week1.js`, `fix_port.cjs`, `fix_sync.cjs`, `fix_ticker_time.cjs`, `fix_ui.cjs`, `update_logo.cjs`, `test_mock.js`, `test_server.cjs`, `test_server2.cjs`, `test_server3.cjs`.
- **Relatórios e artefatos de auditoria em `temp/`:** `etapa-2b-4-2-contrato-aprovado.md`, `etapa-2b-4-2-1-consolidacao-final.md`, `etapa-2b-4-3-implementacao-central.md`, `etapa-2b-5-1-auditoria-paridade-jogosdodia.md`, `etapa-2b-5-2-validacao-central-producao.md`, `etapa-2b-5-2-1-auditoria-commit-backend.md`.
- **Artefatos gerados / ignorados (`.gitignore`):** `dist/`, `node_modules/`, `.env*`.

---

## 3. Mapa de Dependências do Backend

### 3.1. Grafo exato de módulos compilados em `dist/server.cjs`
Rastreamento via `esbuild server.ts --bundle --platform=node --format=cjs --packages=external --metafile` (`409.6 kb`):

```text
server.ts (113.692 B)
 ├── src/utils/brtDate.ts (22.421 B)                      [compartilhado com frontend]
 ├── src/server/collectors/bzzoiroCollector.ts (90.220 B)
 │    └── src/utils/brtDate.ts
 └── src/server/registerCollectorRoutes.ts (28.428 B)
      ├── src/utils/brtDate.ts
      ├── src/server/collectors/bzzoiroCollector.ts
      ├── src/server/collectors/espnAmericanCollector.ts (22.269 B)
      │    └── src/utils/brtDate.ts
      ├── src/server/collectors/espnLacrosseCollector.ts (7.801 B)
      │    └── src/utils/brtDate.ts
      ├── src/server/collectors/unifiedGamesCollector.ts (124.581 B)
      │    ├── src/utils/brtDate.ts
      │    ├── src/utils/gameSets.ts (15.268 B)
      │    ├── src/server/collectors/bzzoiroCollector.ts
      │    ├── src/server/collectors/espnAmericanCollector.ts
      │    ├── src/server/collectors/espnLacrosseCollector.ts
      │    └── src/server/collectors/sportsblazeAmericanCollector.ts (24.603 B)
      │         └── src/utils/brtDate.ts
      └── src/server/centralGamesApi.ts (41.161 B)
           ├── src/utils/brtDate.ts
           └── src/server/collectors/unifiedGamesCollector.ts
```

Total exato no bundle do backend: **10 arquivos-fonte internos** (nenhum arquivo de `src/components/`, `src/services/` ou `src/data/` entra no bundle).

### 3.2. Dependências externas e leituras em disco em runtime
- **Módulos externos requeridos por `dist/server.cjs`:** `campeonato-brasileiro-api`, `express`, `firebase/app`, `firebase/firestore`, `vite` *(chamado apenas quando `NODE_ENV !== "production"`)* e built-ins Node (`fs`, `path`, `url`, `node:crypto`). Todos constam em `package.json -> dependencies` e resolveram sem erro via `require.resolve`.
- **Leituras opcionais de arquivos em disco pelo `server.ts` em runtime:**
  - `./firebase-applet-config.json` (fallback para env vars do Firebase — já presente no repo);
  - `./src/data/sportsData.json` e `./src/data/pllMatches.json` (fallback quando disponíveis no CWD — já presentes no repo);
  - `./espn-search/espn-cache/*.json` (catálogo de busca de favoritos — já presente no repo);
  - `./dist/index.html` (se existir após `npm run build`; caso contrário, serve o HTML `"Órbita Arena Backend"` em `app.get("*")` quando `NODE_ENV=production`).

### 3.3. Build, TypeScript e Start no Render
- **`tsconfig.json`:** `include: ["src/**/*", "server.ts"]`, `exclude: ["node_modules", "espn-search"]`, `noEmit: true`. `npm run lint` (`tsc --noEmit`) valida todo o backend + frontend com 0 erros.
- **`package.json` (`build` e `start`):**
  - `"build"`: `node copy-assets.cjs && vite build && npx esbuild server.ts --bundle --platform=node --format=cjs --packages=external --sourcemap --outfile=dist/server.cjs`
  - `"build:server"`: `node copy-assets.cjs && npx esbuild server.ts --bundle --platform=node --format=cjs --packages=external --sourcemap --outfile=dist/server.cjs`
  - `"start"`: `node dist/server.cjs`
- **Compatibilidade build ↔ start:** `npm run build` gera exatamente `dist/server.cjs` consumido por `npm run start` (`node dist/server.cjs`).
- **Observação de porta (`server.ts:57`):** `const PORT = 3000;` continua hardcoded (não alterar sem necessidade comprovada; o serviço `orbitaarena-v2.onrender.com` já opera normalmente em produção nessa configuração).

---

## 4. Lista Exata de Arquivos para o Commit do Backend

| # | Caminho | Novo / Modificado | Motivo da Inclusão | Etapa de Origem | Indispensável para Deploy? |
|---:|---|---|---|---|---|
| 1 | `server.ts` | Modificado | Registra `registerCollectorRoutes`, leitura de `BZZOIRO_API_KEY`, CORS (`HEAD`, `If-None-Match`, `Expose-Headers`) e preserva rotas legadas. | 2A / 2B.2 / 2B.4.3 | **Sim** |
| 2 | `src/server/centralGamesApi.ts` | Novo | Implementa o contrato `1.0` de `GET / HEAD /api/games/central`, filtros, `ETag`/`304`, `Cache-Control`, `includeStats` e `503`. | 2B.4.3 | **Sim** |
| 3 | `src/server/registerCollectorRoutes.ts` | Novo | Registra `/api/games/central`, `/api/games/shadow` e rotas legadas sobre os coletores sem alterar contratos antigos. | 2B.2 / 2B.3 / 2B.4.3 | **Sim** |
| 4 | `src/server/collectors/unifiedGamesCollector.ts` | Novo | Agrega provedores, reconcilia ESPN×SportsBlaze, mantém cache compartilhado Shadow+Central e fallback *stale* `180s`. | 2B.3 / 2B.3.2 / 2B.4.3 | **Sim** |
| 5 | `src/server/collectors/bzzoiroCollector.ts` | Novo | Coleta e normaliza jogos/tabelas/fixtures de futebol via Bzzoiro lendo `BZZOIRO_API_KEY` exclusivamente no servidor. | 2B.2 | **Sim** |
| 6 | `src/server/collectors/espnAmericanCollector.ts` | Novo | Coleta e normaliza NFL, MLB, NBA, WNBA, NHL e NCAAF com `espnLiveDetail`. | 2B.2 / 2B.3.2 | **Sim** |
| 7 | `src/server/collectors/espnLacrosseCollector.ts` | Novo | Coleta e normaliza Lacrosse PLL via ESPN. | 2B.2 | **Sim** |
| 8 | `src/server/collectors/sportsblazeAmericanCollector.ts` | Novo | Coleta NFL, MLB, NBA e WNBA via feed público `cache.sportsblaze.com` sem autenticação. | 2B.2.1 | **Sim** |
| 9 | `src/utils/brtDate.ts` | Modificado *(compartilhado)* | Padronização BRT (`America/Sao_Paulo`) e interpretação segura de `timestampSource` exigida por todos os coletores. | 2A | **Sim** |
| 10 | `src/utils/gameSets.ts` | Novo | Classificação canônica `TODAY`, `OVERNIGHT`, `TOMORROW` usada pelo agregador unificado e API Central. | 2A | **Sim** |
| 11 | `package.json` *(somente se houver diff real no Git)* | Verificar no `git diff` | Caso o `origin/main` ainda não tenha `build:server` / scripts atuais de compilação para `dist/server.cjs`. *(Se já idêntico no remoto, não haverá alteração).* | Pré-2B / Build | Condicional |
| 12 | `tests/etapa2a.test.ts` | Novo | Cobertura automatizada de `brtDate.ts` e `gameSets.ts`. | 2A | Não (recomendado CI) |
| 13 | `tests/etapa2b2.test.ts` | Novo | Cobertura dos coletores Bzzoiro e ESPN + rotas legadas. | 2B.2 | Não (recomendado CI) |
| 14 | `tests/etapa2b21.test.ts` | Novo | Cobertura do coletor SportsBlaze. | 2B.2.1 | Não (recomendado CI) |
| 15 | `tests/etapa2b3.test.ts` | Novo | Cobertura do agregador unificado e rota `/api/games/shadow`. | 2B.3 | Não (recomendado CI) |
| 16 | `tests/etapa2b32.test.ts` | Novo | Cobertura de proveniência ao vivo (`espn` > `sportsblaze`). | 2B.3.2 | Não (recomendado CI) |
| 17 | `tests/etapa2b43.test.ts` | Novo | Cobertura integral do contrato `GET / HEAD /api/games/central`. | 2B.4.3 | Não (recomendado CI) |

> **Observação sobre os itens 12–17 (`tests/*.test.ts`):** Recomenda-se incluir no commit apenas se os testes datados (`etapa2b2.test.ts:625` e `etapa2b21.test.ts:413`) não bloquearem pipeline de CI automática no Render; o Render executa apenas `npm run build` e `npm run start`, não executando `tests/*.test.ts` durante o deploy. Se houver dúvida e quiser commit estritamente mínimo para deploy, os **arquivos 1 a 10** (+ item 11 se alterado) são suficientes e autossuficientes para o backend em produção.

### Mensagem de commit sugerida
```text
feat(api): publish centralized games aggregation and /api/games/central endpoint
```
*(Ou em formato curto: `feat(api): publish centralized games aggregation`)*

---

## 5. Lista de Arquivos Excluídos do Commit

| Arquivo / Diretório | Categoria | Motivo da Exclusão |
|---|---|---|
| `src/components/JogosDoDiaPage.tsx` e demais componentes em `src/components/**` | B (Frontend futuro) | Migração do frontend só ocorrerá após validação da Central em produção (Etapa 2B.5). |
| `src/services/**` (`cmsService.ts`, `sportsblazeService.ts`, `matchStatsService.ts`, `favoritesService.ts`, etc.) | B (Frontend) | Nenhuma alteração necessária ao backend; `cmsService.ts` já consome `/api/bsd-proxy` sem chaves. |
| `fix_*.cjs`, `fix_*.js`, `update_logo.cjs` (18 arquivos na raiz) | C (Não relacionado) | Scripts históricos avulsos de patch em arquivos frontend/data/servidor; não são importados nem usados em build/runtime. |
| `test_server.cjs`, `test_server2.cjs`, `test_server3.cjs`, `test_mock.js` | C (Não relacionado) | Scripts manuais locais de teste HTTP antigo. |
| `temp/*.md` (`etapa-2b-4-2-*.md`, `etapa-2b-4-3-*.md`, `etapa-2b-5-*.md`) | C (Documentação de auditoria) | Artefatos temporários de análise/planejamento; evitar incluir desnecessariamente em commit de produção. |
| `dist/`, `node_modules/`, `.env*` | Build / Local | Ignorados por `.gitignore`; `dist/` é gerado no build do Render e `.env` nunca deve ser versionado. |
| `public/android-chrome-*.png`, `public/maskable_icon_x512.png` *(se modificados pelo `copy-assets.cjs`)* | C (Assets frontend) | Copiados em tempo de build a partir de `src/assets/`; não pertencem ao commit do backend se não houver alteração intencional de ícones PWA. |

---

## 6. Resultado dos Testes, Build e Verificações de Segurança

### 6.1. Lint e Build (executados em `21/11/2026`)
| Comando | Resultado | Observações |
|---|---|---|
| `npm run lint` (`tsc --noEmit`) | **PASSOU (`0` erros)** | Valida conjuntamente `server.ts` e `src/**/*` (backend + frontend). |
| `npm run build` | **PASSOU** | `copy-assets.cjs` OK; `vite build` concluído em `18.11s` (`5609` módulos); `esbuild server.ts` gerou `dist/server.cjs` (`409.6 kb`) em `64 ms` com 5 avisos esperados de `import.meta` em CJS (com fallback para `__filename`/`__dirname`). |
| Verificação do comando `start` (`node dist/server.cjs`) | **VALIDADO** | Todos os módulos externos presentes em `dist/server.cjs` (`express`, `firebase/app`, `firebase/firestore`, `campeonato-brasileiro-api`, `vite`, built-ins) resolveram corretamente via `require.resolve` (a execução direta simultânea na porta `3000` colidiria intencionalmente com o servidor dev já escutando em `0.0.0.0:3000`). |

### 6.2. Resultados das Suítes Automatizadas (`npx tsx --test`)
| Suíte | Passou / Total | Falhas | Classificação |
|---|---:|---:|---|
| `tests/etapa2a.test.ts` | **14 / 14** | 0 | Aprovada integralmente |
| `tests/etapa2b3.test.ts` | **9 / 9** | 0 | Aprovada integralmente |
| `tests/etapa2b32.test.ts` | **8 / 8** | 0 | Aprovada integralmente |
| `tests/etapa2b43.test.ts` | **7 / 7** | 0 | Aprovada integralmente |
| `tests/etapa2b2.test.ts` | **8 / 9** | **1** | **Pré-existente / dependente de data** (`tests/etapa2b2.test.ts:625`) |
| `tests/etapa2b21.test.ts` | **5 / 6** | **1** | **Pré-existente / dependente de data** (`tests/etapa2b21.test.ts:413`) |
| **Total geral** | **51 / 53** | **2** | **0 regressões** nas implementações de `src/server/**` e `src/utils/**` |

**Diagnóstico exato das 2 falhas pré-existentes (sem alteração de código nesta etapa):**
- Em `src/utils/gameSets.ts:244-249`, `classifyGameForSets(match, options)` espera um objeto `{ now?: Date; referenceDateBrt?: string }`.
- Os testes antigos `tests/etapa2b2.test.ts:617-624` (`nowMidDay`) e `tests/etapa2b21.test.ts:414-417` (`refNow`) passam uma instância `Date("2026-11-20T18:00:00-03:00")` diretamente como 2º argumento, fazendo `options?.now` ficar `undefined` e `gameSets.ts` usar a data real do relógio (`2026-11-21`). Por isso passaram no dia `20/11/2026` e falham a partir de `21/11/2026`. Não representam defeito no código de produção.

### 6.3. Checklist de Segurança e Preservação de Contratos
- **`BZZOIRO_API_KEY` exclusiva no backend:** Confirmado via varredura em `src/` e `server.ts`. Lida exclusivamente em `src/server/collectors/bzzoiroCollector.ts:35` (`process.env.BZZOIRO_API_KEY`) e consumida em `server.ts`. O cliente (`src/services/cmsService.ts:2181`) acessa Bzzoiro apenas via `/api/bsd-proxy`.
- **Ausência de segredos hardcoded:** Confirmado em todos os arquivos auditados; `tests/etapa2b3.test.ts` e `tests/etapa2b43.test.ts` comprovam que `bzzoiroApiKey` nunca vaza na resposta HTTP.
- **SportsBlaze sem autenticação:** `src/server/collectors/sportsblazeAmericanCollector.ts` consulta exclusivamente `https://cache.sportsblaze.com` sem API key nem header `Authorization`/`Bearer`.
- **Variáveis de ambiente:** Nenhuma variável de ambiente de produção foi alterada ou exposta.
- **Rotas legadas preservadas:** `/api/health`, `/api/football/today`, `/api/football/standings`, `/api/football/fixtures`, `/api/sportsblaze/boxscores`, `/api/sportsblaze/schedule`, `/api/sportsblaze/standings`, `/api/brasileirao/*`, `/api/jogos-texto`, `/api/match-stats`, `/api/bsd-proxy/*`, `/api/favorites/*`, `/api/sports/*` permanecem intactas.
- **Snapshot compartilhado Central + Shadow:** `GET /api/games/shadow` e `GET / HEAD /api/games/central` usam o mesmo mapa `unifiedSharedCache` e fila de *coalescing* `unifiedSharedInFlight` em `unifiedGamesCollector.ts`.

---

## 7. Riscos e Pendências

1. **Reconciliação com `origin/main` antes do `git add`:** Como este sandbox não contém `.git`, é obrigatório rodar `git status` e `git diff` no repositório real do projeto (GitHub/AI Studio sync) para garantir que alterações pré-existentes no remoto (ex.: eventuais ajustes em `server.ts` ou `package.json`) não sejam sobrescritas acidentalmente na transferência de arquivos.
2. **Efeito colateral na Vercel (`vercel-ignore-build.sh`):** O arquivo `vercel-ignore-build.sh` só cancela deploys na Vercel quando **exclusivamente** `src/data/sportsData.json` ou `src/data/mediaData.json` são alterados. Como o commit incluirá `server.ts` e `src/utils/brtDate.ts`, a Vercel também executará um build do frontend se monitorar a mesma branch — sem quebra funcional (comprovado pelo `vite build` verde), mas é importante estar ciente antes do push.
3. **Falhas datadas em `etapa2b2.test.ts` e `etapa2b21.test.ts`:** Se alguma pipeline automática rodar todos os testes em `tests/*.test.ts`, esses 2 testes falharão enquanto não forem ajustados para `{ now: ... }` em etapa própria; o deploy padrão do Render (`npm run build` + `npm run start`) não é afetado.
4. **Configuração do Render (`PORT` e `BZZOIRO_API_KEY`):** Manter `BZZOIRO_API_KEY` e `NODE_ENV=production` no serviço Render `orbitaarena-v2` e confirmar que `server.ts:57` (`PORT = 3000`) continua atendido pela configuração atual da instância.

---

## 8. Procedimento Recomendado para Publicação (Aguardando Autorização)

Quando o responsável aprovar o commit e o deploy no ambiente conectado ao GitHub:

1. **Inspeção inicial no repositório Git local/sincronizado:**
   ```bash
   git status
   git diff server.ts package.json src/utils/brtDate.ts
   ```
   Confirmar que não há alterações inesperadas em `src/components/**`, `src/services/**` ou sobrescrita de código pré-existente.
2. **Staging seletivo apenas dos arquivos aprovados (Categoria A):**
   ```bash
   git add server.ts \
     src/server/centralGamesApi.ts \
     src/server/registerCollectorRoutes.ts \
     src/server/collectors/bzzoiroCollector.ts \
     src/server/collectors/espnAmericanCollector.ts \
     src/server/collectors/espnLacrosseCollector.ts \
     src/server/collectors/sportsblazeAmericanCollector.ts \
     src/server/collectors/unifiedGamesCollector.ts \
     src/utils/brtDate.ts \
     src/utils/gameSets.ts
   ```
   *(Adicionar `package.json` somente se `git diff package.json` mostrar mudanças necessárias ao build; adicionar `tests/etapa2*.test.ts` caso deseje versionar a suíte junto).*
3. **Conferência do que ficou em stage e garantia de exclusão dos scripts/temp:**
   ```bash
   git diff --cached --stat
   ```
   Verificar que `fix_*.cjs`, `test_*.cjs`, `temp/*.md`, `.env*`, `dist/` e `src/components/JogosDoDiaPage.tsx` **não** estão na área de stage.
4. **Validação final pré-commit:**
   ```bash
   npm run lint
   npx tsx --test tests/etapa2a.test.ts tests/etapa2b3.test.ts tests/etapa2b32.test.ts tests/etapa2b43.test.ts
   npm run build
   ```
5. **Commit e Push (após aprovação explícita):**
   ```bash
   git commit -m "feat(api): publish centralized games aggregation"
   git push origin <branch-do-render>
   ```
6. **Validação pós-deploy (retomando Etapa 2B.5.2):**
   - Aguardar conclusão do deploy no Render (`https://orbitaarena-v2.onrender.com`).
   - Validar `GET /api/games/central?date=<YYYY-MM-DD>` retornando `HTTP 200`, `Content-Type: application/json`, `degraded: false`, `cache.stale: false`, `HEAD`, `OPTIONS 204`, `POST 405`, `304 If-None-Match`, `400 INVALID_DATE` e paridade contemporânea com `/jogosdodia` antes de iniciar a migração do frontend na Etapa 2B.5.
