# Próximos apps da suíte Local — plano detalhado

> Criado em 2026-07-13 a pedido do João. Complementa o [projetos.md](projetos.md) (documento mestre) — as convenções de suíte (release por tag, sidecar llama.cpp, NSIS+AppImage, catálogo do Hub) valem aqui e não são repetidas em cada app.
>
> Ordem de execução definida pelo João: **LocalAgenda → LocalScribe → LocalMedia → LocalImage → LocalPlayer → LocalDraw → LocalTranslate → LocalKeys**.
> (Na lista original o LocalScribe apareceu em 2º e em 9º — é o mesmo app, tratado uma vez só.)

---

## 0. Convenções comuns (reservas feitas neste plano)

**Stack padrão (igual à suíte):** Tauri 2 + React 19 + Vite + TypeScript no front, Rust no back. IA local (quando houver) = llama.cpp `llama-server` sidecar, só `127.0.0.1`, OpenAI-compat, binário gitignored baixado por `scripts/fetch-llama.{ps1,sh}`. Hardware alvo modesto (default CPU `-ngl 0`, contexto ~4096).

**Regra de independência mantida:** cada app tem repo, Tauri e sidecar próprios; "reaproveitar do X" = portar código manualmente.

| App | Pasta | Repo | Identifier | Porta dev (HMR = +1) | IA llama | Sidecar extra | Associações | Categoria no Hub |
|---|---|---|---|---|---|---|---|---|
| LocalAgenda | `agenda/` | `Anon5T4R/LocalAgenda` | `com.localagenda.app` | **1442** | 8104–8124 | — | `ics` | Escritório |
| LocalScribe | `scribe/` | `Anon5T4R/LocalScribe` | `com.localscribe.app` | **1444** | 8105–8125 | whisper.cpp (**8130–8150**) | — | Inteligência artificial |
| LocalMedia | `media/` | `Anon5T4R/LocalMedia` | `com.localmedia.app` | **1446** | — | ffmpeg (CLI, sem porta) | — | **Mídia** (categoria nova) |
| LocalImage | `image/` | `Anon5T4R/LocalImage` | `com.localimage.app` | **1448** | — | — | `png, jpg, jpeg, webp, gif, bmp` (opcional, ver §4) | Mídia |
| LocalPlayer | `player/` | `Anon5T4R/LocalPlayer` | `com.localplayer.app` | **1450** | — | libmpv (lib, não porta) | `mp4, mkv, webm, avi, mov, mp3, flac` | Mídia |
| LocalDraw | `draw/` | `Anon5T4R/LocalDraw` | `com.localdraw.app` | **1452** | 8106–8126 | — | `tdraw` | Escritório |
| LocalTranslate | `translate/` | `Anon5T4R/LocalTranslate` | `com.localtranslate.app` | **1454** | — (candle in-process) | — | — | Inteligência artificial |
| LocalKeys | `keys/` | `Anon5T4R/LocalKeys` | `com.localkeys.app` | **1456** | — (**sem IA de propósito**) | — | `tkeys` | **Segurança** (categoria nova) |

- Portas dev seguem a sequência existente (última usada: LocalZIM 1440). Tauri não tem fallback de porta — `devUrl` e `vite.config.ts` têm que bater.
- Faixas de IA seguem o padrão `find_free_port` com preferência (última usada: TaylorChat 8103). Whisper ganhou faixa própria (8130+) pra nunca colidir com os llama-server.
- Licença: **MIT por padrão, mas copyleft não é problema** (decisão do João 2026-07-13: todo repo público é open source; se reaproveitar código GPL/AGPL valer a pena, o app adota a licença — como o Writer já é AGPL). A regra prática do §6.4 do projetos.md continua: código copyleft não pode ser portado *para* um app MIT sem relicenciar o destino.
- Cada app entra no catálogo do TaylorHub com 1 entrada (lembrar: release nova do Hub enquanto o catálogo for embutido). Categorias novas propostas: **Mídia** e **Segurança**.
- CI: `ci.yml` (testes front+Rust a cada push) + `release.yml` (tag `v*` → NSIS Windows + AppImage Linux via tauri-action), como LocalZIM/TaylorHub. Validação sempre pelo Actions, nunca `cargo build` local (OneDrive).
- Fora deste plano: **LocalFeed** (leitor RSS — João pediu explicação antes de decidir; registrado no fim) e **e-mail** (decidido fora em 2026-07-06).

---

## 1. LocalAgenda — calendário, tarefas e lembretes — **IMPLEMENTADO (2026-07-13, v0.1.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalAgenda (MIT, público), pasta `LocalAgenda/` (submodule; a pasta segue o padrão `LocalX` da suíte, não `agenda/`), porta dev **1442**, identifier `com.localagenda.app`, associação `.ics`. No catálogo do TaylorHub (categoria Escritório) desde o commit do `catalog.ts` — **falta a release nova do Hub** (esperando a 1ª release do LocalAgenda). A v0.1 saiu completa e verificada no front (tsc/vite verdes, 12 testes vitest de recorrência+ICS). Decisões de implementação que fugiram/ajustaram o plano:
- **Recorrência com `rrule.js` (npm) no front**, não a crate `rrule` do Rust — RFC 5545 idêntico (Google/Outlook), mas evita uma dependência Rust não testável localmente e casa com o padrão da suíte de manter lógica no TS. Usa o truque de espelhar a hora local em UTC (floating time) pra DST/fuso não escorregarem.
- **ICS no webview** (parse/serialize em TS; Rust só move bytes) — mesma filosofia do "zip sempre no webview".
- **Lembretes:** o front materializa linhas concretas (`fireAt` epoch-ms, IDs determinísticos `kind:refId:occ:min`) numa tabela `reminders`; um **tick de 30s no Rust** dispara os vencidos (`INSERT OR IGNORE` + flag `fired` ⇒ cada ocorrência notifica no máximo 1×). Sem chrono/rrule/icalendar no Rust — dependências mínimas (só rusqlite+serde+tauri), CI de baixo risco.
- **Bandeja:** fechar minimiza (padrão, configurável `closeToTray`); "Sair" pela bandeja encerra. **Autostart opcional** via `tauri-plugin-autostart` (opt-in).
- **IA (porta 8104):** linguagem natural → evento (a IA propõe, o código valida em TS e abre o editor pré-preenchido pro usuário confirmar) + resumo da semana. Padrão deckgen do Slides.
- **Notificação inteligente extra:** resumo do dia (opcional, no horário escolhido) + snooze (5/10/30/60 min) direto no toast in-app.

