# Suíte Local — Arquitetura e Planejamento

> **Documento mestre da suíte.** Retrato em **2026-07-06** — versões, portas e pendências mudam rápido; confira o repo de cada app antes de assumir um detalhe como atual. (Título original: "Suíte Taylor"; a marca guarda-chuva virou **Local** em 2026-07.)
>
> Workspace (desde 2026-07-12): `C:\Users\Hades\OneDrive\Desktop\Dev\Local` (repo `Anon5T4R/Office`) — os 10 apps são **submodules** registrados no `.gitmodules`, cada um com repo próprio em `github.com/Anon5T4R/<App>`.
>
> Portas de dev do Vite (únicas por app desde 2026-07-12): LocalOffice 1420 · LocalCode 1422 · TaylorHub 1425 · TaylorChat 1427 · LocalData 1430 · LocalPDF 1432 · LocalSheets 1434 · LocalSlides 1436 · LocalAI 1438 · LocalZIM 1440 (HMR = porta+1).

---

## 1. Visão geral e princípios

Suíte de aplicativos desktop **100% offline**, com **IA local** (modelos GGUF via llama.cpp), sob a marca guarda-chuva **"Taylor"** (aparece como easter egg nos ícones). Conta GitHub: **Anon5T4R**.

**Filosofia (João):**
- Tudo vira **OpenSource na v1**. Nada proprietário; reaproveitar OSS livremente — **copyleft (GPL/AGPL) não é problema** (reafirmado 2026-07-13): se o reaproveitamento valer a pena, o app adota a licença. Só lembrar a mão única do §6.4 (copyleft não entra em app MIT sem relicenciar).
- Monetização: **instalação facilitada** ("ter pronto pra usar") + **personalização/adaptação para empresas com necessidades específicas** + **modelos GGUF próprios** — nunca o software em si.
- Sync é responsabilidade do usuário (pasta no Syncthing/OneDrive); o app nem sabe que existe.

**Stack padrão (apps da pasta Office):**
- **Tauri 2 + React 19 + Vite + TypeScript** (front) + Rust (back).
- IA: **llama.cpp (`llama-server`) como sidecar** gerenciado pelo Rust, só `127.0.0.1`, API OpenAI-compat, zero telemetria. Build **Vulkan com fallback CPU**. Binários gitignored; rebaixados por `scripts/fetch-llama.{ps1,sh}`.
- Distribuição: **Windows NSIS + Linux AppImage** (writer também deb) via `tauri-action`; tag `v*` dispara o release no GitHub Actions.
- Hardware alvo modesto (AMD iGPU ~2GB VRAM, 17.8GB RAM): default CPU (`-ngl 0`), contexto ~4096, modelos pequenos.

**Regra de independência:** os apps são **INDEPENDENTES** — cada um tem repo, Tauri e sidecar próprios. "Reaproveitar do Writer/Sheets" = **portar/adaptar código manualmente**, não dependência compartilhada. (Um eventual pacote comum "taylor-ai-core" é ideia registrada, não realidade.)

**Gotchas de suíte (lições pagas):**
- Tauri v2 renomeia args de comando multi-palavra para **camelCase no `invoke`** (`modelPath`, `ctxSize`, `nGpuLayers`) — snake_case dá "missing required key".
- `resolve.dedupe:['react','react-dom']` no `vite.config.ts` (senão `@tiptap/react/menus` puxa 2ª cópia do React → "Invalid hook call").
- **OneDrive pode corromper/conflitar o repo** — conferir estado antes de editar.
- Regra "**zip sempre no webview**" (JSZip/PptxGenJS), Rust só move bytes (`read/write_file_base64`) — evita dor com AppImage e mantém a lógica em TS.
- `tauri.conf.json` com glob de resources não pode ficar vazio — manter `placeholder.txt` versionado em `binaries/llama/`.
- Checar exit code de `cargo | tail` mascara falha do cargo — redirecionar pra arquivo e checar `$?`.
- **`onCloseRequested` (Tauri v2) transfere o fechamento pro JS** (o wrapper chama `window.destroy()`), e `core:window:default` só tem getters — sem `core:window:allow-destroy` na capability, **o X da janela para de funcionar** (e `setTitle` exige `allow-set-title`). Lição paga no LocalPDF v0.3.2.

**Mapa de portas da IA (estado atual — ver §6 sobre a colisão):**

| App | Porta |
|---|---|
| Writer | **8088 fixa** |
| Code | 8090–8099 (dinâmica) |
| Sheets | 8099–8148 (dinâmica) |
| Slides | 8100–8120 (preferência 8100) |
| LocalData | 8101–8121 (preferência 8101) |
| LocalPDF | 8102–8122 (preferência 8102) |
| TaylorChat | 8103–8123 (preferência 8103) |

---

## 2. Apps ativos na pasta Office

### 2.1 writer/ — LocalOffice (documentos) — v0.14.6

