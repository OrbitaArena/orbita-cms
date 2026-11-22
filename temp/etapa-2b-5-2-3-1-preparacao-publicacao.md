# ÓRBITA ARENA — ETAPA 2B.5.2.3.1
## Consolidação das Alterações e Preparação da Publicação do Backend

**Data:** domingo, 22/11/2026  
**Escopo:** Auditoria final do workspace, correção pontual de assinatura nos testes legados (`etapa2b2.test.ts` e `etapa2b21.test.ts`), validação de integridade de build/bundle/runtime e preparação da relação exata de publicação do backend no Render.  
**Restrições respeitadas:** Nenhum `commit`, `push` ou `deploy` executado; nenhuma alteração no frontend (`JogosDoDiaPage.tsx`, `src/App.tsx`, `src/services/*`), contratos da API, `.env` ou segredos; implementação de produção de `src/utils/gameSets.ts` intacta.

---

## 1. Arquivos Novos e Modificados (Auditoria Completa do Backend)

> **Contexto do repositório neste sandbox:** O ambiente `/app/applet` não possui diretório `.git` (`git status` retorna `fatal: not a git repository`). Por isso, a auditoria combina a inspeção direta do grafo de compilação do backend (`esbuild` sobre `server.ts`), timestamps do workspace e o histórico consolidado da Etapa 2B.5.2.1 e Etapa 2B.5.2.3.

### 1.1 Delta específico desta entrega (Etapa 2B.5.2.3 + 2B.5.2.3.1 — ainda não publicado no Render)
| Arquivo | Ação / Etapa | Papel na Publicação |
|---|---|---|
| `src/server/collectors/bzzoiroCollector.ts` | Modificado (2B.5.2.3) | Elimina `fallback_resilience` em erro frio (`502 bzzoiro_upstream_error`), detecta truncamento de paginação/live, separa `footballLastCompleteCache` (`30 min`) de `footballCache` (`180s`), aplica `fetchWithTimeout` em metadados/broadcasts/incidents/standings e protege `EVENT_BROADCASTS_CACHE`. |
| `src/server/collectors/unifiedGamesCollector.ts` | Modificado (2B.5.2.3) | Propaga degradação, truncamento (`materialPartial`), fallback isolado do Bzzoiro com `collectedAt`/`sourceCacheAgeMs` originais e falha parcial de enriquecimento para o snapshot compartilhado. |
| `src/server/centralGamesApi.ts` | Modificado (2B.5.2.3) | Deduplica `warnings` e projeta `provider_bzzoiro_partial` / `provider_bzzoiro_enrichment_partial` / `provider_bzzoiro_stale_fallback` preservando o contrato `1.0`. |
| `tests/etapa2b523.test.ts` | Novo (2B.5.2.3) | Suíte determinística com 14 blocos cobrindo os 18 cenários de integridade, paginação, caches, stale isolado/global, rotas legadas e `matchStatsId`. |
| `tests/etapa2b2.test.ts` | Modificado (2B.5.2.3.1) | Ajuste pontual na chamada de `classifyGameForSets` (`nowMidDay` → `{ now: nowMidDay }`). |
| `tests/etapa2b21.test.ts` | Modificado (2B.5.2.3.1) | Ajuste pontual nas duas chamadas de `classifyGameForSets` (`refNow` → `{ now: refNow }`). |

### 1.2 Conjunto completo dos módulos de runtime do Backend (Etapas 2A → 2B.5.2.3)
Verificado pelo metafile real do `esbuild` sobre `server.ts` (`dist/server.cjs`, `444.938 B` / `~434.5 KiB`):

| # | Arquivo | Origem / Etapa | Tipo | Comprovado no Bundle? |
|---:|---|---|---|---|
| 1 | `server.ts` | 2A / 2B.2 / 2B.4.3 | Modificado | Sim (entry point) |
| 2 | `src/server/centralGamesApi.ts` | 2B.4.3 / 2B.5.2.3 | Novo / Modificado | Sim |
| 3 | `src/server/registerCollectorRoutes.ts` | 2B.2 / 2B.3 / 2B.4.3 | Novo | Sim |
| 4 | `src/server/collectors/unifiedGamesCollector.ts` | 2B.3 / 2B.3.2 / 2B.4.3 / 2B.5.2.3 | Novo / Modificado | Sim |
| 5 | `src/server/collectors/bzzoiroCollector.ts` | 2B.2 / 2B.5.2.3 | Novo / Modificado | Sim |
| 6 | `src/server/collectors/espnAmericanCollector.ts` | 2B.2 / 2B.3.2 | Novo | Sim |
| 7 | `src/server/collectors/espnLacrosseCollector.ts` | 2B.2 | Novo | Sim |
| 8 | `src/server/collectors/sportsblazeAmericanCollector.ts` | 2B.2.1 | Novo | Sim |
| 9 | `src/utils/brtDate.ts` | 2A *(compartilhado)* | Modificado | Sim |
| 10 | `src/utils/gameSets.ts` | 2A | Novo | Sim |