Feito: mês/semana/dia/agenda com arrastar-pra-criar, tarefas com subtarefas/prioridade/recorrência (rola pro próximo prazo ao concluir), múltiplos calendários coloridos, busca, tema claro/escuro, atalhos (T/setas/m/w/d/a/n), backup export/import da base. **Releases publicadas:** LocalAgenda **v0.1.1** (v0.1.0 = 1ª release; v0.1.1 = ícone no padrão da suíte — card de calendário + "Taylor" cursivo) e **TaylorHub v0.6.2** (LocalAgenda no catálogo, categoria Escritório). **Pendências:** teste GUI real (`tauri dev`) + IA com `.gguf` na máquina do João; edição de ocorrência única (só "pular ocorrência" implementado, exceção via exdate).

**O gap:** a suíte não tem PIM. É o "Outlook sem e-mail": calendário + tarefas + lembretes, 100% local.

**Stack específica:**
- **Armazenamento:** rusqlite (bundled), **um DB único em `app_data`** (agenda não é documento — não faz sentido "abrir arquivo agenda"). Schema versionado (`schema_version` como no LocalData). Export/import da base inteira pra backup.
- **Recorrência:** crate **`rrule`** (RFC 5545 — a mesma semântica do Google Calendar/Outlook). Nunca inventar motor de recorrência próprio.
- **ICS:** crate **`icalendar`** para export; import de `.ics` (evento único e feed exportado de outro app). Associação `ics` registrada.
- **Notificações:** `tauri-plugin-notification` (já usado no LocalData). **Bandeja + minimizar pra tray** (como TaylorChat) — lembrete só funciona com o app vivo; opcional `tauri-plugin-autostart` (desligado por padrão, o usuário liga em Configurações).

**Modelo de dados:** `calendars` (cor, nome) → `events` (título, início/fim, all-day, rrule, exdates, lembretes N minutos antes, descrição, local) + `tasks` (título, notas, due date opcional, prioridade, recorrência simples, concluída_em, subtarefas por parent_id). Lembretes = tabela `reminders` materializada pelas próximas ocorrências (job no Rust a cada minuto, tick barato).

**Funcionalidades por fase:**
- **v0.1 (MVP):** visão mês + semana + lista ("agenda"); criar/editar/arrastar evento; tarefas em painel lateral com checkbox e due date; lembrete por notificação; tray; tema claro/escuro; atalhos (T = hoje, setas = navegar).
- **v0.2:** recorrência completa (rrule + exceções "só esta ocorrência"); import/export ICS; múltiplos calendários com cores; busca.
- **v0.3:** visão dia com grade horária; tarefas recorrentes; "adiar" lembrete (snooze); duração padrão configurável; primeiro dia da semana.
- **v0.4 (IA, porta 8104):** entrada em linguagem natural — "dentista quinta 15h lembra 1h antes" → JSON `{title, start, end, reminder}` validado em TS antes de criar (mesmo padrão do deckgen do Slides: **a IA propõe, o código valida e aplica via store — undo grátis**). Resumo da semana em texto.

**Reuso da suíte:** view de calendário do LocalData como referência de layout; zustand+immer com undo por snapshot (Slides); padrão de settings do Sheets.

**Riscos:** timezone — regra: **tudo em hora local, sem TZ no MVP** (app local pra uso pessoal; TZ só se um dia houver sync). All-day guardado como data pura pra não escorregar um dia.

---

## 2. LocalScribe — transcrição de áudio offline (whisper.cpp) — **IMPLEMENTADO (2026-07-13, v0.1.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalScribe (MIT), pasta `LocalScribe/` (submodule), porta dev **1444**, identifier `com.localscribe.app`, IA llama na 8105–8125. No catálogo do TaylorHub (Inteligência artificial) desde o commit v0.7.0 do Hub. **As fases v0.1–v0.3 saíram juntas na v0.1.0**: fila com progresso/cancelamento, player sincronizado com waveform clicável, editor de segmentos mantendo timestamps, biblioteca SQLite com busca, export TXT/MD/SRT/VTT, gravação de microfone (cpal) com medidor de nível, tela de modelos (ggml tiny/base/small/medium + .en baixados do HF com SHA-1 conferido do models/README.md do whisper.cpp) e IA local (resumo/ata/tópicos/pergunta com map-reduce). Decisões que fugiram/ajustaram o plano:
- **`whisper-cli` por arquivo em vez do `whisper-server`**: o CLI dá progresso real (`-pp` no stderr), cancelamento limpo (kill) e JSON estruturado (`-oj`) — três coisas que o server não dá por requisição; o custo (recarregar modelo, ~1-2 s) é irrelevante. A faixa 8130–8150 fica reservada se o server voltar um dia.
- **O WAV 16 kHz convertido é a fonte do player** (asset protocol, escopo `$APPDATA`): WAV toca em qualquer webview; formato original não precisa tocar. Opt-out em "guardar áudio" (~2 MB/min).
- Linux do fetch-whisper **compila estático da última release** (os binários prontos do projeto podem exigir glibc mais novo que o runner/usuário); Windows usa o zip `whisper-bin-x64`.
- Sem associação de extensão (como planejado); entrada por drag-drop/diálogo/microfone.

**Pendências:** teste GUI real (`tauri dev` + modelo real) na máquina do João; diarização segue fora de escopo; tradução via LocalTranslate no futuro.

**Stack específica:**
- **Motor:** **whisper.cpp `whisper-server`** como sidecar (endpoint `/inference`), porta 8130–8150 com `find_free_port`. Binários baixados por `scripts/fetch-whisper.{ps1,sh}` (release oficial do whisper.cpp: zip Windows x64; Linux compila no CI ou usa build estático). Fallback: se o server não subir, usar `whisper-cli` por arquivo (mesmo binário zip).
- **Modelos:** ggml (`tiny`, `base`, `small` multilíngues — 75 MB a 466 MB). Mesma UX de modelos da suíte: **usuário aponta a pasta**, nada vai no instalador. Tela de modelos com download direto do Hugging Face (`ggml.bin` oficiais do whisper.cpp) com sha256 conferido — padrão do `translate_manifest.json` do LocalZIM.
- **Decodificação de áudio:** whisper quer PCM 16 kHz mono. **Rust puro: `symphonia`** (mp3/m4a/ogg/flac/wav) + **`rubato`** (resample). Sem ffmpeg aqui — mantém o app leve e o CI simples.
- **Gravação de microfone:** crate **`cpal`** → WAV temporário → mesmo pipeline.

**Funcionalidades por fase:**
- **v0.1 (MVP):** abrir arquivo de áudio → transcrever com progresso e cancelamento (evento, como o indexador do LocalZIM); resultado com timestamps por segmento; player de áudio embutido (`<audio>` do webview) com **clique no segmento → pula pro tempo**; export TXT e MD; idioma auto/pt/en/es.
- **v0.2:** gravação de microfone; export **SRT/VTT** (vira legenda — sinergia com LocalPlayer); editor de transcript (corrigir texto mantendo timestamps); fila de arquivos.
- **v0.3 (IA, porta 8105):** resumo e tópicos via llama-server com **map-reduce portado do Writer/LocalPDF** (`chunkDocument`); "gerar ata de reunião" (template); enviar transcript pro OpenObsidian (salvar `.md` na pasta que o usuário escolher).
- **Fora de escopo (registrado):** diarização (quem falou) — whisper.cpp não faz bem; tradução da transcrição usa o caminho do LocalTranslate no futuro, não o modo translate do whisper (só vai pra inglês).