- **Repo:** https://github.com/Anon5T4R/LocalOffice — **Licença: AGPL-3.0-or-later** (⚠️ único não-MIT da suíte, ver §6).
- **Stack:** TipTap/ProseMirror 3.27 (13 extensões) + **pandoc embarcado** (sidecar, DOCX/ODT/RTF↔HTML) + **paged.js** (PDF paginado WYSIWYG) + citeproc + KaTeX + marked/turndown (MD nativo).
- **Formatos:** Markdown, HTML, DOCX, ODT, RTF + export PDF. Associações: `md, markdown, txt, html, htm, docx, odt`.
- **Funcionalidades principais:**
  - Editor rico: ribbon estilo Word, menu slash `/` estilo Notion, tabelas, imagens redimensionáveis/flutuantes, busca/substituição, pincel de formatação, zoom, abas múltiplas.
  - **Acadêmico completo:** numeração automática de títulos (1, 1.1…), sumário clicável, notas de rodapé, **citações offline via `.bib` do Zotero** (citeproc: ABNT/APA/Chicago/IEEE, autocomplete `[@chave]`), bibliografia automática, equações KaTeX, modelos ABNT (NBR 14724)/APA/relatório/carta/artigo.
  - **Páginas reais no editor** (quebras, formatos A4/A5/Carta/A3 + rolagem infinita "Clássico", margens presets, ghost pages) — convergência editor↔PDF por construção.
  - **Revisão:** comentários + controle de alterações (aceitar/rejeitar) com **round-trip DOCX nativo**; versões nomeadas locais; autosave debounced (~2s, 4s p/ pandoc, backstop 60s).
  - Corretor ortográfico (pt-BR, pt-PT, en, es, fr).
- **IA (porta 8088 fixa, `src-tauri/src/llm.rs`):** chat streaming; ações na seleção via bubble menu (reescrever/revisar/resumir/traduzir/tom/continuar); "Resumir documento" via **map-reduce** (`chunkDocument`); motor no hook `useLocalAi`.
- **Estrutura src/:** `ai/`, `editor/` (extensões/marks/nodes), `components/`, `lib/` (pandoc, citações, export), `state/` (zustand), `hooks/`.
- **Pendências conhecidas:** legendas (v7.6), referências cruzadas (v7.7), KaTeX→OMML (v7.5), colunas (backlog). Q&A sobre doc gigante precisaria de RAG (não feito — ver §6).

### 2.2 sheets/ — LocalSheets (planilha) — v0.4.0