- **Dependência compartilhada justificada:** `src/utils/brtDate.ts` é importado por todos os módulos do servidor e também pelo frontend; não importa nada de `src/server/` e compila limpo tanto no `vite build` quanto no `esbuild`.
- **Isolamento do frontend confirmado:** `grep` em `src/components/`, `src/services/` e `src/App.tsx` confirmou **0** imports vindos de `src/server/`.
- **`package.json` e `package-lock.json`:** Nenhuma dependência externa foi adicionada nas etapas 2B.4.3, 2B.5.2.3 ou 2B.5.2.3.1. Todas as dependências externas do bundle (`express`, `vite`, `firebase/app`, `firebase/firestore`, `campeonato-brasileiro-api`, além dos built-ins Node `fs`, `path`, `url`, `node:crypto`) já constam em `package.json -> dependencies` e resolvem sem erro. Só incluir `package.json` / lockfile se `git diff` no repositório real do usuário apontar diferença prévia não comitada.
- **Compatibilidade `build` ↔ `start`:** `npm run build` gera `dist/server.cjs` e `npm run start` executa exatamente `node dist/server.cjs`.

---

## 2. Correção dos Testes Legados

Foi realizada varredura global por `classifyGameForSets` em `tests/` e `src/`, confirmando que `tests/etapa2a.test.ts` já utilizava `{ now, referenceDateBrt }` e que apenas **3 chamadas** em **2 arquivos** usavam o formato antigo (passando um `Date` bruto como 2º argumento, fazendo `options?.now` ficar `undefined` e cair no relógio do sistema):

1. **`tests/etapa2b2.test.ts` (linha 623 — teste `2B.2.7`):**
   - Antes: `classifyGameForSets({ ... }, nowMidDay)`
   - Depois: `classifyGameForSets({ ... }, { now: nowMidDay })`
2. **`tests/etapa2b21.test.ts` (linhas 416 e 427 — teste `2B.2.1.4`):**
   - Antes: `classifyGameForSets({ ... }, refNow)`
   - Depois: `classifyGameForSets({ ... }, { now: refNow })`

A implementação de produção de `src/utils/gameSets.ts` **não foi modificada** e a intenção original dos testes (classificação em `2026-11-20T18:00:00-03:00` para `TODAY` e `TOMORROW`) foi preservada integralmente.

---

## 3. Resultados de Testes, Lint e Build

| Verificação | Comando | Resultado | Cobertura |
|---|---|---|---|
| **TypeScript Lint** | `npm run lint` (`tsc --noEmit`) | **PASSOU (0 erros)** | Todo `server.ts` e `src/**/*` |
| **Build de Produção** | `npm run build` (`copy-assets.cjs` + `vite build` + `esbuild server.ts`) | **PASSOU (0 erros)** | Client + `dist/server.cjs` (`444.938 B`) |
| **Etapa 2A** | `npx tsx --test tests/etapa2a.test.ts` | **14 / 14 PASSARAM** | `brtDate.ts` e `gameSets.ts` |
| **Etapa 2B.2** | `npx tsx --test tests/etapa2b2.test.ts` | **9 / 9 PASSARAM** | Bzzoiro, ESPN American/Lacrosse, rotas legadas |
| **Etapa 2B.2.1** | `npx tsx --test tests/etapa2b21.test.ts` | **6 / 6 PASSARAM** | SportsBlaze collector |
| **Etapa 2B.3** | `npx tsx --test tests/etapa2b3.test.ts` | **9 / 9 PASSARAM** | Agregador unificado e `/api/games/shadow` |
| **Etapa 2B.3.2** | `npx tsx --test tests/etapa2b32.test.ts` | **8 / 8 PASSARAM** | Proveniência ao vivo (`espn` > `sportsblaze`) |
| **Etapa 2B.4.3** | `npx tsx --test tests/etapa2b43.test.ts` | **7 / 7 PASSARAM** | Contrato `1.0` de `GET`/`HEAD /api/games/central` |
| **Etapa 2B.5.2.3** | `npx tsx --test tests/etapa2b523.test.ts` | **14 / 14 PASSARAM** | Todos os 18 cenários de integridade e resiliência Bzzoiro |
| **Total Geral das Suítes** | `npx tsx --test tests/etapa2a.test.ts ... tests/etapa2b523.test.ts` | **67 / 67 PASSARAM (100%)** | **0 falhas, 0 skipped, 0 cancelled** |

---

## 4. Arquivos que Serão Incluídos no Commit do Backend