**Riscos:** vídeo como entrada (extrair áudio de mp4) exige demux — symphonia lê AAC em mp4, cobre o caso comum; o que não abrir, mensagem honesta apontando o LocalMedia ("extraia o áudio lá").

---

## 3. LocalMedia — conversor e cortador de mídia (ffmpeg) — **IMPLEMENTADO (2026-07-13, v0.1.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalMedia (código MIT; binários ffmpeg GPL embutidos), pasta `LocalMedia/` (submodule), porta dev **1446**, identifier `com.localmedia.app`, sem IA. No catálogo do TaylorHub (categoria nova **Mídia**) desde o commit v0.7.0 do Hub. **As fases v0.1–v0.3 saíram juntas na v0.1.0**: presets (MP4 web, WhatsApp, WebM, re-container MKV, MP3/Opus e **WAV 16 kHz mono pro LocalScribe**), compressão CRF com estimativa (rotulada de grosseira), corte com timeline de miniaturas + preview + "usar posição atual" (`-c copy` por padrão, com aviso do quadro-chave), juntar clipes (concat demuxer com checagem de compatibilidade; incompatível = mensagem honesta mandando converter antes), GIF 2 passes (palettegen/paletteuse), faixas de áudio/legenda sem re-encode (mov_text no mp4), redimensionar/rotacionar, loudnorm EBU R128 e lote com `unique_path`. Decisões de implementação:
- **Args do ffmpeg montados no front** (`src/lib/presets.ts`, funções puras com 18 testes vitest); o Rust injeta `-progress pipe:1` e faz progresso estruturado (µs→ms do `out_time_ms`), cancelamento por job e cauda do stderr como mensagem de erro.
- Windows: build GPL do BtbN (link `latest` estável); Linux: estático do johnvansickle (roda em qualquer glibc). ~
- Thumbnails da timeline e preview via asset protocol (escopo home/mídias; formato que o webview não toca cai nas miniaturas + campos de tempo).

**Pendências:** teste GUI real na máquina do João; editor multi-faixa e captura de tela seguem fora de escopo (captura → LocalImage).

**Stack específica:**
- **Motor:** **ffmpeg CLI como sidecar** (BtbN win64-gpl zip no Windows; build estático johnvansickle no Linux), baixado por `scripts/fetch-ffmpeg.{ps1,sh}` no CI e no dev. Sidecar por CLI é a escolha **técnica** (processo isolado, progresso parseável, zero build nativo no CI) — a licença nem entra na conta (copyleft OK); linkar libav* no Rust continua não valendo a pena pela complexidade de build.
- **Progresso:** `-progress pipe:1` parseado no Rust → evento Tauri → barra por job. Fila de jobs serializada (1 encode por vez; encode paraleliza mal em máquina modesta).
- **Preview:** `<video>` do webview pros formatos que o WebView2 toca; pros demais, thumbnail extraído via ffmpeg (`-frames:v 1`).

**Funcionalidades por fase:**
- **v0.1 (MVP):** converter formato (mp4/mkv/webm/mp3/opus/wav… presets simples "vídeo pra web", "só áudio", "compatível com WhatsApp"); comprimir por qualidade (CRF com slider e estimativa); extrair áudio; fila com progresso e cancelamento; arrastar arquivos pra janela.
- **v0.2:** **cortar por trecho** com timeline visual (thumbnails) — regra de ouro: quando não muda codec, `-c copy` (corte instantâneo, sem re-encode); juntar clipes (concat demuxer quando os codecs batem); redimensionar/rotacionar.
- **v0.3:** GIF de trecho (palettegen/paletteuse); remover/escolher faixa de áudio e legenda (mkv); normalizar volume (`loudnorm`); lote (aplicar o mesmo preset em N arquivos).
- **Fora de escopo:** editor de vídeo com timeline multi-faixa (é outro produto); captura de tela (vai pro LocalImage).

**Riscos:** o binário ffmpeg é ~80–120 MB — vai no instalador (como os LSPs do LocalCode, baixado no workflow). Parsing de stderr do ffmpeg é frágil — usar sempre `-progress` estruturado, nunca regex no log humano.

---

## 4. LocalImage — visualizador, anotador e captura de tela — **IMPLEMENTADO (2026-07-13, v0.1.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalImage (MIT), pasta `LocalImage/` (submodule), porta dev **1448**, identifier `com.localimage.app`, sem sidecar e sem IA. No catálogo do TaylorHub (Mídia) desde o commit v0.8.0 — **com `extensions: []` de propósito** (a decisão "não roubar o visualizador do SO" virou regra no instalador também: o NSIS não registra associação nenhuma; "abrir com" funciona por argumento + single-instance). **Fases v0.1–v0.3 + o lote da v0.4 saíram juntos na v0.1.0**: visualizador (pasta com ←/→, tira de miniaturas, zoom por passos com Ctrl+roda, ajustar/100%, pan, fullscreen, girar exibição, lixeira do SO via crate `trash`, painel EXIF), anotador (seta, caixa, realce, tarja, desenho, texto com contorno, **passos numerados** e crop; anotações em coordenadas da imagem original queimadas em resolução nativa; undo/redo; copiar pro clipboard), captura de tela/janela via **xcap** caindo direto no anotador (esconde o app antes, histórico em app_data/captures, **atalho global opt-in** via tauri-plugin-global-shortcut), converter/redimensionar/comprimir (crate `image`; WebP lossy pelo canvas do webview — o crate só faz WebP lossless) e lote PNG/JPG. Decisões:
- **Privacidade por construção:** todo export re-encoda ⇒ EXIF (GPS/câmera/data) nunca sobrevive; o painel EXIF avisa isso.
- **Editor carrega a base via Rust em data: URL** (canvas nunca fica "tainted", e TIFF & cia. funcionam); o visualizador usa asset protocol com fallback pro decoder Rust.
- **Região da captura = capturar a tela + crop no anotador** (sem overlay nativo — robusto em Wayland).

**Pendências (v0.4 do plano):** remoção de fundo (rembg/onnx do Slides) e OCR (tesseract do LocalPDF) ficaram pra próxima versão; teste GUI real na máquina do João (inclusive captura em Wayland e clipboard de imagem no Linux).