- **Repo:** https://github.com/Anon5T4R/LocalSheets — MIT.
- **Stack:** **jspreadsheet-ce v5** (grade; `main.tsx` SEM StrictMode, senão duplica a grade) + **SheetJS/xlsx** (arquivos) + **@formulajs/formulajs** (fórmulas avançadas) + `material-icons` offline.
- **Formatos:** XLSX (todas as abas, **fórmulas persistem** como `cell.f` + valor em cache, compatível Excel) e CSV (aba ativa, valores calculados). Associações: `csv, xlsx`.
- **Funcionalidades:** múltiplas abas, barra de fórmulas estilo Excel (name-box + fx), temas claro/escuro, grid mínimo 26×100 (trim ao salvar), confirmar ao fechar sujo.
- **IA (porta 8099–8148, `find_free_port`):** `SheetAiPanel` manda a planilha como grid A1 no system prompt (`sheetToContext`, até 40×20); modelo devolve ```` ```json [{"cell":"B2","value":"=A2*2"}] ````; `parseEdits` valida e `applyEdits` aplica via `ws.setValue` — **a IA edita células de verdade, inclusive fórmulas**. Think OFF por padrão (`chat_template_kwargs:{enable_thinking:false}`).
- **Motor de fórmulas (`src/lib/formula-fix.ts`):** o "Formula Basic" do CE achata ranges → quebrava lookups. Fix: hook `onbeforeformula` intercepta e computa com formula.js lendo a fórmula ORIGINAL. Cobre VLOOKUP/HLOOKUP/XLOOKUP/LOOKUP/INDEX/MATCH/SUMIF(S)/COUNTIF(S)/AVERAGEIF(S)/MAXIFS/MINIFS/SUMPRODUCT (+aninhadas tipo INDEX(MATCH())).
  - **Limitação:** só quando a célula é UMA chamada top-level dessas funções (`=SUMIF(...)+1` cai no motor básico).
  - **Não tem:** tabelas dinâmicas (pivô), gráficos, arrays dinâmicos (spill), formatação condicional, macros.
- **Estrutura src/:** `ai/`, `lib/` (ai, sheet-codec, sheet-io, formula-fix, settings), `Spreadsheet.tsx`, `App.tsx`.

### 2.3 slides/ — LocalSlides (apresentações) — v0.5.1

- **Repo:** https://github.com/Anon5T4R/LocalSlides — MIT. Design detalhado original: memória `arquitetura-localslides` (sessões do Claude).
- **Princípio:** **canvas posicional** (não fluxo de texto): slide 1280×720 (16:9) ou 960×720 (4:3) em px lógicos; cada elemento tem `x,y,w,h,rotation`; zoom = `transform:scale`. Fronteira PPTX: 1px@96dpi = 9525 EMU.
- **Stack:** TipTap 3 nas caixas (mount-on-edit) + **Zustand+immer** (undo/redo por snapshot; `beginTx/endTx` coalescem drag) + JSZip + **PptxGenJS** (export) + html-to-image (PNG) + **onnxruntime-web** (remoção de fundo de imagem) + 13 famílias @fontsource embutidas.
- **Formato nativo:** **`.tslides`** (zip: `deck.json` + `media/`, assets dedupados, `deck.assets` = biblioteca de mídia). Autosave + recuperação (localStorage p/ decks nunca salvos). Associação: `tslides`.
- **Elementos:** textbox, imagem (drag-and-drop, crop estilo Canva com pan/zoom), vídeo (toca no present), formas (14+ tipos SVG, estilos de traço sólido/tracejado/giz/esfumaçado), tabelas (células TipTap), linhas, desenho à mão (InkEl), comentários (pins editor-only), grupos (groupId flat).
- **Organização/edição:** camadas (painel + z-order), opacidade, contorno (segue silhueta alpha em PNG sem fundo), flip H/V, alinhar/distribuir, snapping com guias (imã 6px), multi-seleção/marquee, painel direito colapsável (Inspector/IA/Mídia/Camadas).
- **Temas e layouts:** 9+ presets de tema (cores como tokens re-tematizáveis), 7 master layouts com placeholders, templates de deck prontos, brand kit.
- **Apresentação:** fullscreen com transições (2 camadas, in+out) e animações de entrada; presenter window.
- **Export:** PDF (print view, 1 slide/página, `@page` no tamanho do deck), PNG/JPEG (rasteriza off-screen), **PPTX** (PptxGenJS; lossy: PM runs achatados, ink não exporta). **Import PPTX** (JSZip+DOMParser, OOXML→modelo; best-effort: temas caem em default, shapes exóticas viram retângulo).
- **IA (porta 8100–8120):** `deckgen.ts` — **"gera apresentação sobre X"** → modelo devolve JSON `{slides:[{layout:title|section|bullets, title, subtitle?, bullets?}]}`, `parseDeckSpec` valida, `specToSlides` monta slides reais via comandos do store (**undo grátis**). Nunca aplica geometria/HTML cru do modelo. + chat livre.
- **Pendências:** exportar ink→PPTX como linhas (TODO pedido), UI de notas do apresentador (modelo já tem `Slide.notes`), parsear tema do PPTX no import, charts/SmartArt no import (ignorados), text-wrap ao redor de imagem (**adiado p/ eventual remake** — briga com o modelo posicional).

### 2.4 code/ — LocalCode (editor de código) — v0.7.1

- **Repo local** `code/` — MIT. (Feito quase sozinho pelo João.)
- **Stack:** **Monaco Editor** + xterm.js/portable-pty (terminal PTY real) + **git2** (libgit2) + **octocrab** (GitHub API) + `lsp-types` + keyring (token no cofre do SO: DPAPI/Secret Service).
- **Funcionalidades:**
  - **LSP** zero-config: TS, Rust, Python, Go, YAML, CSS, HTML, JSON — com **instalador 1-clique** (`lsp-setup/`).
  - **Debugger DAP** (F5): Python (debugpy), JS/Node (js-debug), Rust/C-C++ (CodeLLDB) — breakpoints, variáveis, call stack, console.
  - Git completo (stage/commit/push/pull/branches/histórico) + GitHub (OAuth device flow, listar/criar repos, clonar, PRs).
  - Explorer, busca full-text (regex, respeita .gitignore), outline por símbolos, command palette, **sistema de extensões** (spec em `EXTENSIONS.md`/`EXTENDING.md`).
- **IA (porta 8090–8099 dinâmica):** chat + **agente tool-using** (`src/ai/agent.ts`) — cria/edita/lê arquivos e roda comandos no terminal. É o app com a IA mais "agêntica" da suíte.
- **Associações:** `js, ts, tsx, jsx, rs, json, md, html, css, toml, yaml, yml`.
- **Nota de CI:** LSP/DAP (~500MB) baixados no workflow, não versionados; Linux precisa de Secret Service ativo pro token OAuth.

---

## 3. Apps existentes FORA da pasta (registro — já existem, NÃO criar)

### 3.1 TaylorMind — mapa mental (`Desktop\TaylorMind`)

- Clone de XMind/SimpleMind com IA. **Electron 29 + React + TS** (não Tauri).
- IA: modelos GGUF **locais via node-llama-cpp** (sem LM Studio) OU provedores remotos (Anthropic, OpenAI, Gemini, OpenAI-compat) com API key própria — único app da casa com opção remota.
- Canvas de mapa mental (layout equilibrado 4 lados, redimensionar, cores, recolher, undo/redo); IA gera mapa de tema/texto, expande (1 nível/profundo), explica, edita bloco; chat lateral com inserção como nó/sub-árvore.
- Formato `.tmind`; export PNG/Markdown/HTML. Release por tag no GH Actions (Win + Linux).

### 3.2 OpenObsidian — notas/base de conhecimento (`Desktop\OpenObsidian`) — v0.7.1

- Clone do Obsidian. **Electron + React + CodeMirror 6 + zustand** (não Tauri).
- Lê PDF/EPUB (epubjs)/DOCX (mammoth), converte DOCX↔MD (mammoth/turndown/remark), exporta PDF/HTML, **grafo** (d3), mermaid, busca (flexsearch), watcher (chokidar).
- IA local via **node-llama-cpp**.
- **Versão Android** em repo próprio: https://github.com/Anon5T4R/OpenObsidianAndroid (releases lá). Patreon: https://www.patreon.com/cw/joaoferreirav
- Cobre o slot "notas/PKM" da suíte — **não fazer app novo de notas**; melhorias vão nele.

> Os dois são herança Electron/node-llama-cpp e ficam assim — **sem plano de migração** pra Tauri/sidecar. O Hub (§5) só precisa saber instalar/associar.

---

## 4. Apps planejados (ainda não iniciados)

### 4.1 LocalData / "Taylor Base" — dados estruturados — **IMPLEMENTADO (2026-07-07, `data/` v0.1.0)**

Airtable/Access local. **Por que não é o Sheets:** planilha = células livres; banco = registro tipado com relações e múltiplas visões — forçar isso no jspreadsheet luta contra a natureza da planilha. Sheets = Excel (cálculo); este = Access/Airtable (dados).

**Estado (v0.3.0 committada; releases v0.1.0 e v0.2.0 publicadas; no catálogo do Hub desde o Hub v0.3.0):** repo https://github.com/Anon5T4R/LocalData (MIT, branch main; gitlink `data/`). As fases 1–7 do plano original saíram juntas na v0.1.0; a v0.2.0 (mesmo dia) poliu: comandos Tauri async (fora da main thread), paginação de registros (500/página em background), undo/redo de registros com restauração de IDs originais, navegação por teclado na grade, drag pra reordenar colunas/tabelas e menu de contexto de linha. A **v0.3.0 (2026-07-07)** fechou os gaps vs Airtable: tipos novos (avaliação/URL/e-mail/telefone; número com formato moeda/inteiro/percentual), **lookup/rollup via relações** (computados no front como as fórmulas), **agrupamento colapsável na grade**, **seleção retangular + copiar/colar TSV** (colar cria registros e opções de select que faltam; um passo de undo via updates/inserts em lote), rodapé de agregação por coluna, cor de linha por select, coluna primária sticky, duplicar tabela/campo/view (ids remapeados no backend), descrição de campo, seletor de pasta NATIVO pros modelos GGUF no painel de IA e um passe geral de polimento no front (transições, scrollbars, animações, empty states).

- **Armazenamento:** **rusqlite (bundled)**, um arquivo **`.tbase`** por base (associação registrada) — é um SQLite legítimo, abre em qualquer ferramenta; também abre `.db`/`.sqlite` alheios (só adiciona as tabelas de metadados). Schema rico em `_taylor_tables/_taylor_fields/_taylor_views/_taylor_blobs` + tabela SQL real `t_<id>` por tabela, coluna `c_<id>` por campo (**IDs estáveis: renomear nunca gera DDL**). `schema_version` em `_taylor_meta` pronto pra migrations.
- **Tipos de campo:** texto, texto longo, número, checkbox, data (ISO, ordena como texto), select, multi-select, **relação** (array de ids em JSON — N:N sem join table), **anexo** (blob dentro do `.tbase` + GC ao fechar) e **fórmula** (avaliada no front: `{Campo}`, aritmética, IF/AND/OR/CONCAT/UPPER/ROUND/TODAY/DAYS…). Mudança de tipo converte valores (texto→select auto-cria as opções).
- **Views persistidas** (config JSON na base): grade virtualizada com edição inline/resize/ocultar colunas, kanban (drag entre colunas de select), calendário mensal, galeria (capa = anexo), formulário. Filtros/ordenação/busca **sempre em SQL parametrizado** montado no Rust.
- **Import CSV/XLSX** (SheetJS no webview, Rust só move bytes) com inferência de tipos → tabela nova; **export XLSX/CSV**.
- **IA (porta 8101–8121):** ops `createTable/createField/insert/update/setFilter` em JSON, **validadas contra o schema em TS** → comandos parametrizados; select por NOME com auto-criação de opção; **nunca SQL cru** (regra mantida). Sidecar llama.cpp padrão da suíte.
- Testes: 8 unit Rust (validação/filtros/conversão/duplicação) + 26 vitest (fórmula/inferência/clipboard). CI check+testes (ubuntu) + release NSIS/AppImage.

**v0.4.0 (2026-07-07, tag + release):** **extensões — tipos de campo plugáveis**: `.js` na pasta de extensões (config dir) roda na inicialização com a API `localdata.registerFieldType({id,name,icon,description,placeholder,multiline,parse,format,color})`; sem sandbox de propósito (código local do próprio usuário/dev, sem loja). No banco é sempre tipo `custom` = TEXT com coluna real — a camada SQL não depende de código de terceiro; remover extensão não perde dados. Menu 🧩 no TopBar (tipos carregados, erros, abrir pasta via opener escopo $APPCONFIG, recarregar a quente); exemplo `exemplo-cpf.js` auto-instalado na 1ª execução. E **backup automático ao abrir**: cópia pra `<config>/backups` antes de abrir, retenção POR BASE configurável na tela inicial (desligado/10/50/N, default 10, teto 500; hash do caminho distingue bases homônimas), melhor esforço (nunca bloqueia a abertura), botão abre a pasta.

**v0.5.0 (2026-07-07, tag + release) — "pronto pra empresa média" (itens 1–9 pedidos pelo João):**
- **Modo servidor multiusuário (LAN):** `src-tauri/src/server.rs` (tiny_http, 4 threads) hospeda a base ABERTA na GUI; todo acesso serializa no mesmo Mutex do SQLite (zero escrita concorrente no arquivo — `Db` virou `Arc<Mutex<>>` clonável). Usuários/senhas na própria base (`_taylor_users`, argon2) + papéis leitor<editor<admin e override por tabela (`_taylor_perms`). O servidor injeta o `actor` em cada mutação (auditoria confiável). Front: transporte `call` em backend.ts roteia `invoke` (local) vs `fetch` (remoto) sem duplicar wrappers (`remote.ts`); StartScreen conecta a um host; `ServerPanel` administra usuários/servir; polling `changes_since` (~2s) reflete edições dos colegas. **Headless `--serve` ficou pra 0.6** (exige refatorar comandos pra `&Db`; o servidor hospedado pela GUI já entrega o multiusuário).
- **Auditoria** (`_taylor_audit`): trilha append-only quem/quando/ação/antes→depois na MESMA transação de cada mudança (lote >200 vira resumo). UI: histórico geral (TopBar 🕘) e por registro (no modal).
- **Integridade + validação** (no Rust, dentro da transação): relação `onDelete` restrict/unlink (sem ids órfãos); constraints por campo único/obrigatório/regex/min-máx.
- **Import com upsert** (`upsertImport` + `ImportModal`): reimporta planilha PARA a tabela ativa casando por campo-chave.
- **Escala:** `records_aggregate` faz COUNT/SUM/AVG/MIN/MAX no SQL (rodapé independe das linhas carregadas); PAGE_SIZE 1000; teste de 100k (insert 264ms/página 36ms/agregação 22ms).
- **Relatório imprimível** (`ReportModal` + `@media print`) → PDF pela impressora do SO, zero deps.
- **Automações** (`automations.ts` + `_taylor_automations`): "ao criar / campo vira valor" → notificar (plugin-notification, `{Campo}` interpolado) ou definir campo. Um nível, sem cascata; motor no store pós-mutação.
- Deps novas: `tiny_http`, `argon2`, `regex`, `tauri-plugin-notification`. Testes: 14 Rust + 26 vitest verdes.

**Pendências anotadas:** teste GUI completo (grade/kanban/IA/multiusuário 2 máquinas/automações) na máquina do João; headless `--serve` (0.6); grupos com drag entre eles; lookup aninhado fora de escopo. **Empresa média: 1–9 ✅** — o que sobra é polimento e o headless.

### 4.2 LocalPDF / "Taylor PDF" — editor de PDF — **PUBLICADO (2026-07-11, v0.3.1; repo + releases + Hub v0.3.8)**

Ir além da leitura: anotar, preencher formulários, mesclar/dividir/reordenar, assinar, OCR — e **editar texto existente com re-fluxo** (santo graal, fase final).

**Estado (2026-07-10): v0.1.0 e v0.2.0 no repo https://github.com/Anon5T4R/LocalPDF (MIT, releases Windows NSIS + AppImage via Actions); no catálogo do TaylorHub v0.3.8 (10º app, associação `.pdf`).** Tudo implementado e verificado no navegador (vite dev + PDF sintético + OCR real lendo páginas renderizadas):
- **Viewer** pdf.js: zoom (presets + Ctrl+roda + ajustar), miniaturas com drag pra reordenar, render preguiçoso (IntersectionObserver), "ir para página", **camada de texto selecionável** (copiar + botão flutuante que vira a seleção em realces precisos) e **busca Ctrl+F** (varre texto posicionado + palavras do OCR, navega e pisca o trecho).
- **Páginas** via pdf-lib: mesclar, extrair, reordenar, rotacionar, excluir, inserir em branco; undo/redo por snapshot de bytes (cap 12).
- **Anotações** queimadas no salvar: realce (multiply), texto, desenho, **imagem/carimbo** e **assinatura desenhada** (modal, recorte no traço, reuso via localStorage, arrastar/redimensionar); conversão viewport↔PDF cobre as 4 rotações (`coords.ts`, unit-testada).
- **Edição de texto existente (beta)**: clique numa linha (agrupamento de itens do pdf.js por baseline), reescreve; tarja branca + redesenho. Re-fluxo de parágrafo continua sendo a fase final.
- **AcroForms**: listar/preencher (+`updateFieldAppearances`).
- **OCR offline** (tesseract.js 7, por/eng tessdata_fast, ~26MB em `public/tesseract` via `scripts/fetch-tessdata` + CI): reconhece páginas escaneadas, alimenta busca/IA e **grava texto invisível** (opacity 0) — PDF vira pesquisável em qualquer leitor. Testado de ponta a ponta (leu as páginas de teste com precisão perfeita).
- **IA local porta 8102–8122**: resumo map-reduce + Q&A com recuperação lexical (`chunks.ts`); RAG com embeddings fica pro módulo comum §6.2.
- **Front**: tema claro/escuro/sistema (tokens CSS + data-theme), topbar em 2 linhas, StartScreen com chips de features. 43 testes vitest; front build + cargo check verdes.
- **v0.3.1 (2026-07-11):** caixa de texto consertada pro uso REAL — o default de foco do mousedown matava o textarea recém-criado (caixa parecia não abrir) e trocar de caixa perdia o texto digitado; virou **editor controlado** (cada tecla sincroniza no store), preventDefault segura o foco, clique fora com a ferramenta ativa fecha a edição, clicar numa caixa existente reabre, caixa vazia é descartada sem sujar o undo, e a caixa ganhou **largura com quebra de linha** (resize do editor → `TextAnnot.w` → `drawText maxWidth`). Lição de suíte: bug de foco NÃO aparece com `dispatchEvent` sintético — testar com clique real ou simulando o default do mousedown.
- **v0.3.0 (2026-07-10):** undo/redo (Ctrl+Z/Y/Shift+Z) cobre TODAS as anotações (criar/mover/redimensionar/editar/excluir; arrasto coalesce; seleção→realce e edição de linha = 1 entrada; fast-path sem reparse quando só annots mudam); **girar remapeia as anotações pendentes** (`rotateAnnots`, ida-e-volta testada); caixa de texto com **fonte** (Helvetica/Times/Courier — padrão do PDF) e tamanho, cor/fonte/tamanho/espessura aplicam na annot selecionada e refletem os valores dela; **Sumário (capítulos)** — aba na sidebar lê o outline e navega; **front responsivo** (rótulos→ícones em janela estreita, miniaturas fluidas, painel lateral vira overlay ≤900px, scrollbars/focus-visible, título da janela com arquivo + ●sujo); atalhos Esc/Ctrl+=/−; fix da IA cega pro OCR (cache). O Hub NÃO precisou de release novo: resolve `releases/latest` na instalação/atualização.
- **Caveats:** `copyPages` (reordenar/extrair/mesclar) não reconstrói o AcroForm; texto queimado = Helvetica WinAnsi (pt-BR OK); salvar achata as anotações; edição de linha cobre visualmente (o texto original continua extraível por baixo da tarja).
- **Lições novas de suíte (pagas aqui):** (1) selector zustand devolvendo `[]`/`{}` novo a cada render = loop infinito "getSnapshot should be cached" — usar constante estável; (2) OCR embarcado: copiar só as variantes *-lstm* do tesseract.js-core (as "full" dobram os ~26MB).
- **Falta:** teste GUI real (`tauri dev`) e IA com `.gguf` na máquina do João; re-fluxo de parágrafo (santo graal); assinatura criptográfica (a atual é visual); OCR em mais idiomas.

### 4.3 LocalDraw — diagramas (a avaliar)

Visio/draw.io local. **Custo-benefício alto:** o canvas do Slides já entrega ~70% (formas SVG, snapping, alinhar/distribuir, grupos, camadas, export PNG/PDF). Novidade principal = **conectores entre formas** (linhas que seguem o elemento ao mover) + formas de fluxograma + export SVG. Decidir depois de LocalData/LocalPDF; se sair, é porte do slides/.

### 4.4 Taylor Chat — mensageria P2P — **PUBLICADO (v0.1.11)** · design em [plano.md](plano.md), roadmap em [taylorchat-roadmap.md](taylorchat-roadmap.md)

Substitui a ideia de e-mail (descartada). **Mensageiro P2P sem servidor**: mensagens, arquivos comprimidos/retomáveis e mídia direto entre pares, E2E, sem cadastro/telefone. Nome **TaylorChat** (foge do padrão `Local*` de propósito — decisão do João 2026-07-07); pasta `chat/`, repo `Anon5T4R/TaylorChat`, MIT; no catálogo do TaylorHub. **Todas as fases do MVP feitas + evolução até v0.1.11** (releases Windows/Linux com `--features p2p`): identidade ed25519 no keyring, SQLite cifrado em repouso, pareamento por convite/QR, double ratchet vodozemac (X3DH + forward secrecy), rede iroh (**mDNS LAN + n0**), anexos retomáveis, stickers, multichat, auditoria, palavra-chave anti-MITM, IA local (só propõe), i18n pt/es/en, bandeja, **lista de contatos**, **canal quente com heartbeat ping/pong + presença em tempo real**. **Revisão vs WhatsApp Desktop (2026-07-09): 62/100** de prontidão pra uso diário — o que falta (com destaque pro furo P0 de **notificações de desktop**) está na checklist do [taylorchat-roadmap.md](taylorchat-roadmap.md). Pendência operacional: **teste ponta a ponta com 2 PCs**.

As perguntas em aberto abaixo foram **fechadas no [plano.md](plano.md)**: transporte **iroh** (traz blobs resumíveis + gossip de graça), cripto **QUIC/TLS + double ratchet `vodozemac`**, relays públicos do n0 OK (NAT-traversal, não é infra nossa), histórico **rusqlite cifrado**, IA na **porta 8103**, MVP 1-device (multi-device e store-and-forward são fases futuras). Esboço original preservado:
- **Transporte P2P:** avaliar **iroh** (Rust, QUIC, hole-punching, direto ao ponto) vs **libp2p** (mais ecossistema, mais complexo). Ambos Rust/OSS — casam com Tauri.
- **Criptografia ponta a ponta** obrigatória (o que a lib der; ideal double ratchet ou Noise).
- **Descoberta/pareamento:** QR code / código de convite (troca de chaves fora de banda) — sem cadastro, sem número de telefone.
- **Entrega:** inicialmente só quando ambos online (sem relay próprio = sem servidor = coerente com a filosofia). Avaliar depois relays públicos/opcionais ou store-and-forward entre os próprios contatos.
- **Arquivos/mídia:** compressão (**zstd**) antes do envio, transferência retomável em chunks.
- **Histórico:** local em SQLite (cifrado em repouso, se viável).
- **Sem data.** Registrado aqui só pra não perder a intenção.

### 4.5 LocalZIM — leitor de bibliotecas ZIM / clone do Kiwix — **PUBLICADO (2026-07-12, v0.2.0; Hub v0.4.0)**

Lê arquivos `.zim` (Wikipédia offline, Stack Overflow, Gutenberg — library.kiwix.org). Repo https://github.com/Anon5T4R/LocalZIM (MIT, branch main; README credita o Kiwix e aponta o zimit pra criar `.zim`). Pasta `LocalZIM/`, porta dev 1440, identifier `com.localzim.app`, associação `.zim` (11º app do catálogo do Hub desde o v0.4.0).

- **Parser ZIM em Rust puro** (`src-tauri/src/zim.rs` + testes com ZIM sintético byte a byte): header, dirents, busca binária por URL/título, clusters zstd/LZMA2-XZ/raw (cache LRU 12; raw lê só a fatia do blob — vídeo grande não entra na RAM), redirects, metadados M, esquemas de namespace antigo (A/I/M) e novo (C/M/W/X), índice `X/listing/titleOrdered/v1`. **Sem libzim/C++** — zero dor no CI Windows.
- **Protocolo `zim://`** (no Windows `http://zim.localhost/`) serve `/<id>/<N>/<url>` direto ao WebView2; links relativos de graça, redirect vira 302, **Range → 206** (vídeo/áudio com seek). Em todo HTML o Rust injeta `bridge.js` (postMessage): título/URL → app, links externos → navegador do sistema, comandos de zoom/modo escuro/localizar/atalhos.
- **Busca full-text (v0.2.0, `search.rs`)**: o Xapian embutido no ZIM não é lido (C++); o app constrói índice **tantivy** próprio 1x em background (progresso/cancelamento via evento `fulltext`), guardado em `app_data/fulltext/<uuid do ZIM>`; BM25 com boost de título, tokenização lowercase+sem acentos, corpo NÃO armazenado (trecho re-extraído do ZIM na busca). UI: item "buscar no texto completo" no dropdown + painel lateral com trechos destacados.
- **Front**: biblioteca (recentes em localStorage com ícone/descrição do próprio ZIM) + leitor em iframe: voltar/avançar, página principal, artigo aleatório, sugestões por título (prefixo + variantes de capitalização), zoom por livro, localizar na página (Ctrl+F via `window.find` na ponte), tema claro/escuro dentro do artigo, atalhos que atravessam o iframe. Single-instance + arquivo por argumento de CLI.
- **Criar .zim de pasta (v0.3.0, `zimwriter.rs`; v0.4.1 corrigiu pro esquema NOVO de namespaces)**: empacota pasta com HTML num ZIM válido (tudo no namespace C, minor 1 — página e CSS/imagem no mesmo namespace, senão link relativo dá 404; lição paga no rust-br.github.io. Clusters zstd pra texto e crus streaming pra mídia, título extraído de cada página, metadados M, favicon/Illustration, listas ordenadas, md5 no rodapé; arquivos >4 GiB rejeitados — offsets u32). Modal na biblioteca com progresso/cancelamento (evento `zim-create`); ao terminar já abre o arquivo. Round-trip testado (cria → reabre com o próprio parser). Rastrear site da internet continua sendo papel do zimit (documentado na UI e no README); o caminho rápido local é `wget --mirror` + empacotar aqui.
- **Criar .zim de um site (v0.4.0, `crawler.rs`)**: crawler estático local — BFS mesmo host (profundidade/máx. páginas configuráveis, robots.txt respeitado, delay educado), assets de qualquer host em `_ext/`, segue `url()`/`@import` de CSS, reescreve links pra relativos em 2ª fase (externos ficam absolutos → abrem no navegador via ponte), staging em temp → `zimwriter`. HTML via lol_html (extração e reescrita, inclusive srcset e `<style>` inline; `<base>` removido). Teste de integração com servidor HTTP local nos testes. Limite honesto: SPA/JS sai incompleto → zimit (dito na UI).
- **Limitações anotadas no README**: índice full-text é opt-in e ocupa disco; histórico compartilhado ao alternar livros abertos.
- Pendência: teste com um ZIM real grande na máquina do João (inclusive indexação full-text).