### 4.1 Delta mínimo caso o `origin/main` já contenha a Etapa 2B.4.3 publicada
*(Obrigatório para levar a correção 2B.5.2.3 e os testes corrigidos ao Render):*
- `src/server/collectors/bzzoiroCollector.ts`
- `src/server/collectors/unifiedGamesCollector.ts`
- `src/server/centralGamesApi.ts`
- `tests/etapa2b523.test.ts`
- `tests/etapa2b2.test.ts`
- `tests/etapa2b21.test.ts`

### 4.2 Relação completa do Backend + Suíte (caso existam pendências de 2A → 2B.4.3 na branch de publicação)
**Código de runtime (10 arquivos obrigatórios no bundle `dist/server.cjs`):**
1. `server.ts`
2. `src/server/centralGamesApi.ts`
3. `src/server/registerCollectorRoutes.ts`
4. `src/server/collectors/unifiedGamesCollector.ts`
5. `src/server/collectors/bzzoiroCollector.ts`
6. `src/server/collectors/espnAmericanCollector.ts`
7. `src/server/collectors/espnLacrosseCollector.ts`
8. `src/server/collectors/sportsblazeAmericanCollector.ts`
9. `src/utils/brtDate.ts` *(compartilhado; necessário para compilar `server.ts`)*
10. `src/utils/gameSets.ts`

**Suíte automatizada de regressão do backend (7 arquivos):**
11. `tests/etapa2a.test.ts`
12. `tests/etapa2b2.test.ts`
13. `tests/etapa2b21.test.ts`
14. `tests/etapa2b3.test.ts`
15. `tests/etapa2b32.test.ts`
16. `tests/etapa2b43.test.ts`
17. `tests/etapa2b523.test.ts`

**Condicional (apenas se `git diff` real mostrar alteração necessária no build):**
- `package.json` / `package-lock.json` *(verificar antes via `git diff package.json package-lock.json`; nenhuma alteração foi feita na 2B.5.2.3 ou 2B.5.2.3.1).*

---

## 5. Arquivos Excluídos do Commit

| Arquivo / Padrão | Motivo da Exclusão |
|---|---|
| `.env*` (se existir localmente) e credenciais/segredos | `.env*` é ignorado por `.gitignore`; apenas `.env.example` existe como modelo sem segredos. `BZZOIRO_API_KEY` continua lida exclusivamente via `process.env.BZZOIRO_API_KEY`. |
| `src/components/**` (especialmente `JogosDoDiaPage.tsx`), `src/App.tsx`, `src/services/**` | Migração do frontend será realizada em etapa posterior; não misturar alterações visuais ou de consumo client-side neste commit do backend. |
| `fix_*.cjs`, `fix_*.js`, `update_logo.cjs`, `test_mock.js`, `test_server*.cjs` (22 scripts na raiz) | Scripts históricos avulsos fora do build e fora do runtime do servidor. |
| `temp/*.md` (`etapa-2b-*.md`) | Relatórios internos de auditoria e planejamento; não necessários no deploy de produção. |
| `dist/`, `node_modules/`, `public/android-chrome-*.png`, `public/maskable_icon_x512.png` | Artefatos gerados em tempo de build (`dist/` no `.gitignore`; ícones copiados pelo `copy-assets.cjs`). |

---

## 6. Riscos Residuais

1. **Conferência de `git diff` no repositório sincronizado com o GitHub:** Como este sandbox não contém `.git`, antes de executar `git add` no ambiente conectado ao GitHub recomenda-se rodar `git status` e `git diff` para confirmar exatamente quais arquivos dos itens 1–17 ainda diferem do remoto e evitar sobrescrever edições de outras branches.
2. **Trigger simultâneo da Vercel (`vercel-ignore-build.sh`):** Como o script da Vercel só cancela o build quando exclusivamente `src/data/sportsData.json` ou `src/data/mediaData.json` são alterados, o push deste commit no repositório também acionará build na Vercel caso ela observe a mesma branch — sem impacto funcional no frontend (confirmado pelo `vite build` e `tsc --noEmit` com 0 erros).
3. **Variáveis de ambiente no Render (`orbitaarena-v2`):** Confirmar que `BZZOIRO_API_KEY` e `NODE_ENV=production` permanecem configuradas no serviço Render antes/após o deploy.

---

## 7. Mensagem Sugerida para o Commit

**Se for o commit focado na correção Bzzoiro + consolidação de testes (2B.5.2.3 + 2B.5.2.3.1 sobre 2B.4.3 já publicada):**
```text
fix(api): prevent silent Bzzoiro failures/truncation and fix legacy gameSets test calls
```

**Se consolidar todo o pacote pendente do backend (2A → 2B.5.2.3.1) em um único commit:**
```text
feat(api): harden Bzzoiro collector integrity and prepare centralized games backend release
```

---

**Status:** Pronto para sua revisão e aprovação antes da publicação (`git commit` / `push` / deploy no Render).