**Stack específica:**
- **Operações de imagem:** crate **`image`** no Rust (decode/resize/convert/compress png-jpg-webp-gif-bmp-tiff) — rápido e sem sidecar. EXIF via `kamadak-exif` (mostrar e **opção de remover ao exportar** — privacidade).
- **Anotação:** camada de anotações **portada do LocalPDF** (setas, caixas, realce, tarja, texto, desenho à mão, carimbo) sobre canvas 2D; anotações "queimam" no export como lá. Crop com a UX do Slides (pan/zoom estilo Canva).
- **Captura de tela:** crate **`xcap`** (multiplataforma, Windows/X11/Wayland) — tela cheia, janela, região (overlay de seleção). Atalho global via `tauri-plugin-global-shortcut` (configurável, desligado por padrão pra não brigar com o SO).
- **Remoção de fundo:** **onnxruntime-web portado do Slides** (mesmo modelo).
- **OCR:** **tesseract.js portado do LocalPDF** (por/eng, tessdata_fast via fetch-script) — "copiar texto da imagem".

**Funcionalidades por fase:**
- **v0.1 (MVP):** visualizador rápido (abrir arquivo → navegar a pasta com setas, zoom Ctrl+roda, ajustar/100%, fullscreen, girar, deletar com confirmação); converter/redimensionar/comprimir um arquivo; associação de extensão **desligada por padrão no Hub** (não roubar o visualizador do SO sem o usuário pedir — decidir na entrada do catálogo).
- **v0.2:** anotação completa + crop; copiar resultado pro clipboard (`tauri-plugin-clipboard-manager` com imagem); "salvar como" mantendo original.
- **v0.3:** captura de tela (tela/janela/região) caindo direto no anotador; atalho global opcional; histórico de capturas em pasta configurável.
- **v0.4:** lote (converter/redimensionar N arquivos); remoção de fundo; OCR; strip de miniaturas da pasta.
- **Fora de escopo:** edição raster de verdade (camadas, brushes, filtros) — ver nota do Krita no §6.

**Riscos:** Wayland restringe captura (xcap usa portais — testar no CI só compila, captura é teste manual); clipboard de imagem no Linux é historicamente chato — aceitar "salvar em arquivo" como fallback honesto.

---

## 5. LocalPlayer — player de vídeo minimalista (motor pronto) — **IMPLEMENTADO (2026-07-14, v0.1.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalPlayer (MIT, público; submodule `LocalPlayer/`), porta dev **1450**, identifier `com.localplayer.app`, associações de mídia (`mp4, mkv, webm, avi, mov, m4v, mpg, mpeg, flv, ts, wmv` + `mp3, flac, m4a, ogg, opus, wav, aac, wma`). No catálogo do TaylorHub (categoria Mídia, id `player`, com extensões associadas — não há conflito, é o único player) — **falta a release nova do Hub**. Front verde (build+34 vitest) e **`cargo check` no Windows confirmou o caminho winapi/embed** (regra Actions-puro dispensada só pra isso, `target/` apagado no fim; o CI Linux não compila o embed). Decisões que ajustaram o plano:
- **Motor mpv como PROCESSO SEPARADO + JSON IPC** (`--input-ipc-server`), **não** a crate `libmpv2` linkada. Mesma lógica do LocalMedia (ffmpeg CLI) e LocalScribe (whisper-cli): zero build nativo no CI, crash do mpv não derruba o app, e o código segue **MIT** (binário GPL do mpv é só um processo ao lado). O Rust é um "cano burro" (`mpv.rs`): manda `{"command":[...]}` e repassa cada linha de evento pro front (`mpv-event`); quem observa propriedades e interpreta é o TS (`mpvEvents.ts`, testado).
- **Embed no Windows (Plano A):** child window nativa (winapi, classe "Static", sem WndProc próprio) criada sob o HWND do Tauri e passada pro mpv via `--wid` (`embed.rs`). A UI fica ao redor num **layout de grade** — o embed nunca fica atrás da UI; **modal/popover escondem o embed** (`stage_rect(...,visible=false)`) pra aparecerem por cima, e áudio-puro também esconde (mostra "now playing"). Coordenadas em pixels FÍSICOS (× devicePixelRatio), sincronizadas por `ResizeObserver`.
- **Plano B assumido como fallback + toggle:** "Vídeo em janela separada" nas Configurações (Windows) e **automático no Linux** — a v0.1 do Linux usa o **mpv do sistema** (`apt install mpv`) em janela própria; embed X11 fica pra depois. Mesma IPC/controles nos dois modos. (A "fase 0" spike não foi validada interativamente aqui — ver pendência.)
- **Fases v0.1 + boa parte da v0.2 saíram juntas:** playlist da pasta em ordem natural (aleatório, repetir tudo/uma), legendas embutidas/externas (`sub-add`), faixas de áudio, **capítulos** na barra e no menu, **velocidade** com `scaletempo2` (voz inteligível), **loop A-B**, print (`screenshot`), **lembra a posição** (watch-later do mpv), tema claro/escuro/sistema, imersivo com auto-ocultar controles, tela cheia, recentes, e atalhos estilo mpv (Espaço, ← →, J/L, ↑↓, F, M, T, N/P, S, [ ], R, Tab, 0–9).

**v0.1.1 (2026-07-14) — corrige o vídeo que "não abria":** o 1º comando ao mpv travava. Gotcha do Windows: ler e escrever no MESMO named pipe por handles clonados serializa o I/O síncrono no mesmo file object (leitora bloqueada num `ReadFile` prende a escrita) → deadlock; `openFile` nunca terminava e a tela ficava na inicial **sem erro**. Fix: handle único com leitura **não-bloqueante** (`PeekNamedPipe`) sob `Mutex` (Unix mantém socket full-duplex). Detalhe no [MEMORY.md](../../MEMORY.md).

**v0.1.2 (2026-07-14) — o embed foi REPROVADO no teste real e o Plano B virou o padrão.** Duas lições pagas: (1) a child window do `--wid` **briga com a composição do WebView2** (janela inteira preta, processo instável — visto na máquina do João); (2) os comandos `async` do Tauri chegam **fora de ordem** — um `stage_rect` "esconde" velho aterrissava depois do "mostra" e deixava o vídeo invisível (fix: número de sequência com `fetch_max` atômico no Rust; o front só esconde o palco no unmount real). Resultado: **padrão = janela própria do mpv com OSC/atalhos nativos LIGADOS** (a janela do vídeo é um player completo por si; o app soma playlist/legendas/velocidade via IPC — estado sincroniza pelos observe_property). O embed ficou como **toggle experimental** (`settings.embedVideo`, chave nova pra ignorar configs antigas). **Verificado com vídeo real (h264+aac): janela do player mostra o vídeo com título correto, IPC bidirecional ok.** Se o embed voltar um dia, o caminho a investigar é webview transparente + child ATRÁS (não por cima).