---

## 5. Hub — TaylorHub (DECISÃO 2026-07-06: começar JÁ; plano detalhado em [hub.md](hub.md))

**Hub bem simples** — instalador unificado + registrador de associações, que faz o arquivo certo abrir no app certo ao clicar. Sem launcher/dock elaborado.

A condição do João ("só começo agora se ficar fácil adicionar apps novos") foi atendida: o Hub é **dirigido por catálogo** (`catalog.json` embutido + remoto) — app novo = 1 entrada JSON, zero código. LocalData/LocalPDF/Chat entram no catálogo quando tiverem release. Pré-requisito verificado: todos os 6 apps existentes têm release no GitHub com instalador NSIS silenciável (`/S`). Arquitetura, fases e decisões: [hub.md](hub.md).

**Escopo:**
1. Instalador (NSIS no Windows; avaliar equivalente Linux) que baixa/instala os apps selecionados (checkboxes).
2. Registra associações de extensão → app.
3. Talvez uma tela mínima "abrir/criar arquivo" — opcional, não é o objetivo.

**Tabela de associação proposta:**

| Extensões | App |
|---|---|
| md, markdown, txt, docx, odt, rtf, html | Writer |
| xlsx, csv | Sheets |
| tslides, pptx | Slides |
| js, ts, tsx, jsx, rs, json, toml, yaml, yml, css… | Code |
| tmind | TaylorMind |
| (vault/pasta md) | OpenObsidian |
| pdf | LocalPDF (quando existir) |
| db (bases Taylor) | LocalData (quando existir) |

**Conflitos a resolver no Hub:** hoje Writer e Code disputam `md`, `html` (e Code registra `md` também). Proposta: documento (md/html/docx) → Writer por padrão; o Hub decide e os apps individuais param de brigar pela associação quando instalados via Hub.

---

## 6. Itens transversais

### 6.1 Runtime de IA compartilhado (ordem decidida: DEPOIS do Hub)

Hoje cada app embarca o próprio llama.cpp e sobe o próprio servidor → com 2–3 apps abertos, duplica binário em disco e **modelo em RAM** (o recurso mais escasso da máquina alvo). A ideia: **um serviço local único** (llama-server compartilhado + pasta única de modelos) que todos os apps consomem; é também onde a **venda de modelos GGUF** se encaixa (gerenciador/loja de modelos).

- Só faz sentido **pós-Hub** (o Hub é quem instala o serviço; sem Hub, cada app precisa continuar autônomo).
- Requisito: apps continuam funcionando standalone (fallback pro sidecar próprio se o serviço não existir).
- **Portas — DECIDIDO (2026-07-06): deixar como está.** Writer 8088 fixa; Code 8090–8099; Sheets 8099–8148; Slides 8100–8120. As faixas se sobrepõem, mas todos buscam porta livre e funciona; não mexer. O runtime compartilhado, quando vier, mata o assunto (uma porta só).