**v0.1.3 (2026-07-14) — a "tela preta do app" era um loop de setState (React #185), não o embed.** O ControlBar assinava a store de UI **inteira** (`useUi()` sem selector) e usava o objeto como dep de um `useEffect` que chama `setPopoverOpen`: cada set → estado novo → re-render → objeto novo → efeito → set → **loop infinito**; o React derruba a árvore e sobra o body escuro do tema (a janela preta do teste do João; também explicava webviews "morrendo" nos testes). Só estoura no **build de produção**. Fix: selectors individuais + actions (estáveis). **Lição de suíte (variante do gotcha do LocalPDF):** nunca usar a store inteira do zustand como dependência de efeito que seta estado. **Verificado contra o build de produção** (`vite preview` na porta do dev + exe debug — receita boa pra reproduzir bug "só em prod" sem bundlar instalador): PlayerView com topbar/controles/playlist visíveis (medido via DOM) e mpv tocando.

**Pendências:** teste do João na v0.1.3 (UI do app + janela do vídeo + atalhos); v0.3 = sinergia "gerar legenda com o LocalScribe" (detecção via registro); miniatura no hover da barra; embed X11 no Linux.

**Decisão de motor (registro original):** **libmpv** (crate `libmpv2`). O mpv é o melhor motor pronto que existe (toca tudo, legendas perfeitas, watch-later) e a filosofia é exatamente essa: **não escrever pipeline de vídeo — usar motor consagrado e fazer só a casca leve**.

**Arquitetura (o ponto técnico crítico do app):**
- **Embed por janela filha:** no Windows, criar child window nativa (HWND via `raw-window-handle` da janela Tauri) e passar `--wid` pro mpv — o vídeo renderiza nela; os controles ficam no webview por cima (overlay auto-oculto). No Linux/X11 igual; **Wayland não suporta `--wid`** → fallback: render API do mpv (OpenGL callback) ou, plano C, janela do mpv separada controlada pelo app via IPC.
- **Plano B assumido no plano:** se o embed brigar com o webview (z-order/transparência), a v0.1 sai com **mpv em processo separado + `--input-ipc-server`** (named pipe no Windows / socket no Linux) e o app vira controle remoto bonito. Funcionalidade idêntica, menos elegante. Decidir no spike da fase 0.
- **Licença:** copyleft OK (decisão 2026-07-13) — usar os **builds GPL completos do mpv** sem cerimônia (mais codecs/features que o build LGPL); se o linking tornar o app GPL, o app sai GPL. Binário/dll via `scripts/fetch-mpv.{ps1,sh}` (builds do mpv pra Windows; no Linux, dependência do sistema ou AppImage com a lib).

**Funcionalidades por fase:**
- **fase 0 (spike, 1–2 dias):** provar embed `--wid` + overlay webview no Windows. O resultado decide plano A vs B.
- **v0.1 (MVP):** abrir arquivo/pasta (playlist da pasta); play/pause/seek/volume/velocidade; fullscreen; legendas (carregar `.srt` externo + faixas embutidas — o mpv faz sozinho); **lembrar posição** (`--save-position-on-quit`); atalhos padrão mpv (espaço, setas, `[`/`]`, `s` screenshot); associações mp4/mkv/webm/avi/mov + áudio.
- **v0.2:** miniatura no hover da barra (thumbfast-like via segundo mpv ou ffmpeg thumbs); áudio/legenda selecionáveis no menu; A-B loop; equalização básica de velocidade de fala (`--af=scaletempo2`).
- **v0.3 (sinergia):** "gerar legenda com o LocalScribe" — se o Scribe estiver instalado (detecção estilo Hub via registro), botão exporta o áudio e abre lá; a legenda SRT volta pra pasta do vídeo.
- **Fora de escopo:** biblioteca de mídia/metadata scraping (é outro produto — o app é player, não Jellyfin), streaming de rede, codecs além do que o mpv trouxer.

**Riscos:** este é o app com maior risco técnico da lista (embed nativo × webview). Por isso a fase 0 é um spike com critério de saída explícito, antes de qualquer UI.

---

## 6. LocalDraw — diagramas (Excalidraw embutido) — **IMPLEMENTADO (2026-07-14, v0.2.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalDraw (MIT, público; submodule `LocalDraw/`), pasta `LocalDraw/`, porta dev **1452**, identifier `com.localdraw.app`, IA llama na 8106–8126, associação `.tdraw`. Releases v0.1.0 e **v0.2.0** (Windows NSIS + Linux AppImage) publicadas pelo Actions; CI verde (build+10 vitest+cargo test). **No catálogo do TaylorHub (categoria Escritório) desde o Hub v0.8.2.** **A decisão foi Excalidraw (não o porte do Slides)** — o spike valeu: é um componente React MIT com canvas infinito, conectores ancorados que seguem as formas, undo/redo e polimento prontos. Decisões:
- **Embed offline de verdade (o gotcha pago aqui):** o Excalidraw 0.18 faz **lazy-load das fontes por CDN (esm.sh)** em runtime — inaceitável pra suíte offline. Fix: `window.EXCALIDRAW_ASSET_PATH="/excalidraw-assets/"` no `index.html` (antes do módulo carregar — imports ES são içados) + um plugin Vite inline (`vite.config.ts`) que serve as fontes do pacote em dev (middleware) e as copia pra `dist/excalidraw-assets/fonts` no build (não versionadas). Verificado no navegador: 234 font-faces, Excalifont servida local (200/`font/woff2`), **zero requests externos**. `@excalidraw/excalidraw/index.css` importado empacota o resto.
- **Formato `.tdraw` = JSON padrão do Excalidraw** (`serializeAsJSON`), não zip+media como o `.tslides`: o Excalidraw já embute imagens inline em `files`, então um `.tdraw` também abre como `.excalidraw`. Rust só move bytes/texto (comandos `read_text_file`/`write_text_file`/`write_file_base64`), abrir por "abrir com" via single-instance.
- **IA de fluxograma (padrão deckgen do Slides):** a IA devolve `{nodes,edges}` (JSON), o código valida em TS (`diagram.ts`, puro e testado), faz **layout topológico em camadas (Kahn, tolerante a ciclo)** e monta a geometria com `convertToExcalidrawElements` (`diagramBuild.ts`) — nós viram rectangle/diamond/ellipse por tipo, arestas viram setas **vinculadas por id** que seguem as formas. Nunca aplica coordenada crua do modelo.
- Export PNG (`exportToBlob`) e SVG (`exportToSvg`); tema claro/escuro/sistema; atalhos Ctrl+N/O/S/Shift+S; aviso de alterações não salvas ao fechar.

**v0.2.0 (2026-07-14) — "Excalidraw offline de verdade" (não só o site numa janela):** (1) **menu próprio 100% offline** via `<MainMenu>` customizado — remove os itens online/promo do padrão (Excalidraw+, redes, colaboração ao vivo, login) e liga os comandos de arquivo nativos; (2) **recuperação de sessão** — autosave da cena no `localStorage` (`serializeAsJSON`, debounce 700ms) restaurado via `initialData` no mount, com a baseline ancorada no próprio initialData pra não abrir marcado como sujo (a corrida do restore assíncrono dava ● falso); (3) **biblioteca de formas persistente** + import `.excalidrawlib` offline via `useHandleLibrary`. Verificado no navegador (`vite preview`): menu sem tralha online, cena injetada some/volta no reload, abre limpo. Sobra refino: tela de "recentes" e os links online do command palette (Ctrl+/).

**Pendências:** teste GUI real (`tauri dev`) + IA com `.gguf` na máquina do João; v0.3+ (roteamento de conector desviando de formas, ícones embutidos, modelos prontos) segue como evolução. O porte do Slides abaixo fica como **registro histórico** (plano B que não foi necessário).

---

### Plano original (porte do Slides) — mantido como referência

**Nota honesta sobre "reaproveitar muito do Krita" (pedido do João):** a licença GPL não é o problema (copyleft OK desde 2026-07-13) — o problema é **técnico**: o Krita é C++/Qt, e nada dele roda dentro de Tauri/React/TS. Do Krita reaproveitam-se **UX e conceitos** (docking de painéis, atalhos, presets, autosave incremental). **Se a intenção era pintura raster estilo Krita** (brushes, camadas raster, tablet/pressão), isso é um app diferente e enorme — registrado no fim desta seção como "LocalPaint — a avaliar", fora deste plano. O LocalDraw abaixo segue o §4.3 do projetos.md: **diagramas (Visio/draw.io local)**.

**Atalho a avaliar antes de codar (aberto pela liberação de licenças):** existem dois motores de diagrama open source que RODAM no nosso stack e poderiam substituir o porte do Slides:
- **Excalidraw (MIT)** — é literalmente um **componente React**: canvas infinito, formas, conectores que seguem elementos, texto, imagem, export PNG/SVG, tudo pronto. Embutir no Tauri + salvar `.excalidraw`/`.tdraw` + IA por cima = MVP em dias, não semanas. Contra: estética "desenhado à mão" (tem modo reto, mas a identidade é outra) e menos controle sobre o modelo de dados.
- **draw.io/mxGraph (Apache-2.0)** — o mais completo (fluxograma sério, ER, rede), mas é um app monolítico JS antigo; integrar é embutir, não portar — vira "casca do draw.io", com pouco espaço pra IA/identidade da suíte.

**Decisão proposta:** spike de 1 dia com Excalidraw embutido; se a estética e o modelo servirem, o LocalDraw v0.1 nasce dele (fases abaixo viram: v0.1 = embed + `.tdraw` + associação, v0.2 = export/IA); senão, segue o porte do Slides abaixo.

**Origem:** ~70% é **porte do LocalSlides** (canvas posicional, formas SVG, snapping com guias, alinhar/distribuir, grupos, camadas, zustand+immer com undo, export PNG). O que muda:

- **Canvas infinito** em vez de slide 1280×720: viewport com pan (espaço+arrastar) e zoom no cursor; sem conceito de "apresentar".
- **Conectores** (a feature que justifica o app): linha ligada a **âncoras** das formas (N/S/L/O + centro), que **segue a forma ao mover**; roteamento ortogonal simples (Manhattan com 1–2 dobras) no MVP, curvas depois; setas nas pontas; rótulo de texto no conector.
- **Formas de fluxograma:** processo, decisão, terminador, dado, BD, documento, nota — SVG paths como as 14+ formas do Slides, mesma infra de estilo de traço.
- **Formato nativo `.tdraw`** (zip: `doc.json` + `media/`, mesmo desenho do `.tslides`).

**Funcionalidades por fase:**
- **v0.1 (MVP):** porte do canvas do Slides (formas, texto TipTap, imagens, snapping, grupos, camadas, undo) + canvas infinito + conectores ortogonais com âncoras + paleta de fluxograma + `.tdraw` + export PNG.
- **v0.2:** **export SVG** (o canvas já é geometria — mapear elementos pra SVG puro); export PDF (print view como Slides); páginas múltiplas no mesmo arquivo; ink (portar InkEl do Slides).
- **v0.3:** roteamento de conector desviando de formas; ícones embutidos (pack offline tipo Tabler); modelos prontos (fluxograma, orgstack, rede, ER).
- **v0.4 (IA, porta 8106):** "gera fluxograma disso" → JSON `{nodes:[{id,type,label}], edges:[{from,to,label}]}` validado + **layout automático em TS** (camadas topológicas simples) → elementos reais via store (mesmo contrato do deckgen: nunca aplicar geometria crua do modelo).
- **Import/export draw.io (XML) fica registrado como ideia**, sem compromisso (formato documentado, mas grande).

**LocalPaint (a avaliar, fora do plano):** se o João quiser pintura raster, o caminho leve seria canvas 2D/WebGL com brushes simples + camadas raster + pressão de caneta (Pointer Events já dá pressão no WebView2) — escopo próprio, decidir depois do LocalDraw.

---

## 7. LocalTranslate — tradutor offline (extração do LocalZIM) — **IMPLEMENTADO (2026-07-15, v0.1.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalTranslate (MIT, público), porta dev **1454**, identifier `com.localtranslate.app`, sem sidecar/porta (candle in-process), sem associação. Releases Windows NSIS (`LocalTranslate_0.1.0_x64-setup.exe`, ~4 MB — sem sidecar, WebView2 do sistema) + Linux AppImage (`LocalTranslate_0.1.0_amd64.AppImage`, ~81 MB) publicadas pelo Actions **na 1ª tentativa** (sem job flaky). CI verde (front build + 10 vitest + 10 cargo test). **Ainda falta:** entrar no catálogo do TaylorHub (categoria Inteligência artificial) + release nova do Hub, e virar submodule do superprojeto `Local/`. Decisões que ajustaram o plano:
- **Motor portado do LocalZIM** (`translate.rs` + `translate_manifest.json`), mas **sem o cache por artigo/UUID** do ZIM — aqui o texto vem das duas caixas e o histórico fica no SQLite. A API pública virou `translate(direction, text)` (multi-linha, preserva quebras; linha sem letra passa intacta) sobre o mesmo lote-por-perna com pivô pt↔es pelo inglês. Mesma release `v1` de modelos do `LocalZIM-models` (download por perna com sha256 conferido em streaming, evict da RAM após 5 min ociosos).
- **v0.1 saiu com o MVP do plano:** duas caixas + seletor das 6 direções + swap, **detecção de idioma por heurística** (stopwords distintivas + pistas de acento ã/õ/ç→pt, ñ/¿/¡→es) com sugestão de inverter, copiar, `Ctrl+Enter`, abrir `.txt`/`.md`, **histórico SQLite** (500 últimas, reabre no clique, dedup da última idêntica) e **gerenciador de modelos** (baixa só os pares usados, barra de progresso via evento `download-progress`/`download-done`, remover). Tema claro/escuro/sistema. `lto=true` no release profile (decodificação candle é CPU-bound).
- **Pendências:** teste GUI real (`tauri dev` + baixar um par de modelo) na máquina do João; v0.2 (traduzir arquivo preservando markdown com `pulldown-cmark`), v0.3 (janela rápida por atalho global + bandeja), v0.4 (fr/de + glossário) seguem como evolução.

**Origem (plano):** o motor já existe e está testado — Marian/OPUS-MT no **candle** (CPU, crate `tokenizers`), 4 direções (en↔pt-BR, en↔es, pt↔es pivotando pelo inglês), bundles hospedados na **release `v1` do `Anon5T4R/LocalZIM-models`**. Este app é ~"só" UI + extração do módulo.

**Stack específica:**
- **Portar do LocalZIM:** `translate.rs` (motor + `legs()`), `translate_manifest.json` (sha256/bytes embutido) e o downloader de bundles. Sem sidecar e sem porta — candle roda **in-process**. Regra da casa: mudou bundle ⇒ regenerar manifest e commitar.
- Modelos novos = pipeline documentado no LocalZIM-models (`convert.py`, venv transformers+torch) → sobe asset na release → adiciona direção no manifest + `legs()` + UI.

**Funcionalidades por fase:**
- **v0.1 (MVP):** duas caixas (origem→destino), detecção simples de idioma (heurística por stopwords, os 3 idiomas), swap, copiar resultado, histórico local (últimas N traduções, SQLite ou JSON), tela de gerenciamento de modelos (baixar/remover por direção, com tamanho).
- **v0.2:** traduzir **arquivo** `.txt`/`.md` (preservando estrutura de markdown: traduzir só nós de texto — parsear com `pulldown-cmark`, não regex); fila com progresso (documento grande = chunks por parágrafo).
- **v0.3:** **janela rápida** — atalho global (`tauri-plugin-global-shortcut`, opt-in) abre mini-janela com o clipboard já colado e traduzido; bandeja.
- **v0.4:** mais idiomas (fr, de — depende de converter os pares OPUS-MT e subir na release); glossário do usuário (termos que não traduz).
- **Fora de escopo:** DOCX/PDF (usuário converte no Writer/LocalPDF antes); tradução dentro dos outros apps continua sendo de cada app (regra de independência).

**Risco:** baixo — é o app mais barato da lista. Cuidado só com RAM: um par carregado por vez (LRU de 1–2 modelos), descarregar ao minimizar pra bandeja.

---

## 8. LocalKeys — gerenciador de senhas ⚠️ segurança primeiro — **IMPLEMENTADO (2026-07-15, v0.2.0)**

**Estado:** repo https://github.com/Anon5T4R/LocalKeys (**GPL-3**, público), porta dev **1456**, identifier `com.localkeys.app`, ext `.tkeys`, categoria **Segurança** (nova) no Hub. No catálogo do TaylorHub (release **v0.8.4**) e virou submodule do superprojeto `Local/`. CI verde nas duas versões (front build + vitest + cargo test + **cargo audit**); releases NSIS + AppImage publicadas pelo Actions **na 1ª tentativa** as duas vezes.

- **v0.1.0:** cofre `.tkeys` (**XChaCha20-Poly1305 + Argon2id**, header em claro autenticado como AAD, chave só no back-end com `zeroize`, **11 testes de propriedade** — round-trip/senha errada/adulteração blob+header+salt/nonce único), CRUD dos 4 tipos, busca, favoritos, lixeira, **gerador** (senha por classes + passphrase diceware wordlist EFF real de 7776 palavras) + **força zxcvbn**, auto-lock por inatividade e ao ocultar, limpeza de clipboard em 30 s. `SECURITY.md` com threat model.
- **v0.2.0:** **TOTP** (RFC 6238 via `totp-rs`, código de 6 dígitos com contagem regressiva; **KAT do RFC** nos testes); **importadores** — Bitwarden JSON, CSV (Chrome/Edge, LastPass, 1Password, genérico por mapeamento de colunas) e **KeePass `.kdbx`** (crate `keepass`, leitura); **export** JSON/CSV em claro com aviso forte.

**Decisões/gotchas que ajustaram o plano:**
- **Cripto ficou no Rust** (não a TS do Bitwarden, como o plano já previa). Formato `.tkeys` = header de 60 bytes (`TKEYS\0` + versão + params Argon2id + salt 16 + nonce 24) **autenticado como AAD** + blob; salvar recifra com nonce novo reusando a chave de sessão (não re-roda Argon2 nem guarda a senha). Regra "nenhuma primitiva caseira" mantida (só RustCrypto, que já traz os vetores IETF).
- **`totp-rs`: usar `TOTP::new_unchecked`, NÃO `new`** — o `new` exige segredo ≥128 bits, mas segredos reais (Google Authenticator etc.) têm 80 bits (16 chars base32); `new` rejeitaria e quebraria o import. `default-features=false` tira o QR/parser de URL.
- **`keepass` fixado em 0.7** (0.13 é a última, API diferente): `Database::open(&mut Read, DatabaseKey::new().with_password(pw))`, iterar `&db.root` → `NodeRef::Entry(e)`, getters `get_title/username/password/url` + `get("otp")`/`get("Notes")`. Extrai o `secret=`/`key=` de otpauth.
- **Importadores em TS** (mapeamento de colunas), não o porte pesado do `libs/importer` do Bitwarden — cobre os formatos mais comuns com muito menos código; só o `.kdbx` (binário/cifrado) foi pro Rust.

**Ainda falta / pendências:**
- **Teste GUI real** na máquina do João (rodar o `.exe`/`.AppImage`) — o que foi validado é compilação + testes no CI, não o app rodando.
- **v0.3:** histórico de senhas por item, campos customizados, anexos pequenos cifrados no blob, relatório local de senhas fracas/repetidas.
- **Endurecimentos:** limpeza de clipboard movida pro Rust (`arboard`), **gravação atômica** (rename em vez de escrever por cima com `.bak`), auto-lock também no back-end.
- **v0.4 (a avaliar):** import do JSON **cifrado** do Bitwarden, auto-type via `enigo`. **keyring/Windows Hello** (desbloqueio opt-in) ainda não foi feito.

**Nota sobre "reaproveitar muito do Bitwarden" (revisada com copyleft liberado):** os clients do Bitwarden são **TypeScript (GPL-3)** — e com GPL OK, **dá pra portar código de verdade**, porque TS roda no nosso front. O LocalKeys sai **GPL-3** e reaproveita do monorepo `bitwarden/clients`:
- **`libs/importer`** — o maior prêmio: **dezenas de importers prontos** (LastPass, Chrome/Edge, 1Password, KeePass, Dashlane, CSV genérico…) em TS puro, portáveis com ajustes pequenos. Isso transforma o import (normalmente o recurso mais chato de um gerenciador) em trabalho de adaptação.
- **Gerador de senhas/passphrase** e a lógica de força/política — TS portável.
- **O desenho** (vault cifrado por master password reforçada por KDF, 4 tipos de item, TOTP embutido, auto-lock, timeout de clipboard).

O que **não** portar: a cripto TS do Bitwarden (é orientada a navegador/sync com servidor — protected key, enc-strings). A cifra do vault fica **no Rust** com crates auditados (abaixo), que é mais forte no nosso modelo (chave nunca no webview). Partes do SDK novo do Bitwarden têm licença comercial própria (Bitwarden License) — **usar só o que for GPL** do repo clients. Regra absoluta mantida: **nenhuma primitiva inventada em casa**.

**Arquitetura de segurança (o coração do app):**
- **Formato `.tkeys`:** arquivo único = header claro (versão, params do KDF, salt) + **blob cifrado** (JSON do vault). Cifra: **XChaCha20-Poly1305** (`chacha20poly1305` do RustCrypto); KDF: **Argon2id** (`argon2`, já usado no LocalData v0.5.0) com params calibrados (~250–500 ms na máquina alvo). Nada de SQLite claro com colunas cifradas — blob único simplifica o modelo e o rekey.
- **Chave só em memória**, no Rust (`zeroize` ao trancar); o front nunca vê a master password depois do unlock (manda pro back, recebe "unlocked"). **Auto-lock** por inatividade (configurável, default 5 min) e ao minimizar/suspender.
- **Clipboard:** copiar senha limpa o clipboard após 30 s (timer no Rust); avisar que histórico de clipboard do Windows deve ficar desligado (link pra config do SO).
- **Keyring do SO** (crate `keyring`, já usada no LocalCode/TaylorChat): **opcional e opt-in**, só pra "desbloquear com o Windows Hello/sessão" guardando uma chave intermediária — nunca a master password. Default: sempre pedir a master.
- **Sem IA de propósito** (segredos jamais passam por modelo) e **sem rede alguma** no app (nem update check próprio — quem atualiza é o Hub).
- **Backup:** cópia automática do `.tkeys` ao abrir (mesmo padrão de retenção do LocalData v0.4.0).

**Modelo de dados (dentro do blob):** itens tipo **login** (usuário, senha, URLs, TOTP secret, notas), **nota segura**, **cartão**, **identidade** (os 4 tipos do Bitwarden); pastas; favoritos; histórico de senhas por item; lixeira com retenção.

**Funcionalidades por fase:**
- **v0.1 (MVP):** criar/abrir vault; CRUD dos 4 tipos; busca instantânea; **gerador** (comprimento, classes, passphrase estilo diceware com wordlist pt/en embutida); copiar usuário/senha; auto-lock; força da senha (zxcvbn via `zxcvbn` crate) no item e na master.
- **v0.2:** **TOTP** (`totp-rs`, código com contagem regressiva); **import via porte do `libs/importer` do Bitwarden** — começar por Bitwarden JSON/CSV, Chrome/Edge, LastPass e 1Password (os 4 de onde mais se migra) + **KeePass `.kdbx`** nativo (crate `keepass`, leitura); export JSON claro (com aviso forte) e CSV.
- **v0.3:** histórico de senhas; lixeira; campos customizados; anexos pequenos (cifrados no blob, limite de tamanho); relatório local de senhas fracas/repetidas.
- **v0.4 (a avaliar, cada um tem custo/risco):** import do JSON **cifrado** do Bitwarden (formato documentado: PBKDF2/Argon2 + AES-CBC — dá pra decifrar com as mesmas crates); auto-type básico via `enigo` (digitação simulada — explicitamente sem hook de navegador); extensão de navegador **fora do escopo da suíte** por ora (superfície de ataque e manutenção grandes demais).
- **Fora de escopo permanente:** sync próprio (filosofia: a pasta do `.tkeys` no Syncthing/OneDrive do usuário resolve — e o formato blob-único torna conflito detectável), servidor, compartilhamento.

**Processo:** por ser o app crítico, o `ci.yml` roda também `cargo audit`; testes de vetores conhecidos do Argon2/XChaCha; um doc `SECURITY.md` no repo com o threat model (o que protege: arquivo roubado; o que não protege: máquina comprometida com keylogger).

---

## 9. Registro — LocalFeed (explicado, aguardando decisão)

O que seria: **leitor de RSS/Atom** — o usuário assina sites/blogs/portais (toda publicação séria expõe um feed), o app baixa os artigos novos e guarda **pra ler offline**, com resumo opcional pela IA local. É o jeito de acompanhar notícias sem algoritmo, sem conta e sem tracker — casa com o cluster de conhecimento (LocalZIM/OpenObsidian). Crates maduras: `feed-rs` (parse) + `readability` (extrair o artigo limpo da página). **Fica fora do plano até o João dizer se quer.**

---

## 10. Ordem, dependências e sinergias

1. ~~**LocalAgenda**~~ **FEITO (2026-07-13, v0.1.0)** — gap funcional maior; sem risco técnico.
2. ~~**LocalScribe**~~ **FEITO (2026-07-13, v0.1.0)** — entrega SRT/VTT que o LocalPlayer usa depois.
3. ~~**LocalMedia**~~ **FEITO (2026-07-13, v0.1.0)** — o preset "WAV 16 kHz" socorre o Scribe, como planejado.
4. ~~**LocalImage**~~ **FEITO (2026-07-13, v0.1.0)** — anotador próprio + captura; rembg/OCR ficaram pra v0.4.
5. ~~**LocalPlayer**~~ **FEITO (2026-07-14, v0.1.0)** — embed no Windows (winapi/`--wid`) com fallback/janela separada; Linux usa mpv do sistema. Consome legendas do Scribe fica pra v0.3.
6. ~~**LocalDraw**~~ **FEITO (2026-07-14, v0.1.0)** — foi de **Excalidraw embutido** (não o porte do Slides); conectores ancorados + IA de fluxograma. Falta catálogo/Hub.
7. ~~**LocalTranslate**~~ **FEITO (2026-07-15, v0.1.0)** — extração do motor candle do LocalZIM; MVP (2 caixas + detecção + histórico SQLite + gerenciador de modelos). Falta catálogo/Hub + submodule.
8. ~~**LocalKeys**~~ **FEITO (2026-07-15, v0.2.0)** — cofre `.tkeys` (XChaCha20-Poly1305 + Argon2id, chave só no Rust), gerador, auto-lock, TOTP e importadores (Bitwarden/CSV/KeePass). `SECURITY.md` + `cargo audit` no CI, como o processo pedia. No Hub (v0.8.4) + submodule. **Falta:** teste GUI do João; v0.3 (histórico/anexos/relatório) e endurecimentos (clipboard no Rust, gravação atômica).

**✅ Todos os 8 apps do plano foram implementados.** O que resta agora é evolução (v0.2+ de cada) e as pendências transversais abaixo.

**Transversais que este plano reforça:** o **runtime de IA compartilhado (§6.1 do projetos.md)** fica ainda mais necessário (Agenda/Scribe/Draw somam 3 llama-servers novos); o whisper entra na mesma conversa quando o runtime existir. Cada app novo = 1 entrada no catálogo do Hub + release nova do Hub (até a fase 3 do catálogo remoto sair — este plano é mais um argumento pra ela).