### 6.2 RAG local reutilizável

Q&A sobre documento grande aparece no LocalPDF (§4.2) e seria útil no Writer e no OpenObsidian. Extrair um módulo comum (extração de texto → chunks → **embeddings via llama.cpp** → índice local → busca) em vez de refazer por app. Nasce junto com o LocalPDF fase 5.

### 6.3 Interoperabilidade entre apps — BAIXA prioridade

Decisão do João: interessante, mas **trabalhoso e bugável** — não perseguir. Registro do caminho barato caso um dia valha: clipboard rico já dá muito de graça (HTML de tabela do Sheets cola em Writer/Slides; PM JSON entre Writer↔Slides seria colável com pouco código). Nada de OLE/embedding vivo.

### 6.4 Licenças — DECIDIDO (2026-07-06): misto MIT/AGPL é ok

O Writer está **AGPL-3.0-or-later**, o resto (sheets/slides/code/TaylorMind) é **MIT** — e está tudo bem assim. Decisão do João: AGPL não é problema quando viabiliza usar/copiar código aberto (libs/soluções AGPL entram sem medo no app que precisar delas, que então fica AGPL).

**Única regra prática a lembrar:** a compatibilidade é de mão única — código **MIT pode entrar em app AGPL**, mas código **AGPL NÃO pode ser copiado pra app MIT** (o app MIT viraria AGPL). Como a suíte porta código manualmente entre apps: portar de sheets/slides/code → writer é livre; portar do writer → um app MIT exige reescrever ou relicenciar o app de destino.

### 6.5 Identidade visual e branding

- Ícone por app: mesma linguagem, cor por domínio (documento azul, planilha verde, slides azul→ciano…) + **"Taylor" cursivo** como easter egg.
- Fonte dos ícones: `src-tauri/icons/source-*.svg` → `tauri icon` (aceita SVG).
- Nomes públicos: LocalOffice, LocalSheets, LocalSlides, LocalCode, (LocalData, LocalPDF, LocalDraw?) — marca Taylor por cima no Hub.

### 6.6 CI/Release padrão

- Workflow por repo: tag `v*` → matriz Windows (NSIS) + Ubuntu 22.04 (AppImage; writer também deb) via `tauri-action`.
- Sidecars baixados no CI por `scripts/fetch-*.{ps1,sh}` (llama.cpp win-vulkan-x64.zip / ubuntu tar.gz; pandoc no writer; LSP/DAP ~500MB no code).
- Instaladores **não assinados** (SmartScreen avisa) — custo de assinatura fica pra quando houver receita.
- Release body padrão: lembrar que **modelos GGUF não vão no instalador** (usuário aponta a pasta).

---

## 7. Ordem de execução (resumo — atualizada 2026-07-06)

1. **Hub / TaylorHub** (§5, [hub.md](hub.md)) — antecipado: por ser catálogo-driven, não depende dos apps futuros; eles entram depois com 1 entrada JSON.
2. ~~**LocalData** (§4.1)~~ **FEITO (2026-07-07, v0.1.0)** → ~~**LocalPDF** (§4.2)~~ **FEITO (2026-07-10, v0.1.0 — falta release/Hub)** → avaliar **LocalDraw** (§4.3). Pendências dos apps atuais (§2) em paralelo, são refinos.
3. **Runtime de IA compartilhado** (§6.1) — pós-Hub. RAG comum (§6.2) pode nascer antes, dentro do LocalPDF.
4. **Taylor Chat P2P** (§4.4) — futuro, sem data.
5. **Próximos apps (decidido 2026-07-13):** ~~LocalAgenda~~ **FEITO (v0.1.0)** → ~~LocalScribe~~ **FEITO (v0.1.0)** → ~~LocalMedia~~ **FEITO (v0.1.0)** → LocalImage → LocalPlayer → LocalDraw → LocalTranslate → LocalKeys — plano detalhado (stack, portas, fases, riscos) em [proximos-apps.md](proximos-apps.md).

E-mail: **fora** (decidido 2026-07-06). Interop: baixa prioridade (§6.3).
