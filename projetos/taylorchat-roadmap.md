# TaylorChat — roadmap de execução (paridade com WhatsApp Desktop)

> Plano de trabalho vivo, criado em 2026-07-09 a partir da **revisão vs WhatsApp Desktop** (artifact + achados no código). Complementa o [plano.md](plano.md) (design + histórico) e o §4.4 do [projetos.md](projetos.md). É a **checklist do que falta** pra sair de "prova de conceito" e virar app de uso diário — organizada em versões shippáveis, do bloqueador ao fôlego grande.
>
> Estado base: **`chat/` v0.1.11** (lista de contatos + canal quente P2P: mDNS LAN, conexão QUIC quente reusada, heartbeat ping/pong, presença em tempo real). Prontidão medida na revisão: **62/100**.
>
> **Regra de "pronto" (Definition of Done) de TODA versão abaixo:**
> 1. `cargo check --features p2p` **e** `cargo check` (padrão) verdes, 0 warnings.
> 2. `npm run build` (tsc + vite) verde.
> 3. i18n pt/es/en de toda string nova.
> 4. Bump de versão nos 3 lugares (`Cargo.toml`, `tauri.conf.json`, `package.json`) + `Cargo.lock`.
> 5. commit + tag `vX.Y.Z` + push → release verde (Windows NSIS + Linux AppImage).
> 6. Atualizar a memória [[projeto-taylorchat-progresso]] e marcar os itens aqui.
> 7. Anotar o que precisa de **teste 2-PC** (não exercitável no assistente).

---

## ⚠️ Pendências operacionais SEMPRE em aberto (não são código)

- [ ] **Teste ponta a ponta com 2 PCs** da v0.1.11: parear → contato aparece nos dois → bolinha de presença fica verde → mensagem nos 2 sentidos → arquivo. Se falhar, **Configurações → Diagnóstico** dos dois lados (o log agora mostra connect/ping/reconexão).
- [ ] Confirmar que o **mDNS** resolveu o "não chega" na mesma LAN (hipótese central da v0.1.11).
- [ ] Testar a **IA local** com um `.gguf` real na máquina (baixar `llama-server` via `scripts/fetch-llama` antes).

---

## v0.2.0 — "Não perco mensagem" 🔴 P0 ✅ ENTREGUE (2026-07-10)

> Um mensageiro que não avisa quando chega mensagem não é um mensageiro. Maior salto de percepção com o menor esforço. **Publicado na v0.2.0** — só falta validar no 2-PC.

### 1. Notificações de desktop ✅
- [x] `tauri-plugin-notification` no `Cargo.toml` + `.plugin(tauri_plugin_notification::init())` no `lib.rs`.
- [x] Permissão `notification:default` em `capabilities/default.json`.
- [x] Dispara ao receber texto/arquivo **só quando a janela não está focada** (`notify_incoming` checa `window.is_focused()`).
- [x] Título = nome do contato (`db::contact_name`: apelido ‖ profile_name ‖ shortId); corpo = prévia do texto ou "📎 arquivo"/"🎨 sticker".
- [ ] **PENDENTE (refino):** clicar na notificação → focar janela + abrir a conversa daquele peer (roteamento de clique no desktop é chato; deixei pra depois).
- [ ] Respeitar futuro "silenciar conversa" (v0.4) — por ora, sempre notifica.
- **Validar no 2-PC:** só observável no build compilado.

### 2. Não-lidos persistentes + badge na bandeja/taskbar ✅  *(resolveu L3)*
- [x] Tabela `unread(convo PK, n)` + comandos `unread_list`/`unread_set`; o front decide quando conta, o back persiste.
- [x] Carrega do banco no boot (não some mais ao reiniciar).
- [x] Total no **tooltip da bandeja** (`tray.set_tooltip`) + **título da janela** (`set_badge`; Windows mostra no hover da taskbar/Alt+Tab).
- [ ] **PENDENTE (refino):** overlay icon de verdade na taskbar (usei o fallback tooltip+título).

### 3. Busca global na sidebar ✅
- [x] Campo de busca no topo da sidebar (acima das abas).
- [x] Filtra **contatos por nome/id** (client-side) + **mensagens por texto** entre conversas (`search_messages`, decifra e casa, para no limite).
- [x] Clicar num resultado → abre a conversa. i18n pt/es/en.
- [ ] **PENDENTE (refino):** rolar até a mensagem exata do acerto + destacar o trecho.

---

## v0.3.0 — "Conversa de gente" 🟠 P1 ✅ ENTREGUE PARCIAL (2026-07-10)

> Publicado na v0.3.0: menu por mensagem (copiar/responder/apagar), links, emoji, "digitando…". **Encaminhar e reações foram pra v0.3.1** (abaixo).

### 4. Menu por mensagem (⋯ no hover da bolha) ✅ (sem encaminhar)
- [x] Menu de contexto na `.bubble` (⋯ no hover, click-away fecha).
- [x] **Copiar texto** (`navigator.clipboard`).
- [x] **Responder / citar**: coluna `reply_to` (ts da citada); envelope leva `replyTo`; prévia "respondendo a" no composer; bolha renderiza a citação. *(rolar até o original = refino pendente)*
- [x] **Apagar para mim**: `message_delete(id)` remove a linha local.
- [x] **Apagar para todos**: `mark_deleted` (soft-delete: esvazia o corpo cifrado + flag); controle `{k:"delete",targetTs}` pelo canal E2E; evento `msg-deleted` sincroniza os 2 lados. Migração `ALTER TABLE messages ADD reply_to, deleted`.
- [ ] **PENDENTE (v0.3.1):** encaminhar (precisa de um seletor de conversa).

### 5. Links clicáveis ✅
- [x] URLs no texto viram `<a>` (sem `dangerouslySetInnerHTML`, sem XSS), abrem com `openUrl`.

### 6. Seletor de emoji no composer ✅
- [x] Popover no composer (botão 🙂), insere na posição do cursor; stickers seguem no 😀.

### 7. "Digitando…" (typing indicator) ✅
- [x] Front: throttle 2s ao digitar + para 3,5s após a última tecla / ao enviar.
- [x] Back: frame de controle leve `{t:"typing"}` pela conexão quente (sem ratchet, como o ping; só se há conexão viva).
- [x] Cabeçalho mostra "digitando…" no lugar do online/offline (timeout de segurança 6s).

## v0.3.1 — resto do P1 + correções da revisão ✅ ENTREGUE (2026-07-10)

### 8. Encaminhar mensagem ✅
- [x] Menu "Encaminhar" → `ForwardModal` (seletor de conversa por atividade + busca). Texto via `sendMessage`; arquivo/sticker via a cópia local (`attachPath`/`sendSticker`).

### 9. Reações de emoji na bolha ✅
- [x] Controle `{k:"reaction", targetTs, emoji}` (vazio = remove); tabela `reactions(convo, target_ts, mine, emoji)`; comando `react` + `reactions_list` + evento `reaction`. Menu com 6 reações rápidas; chips abaixo da bolha; clicar na minha remove.

### 10. Polir estados de entrega  *(resolve achado L5)* — **segue pendente**
- [ ] Usar o ✓ simples pra "saiu, sem ACK" (hoje pula pra ✓✓), ou assumir 2 estados e simplificar o `stateGlyph`.

### Correções da revisão pós-v0.3.0 (entraram na v0.3.1)
- [x] **Compat entre versões — tipo de controle desconhecido não quebra mais.** Recepção (`handle_stream` frame externo `t` e `process_message` conteúdo `k`) **ignora com graça + ACK** em vez de `Err`. (⚠️ v0.3.0 saiu SEM isso — protege v0.3.1 em diante.)
- [x] **Busca não decifra o histórico inteiro a cada tecla** — `search_messages` com `LIMIT 4000` + `deleted=0`.
- [x] **Apagar-para-todos confiável offline** (correção A): tabela `pending_deletes`; `resend_all` reenvia o aviso até o ACK.

**Revisão pós-v0.4.3 (João pediu "conferida em bugs/gaps/riscos") — 2026-07-10:**
Sem bug crítico. Decisões do João sobre os 11 achados → **v0.4.4** (#2 toggle prévia notificação, #3 reação confiável offline via `pending_reactions`, #6 pill "nova mensagem ↓", #7 limpar reações ao apagar, #9 upsert da ficha, #10 âncora de scroll no "carregar antigas") e **v0.4.5** (#4 `reply_preview` guardado → citação não some com paginação; #5 pragmático → `apagar-p/-todos` mira por **direção** out/in, sem colisão de ts e sem dor de migração de id único).
Dispensados pelo João: **#1** (cifrar ficha — "acesso ao PC já fodeu antes disso"), **#8** (badge kw estático após salvar), **#11** (OneDrive — "eu que mexi").
- [x] #2 #3 #4 #5 #6 #7 #9 #10 — entregues (v0.4.4/v0.4.5).

---

## v0.4.0 — "Mídia e escala" 🟢 P2 + achados de lógica ✅ ENTREGUE PARCIAL (2026-07-10)

> Publicado na v0.4.0: paginação (L2), ordem por chegada (L1), prévia de apagada (C), limpeza do não-lido (D). **Voz, silenciar, colar imagem, galeria e L4 seguem pendentes** (abaixo).

### 10. Paginação do histórico ✅  *(resolveu L2)*
- [x] `messages_list(peer, before_id?, limit?)` — carrega as últimas 300 ao abrir; botão "Carregar mensagens antigas" pagina pra cima (prepend). Antes decifrava a conversa inteira.
- [x] Não rola pro fim ao prepender (mantém a posição; detecta prepend por `firstId` menor). *(scroll-anchor exato = refino)*

### 11. Ordenação estável de mensagens ✅  *(resolveu L1)*
- [x] `messages_list` ordena por `id` (ordem vivida NESTE aparelho) em vez de `ts` → relógio torto do par não embaralha. O `ts` (hora exibida + auditoria) segue o do remetente, **intacto** (clamp quebraria a auditoria — por isso ordenei por id em vez de mexer no ts).

### 12. Prévia de apagada na sidebar ✅  *(achado C)*
- [x] `conversations_summary` devolve flag `deleted`; sidebar mostra "🚫 mensagem apagada". *(L6 — ordenar por MAX(ts) — virou sem-efeito: com ordem por id, MAX(id) = última coisa vivida = prévia correta.)*

### 12b. Limpeza do não-lido ✅  *(achado D)*
- [x] `api.unreadSet` saiu de dentro do `setUnread` (usa `unreadRef`).

### 13. Encerrar watcher de presença ocioso ✅  *(resolveu L4 — v0.4.1)*
- [x] `WATCHING` virou `HashMap<peer, token>`; `unwatch()` remove e o watcher sai na próxima checagem (`watch_valid` compara o token — sem corrida com re-watch). ChatPanel chama `peer_unwatch` ao fechar/trocar a conversa → só o chat aberto é observado.

### 16. Silenciar conversa ✅  *(v0.4.1)*
- [x] Coluna `contacts.muted` + comando `set_muted` + `is_muted`; `notify_incoming` pula contato silenciado (não-lido segue contando). Botão 🔔/🔕 no cabeçalho.

### Ficha do contato + limpar cabeçalho ✅  *(v0.4.2, a pedido do João)*
- [x] **ContactProfile** — modal com nome/foto cacheados (dele, só leitura) + campos LOCAIS meus (apelido, telefone, email, aniversário, notas; colunas em `contacts`, comando `set_contact_info`). Abre por **clique-direito** na sidebar e por **clique no nome** no cabeçalho.
- [x] **Apagar contato movido pra dentro da ficha** (zona de perigo separada, com confirmação) — não confunde mais com o 🧹 limpar conversa.
- [x] **Cabeçalho descongestionado** — removidos ✏️ renomear e 🗑 apagar (foram pra ficha); nome/foto viraram botão.
- [x] **v0.4.3:** palavra-chave (🔑) também foi pra ficha (campo com status confere/diverge/aguardando; salva junto). Cabeçalho perdeu o botão 🔑 (o banner "não confere" na conversa fica). Tudo do contato num lugar só.

### Pendentes (v0.4.x / v0.5)
- [ ] **14. Recado de voz** — gravar (`MediaRecorder`), mandar pelo transfer de arquivo (marca `voice`), player inline.
- [ ] **15. Colar imagem do clipboard** — `paste` no composer → salvar bytes → anexar (comando novo `attach_bytes`).
- [ ] **16b. Fixar / arquivar** conversas (mudo já feito).
- [ ] **17. Galeria de mídia** da conversa.
- [ ] **L5** — polir ✓/✓✓ (o ✓ simples é inalcançável).
- [ ] **Latente:** `ts` como identidade da mensagem (reply/delete/react miram por ts).

### 14. Recado de voz
- [ ] Gravar no front (`MediaRecorder`, webm/opus), comprimir se preciso, mandar pelo transfer de arquivo que já existe (marcar `voice: true` no meta).
- [ ] Bolha com player inline (waveform é upgrade; começar com `<audio>`).

### 15. Colar imagem do clipboard
- [ ] Ouvir `paste` no composer; se houver imagem, salvar temp e enviar como anexo (reusa `attach_file`).

### 16. Fixar / arquivar / silenciar conversas
- [ ] Flags por conversa (`pinned`, `archived`, `muted`) em `threads` (ou tabela à parte). Silenciar = não notificar (liga no item 1).
- [ ] UI: menu de contexto na linha da sidebar; seção "Arquivadas".

### 17. Galeria de mídia da conversa
- [ ] Painel que lista todas as imagens/arquivos daquela conversa (query por `kind='file'`), grid clicável.

---

## Polimento de desktop (encaixar quando fizer sentido)

- [ ] **Iniciar com o Windows** (autostart) — `tauri-plugin-autostart`, opção nas Configurações.
- [ ] **Persistir estado da janela** (tamanho/posição) — `tauri-plugin-window-state`.
- [ ] **Painel de info do contato** (clicar no nome no cabeçalho): node_id completo, palavra-chave, mídia compartilhada, botões renomear/remover/limpar/auditar reunidos.
- [ ] Divisor "novas mensagens" na conversa + botão "descer" quando rolado pra cima.

---

## Fôlego grande — backlog planejado (cada um é um projeto próprio)

> Registrados pra **nada ficar faltando**, mas ficam pra depois do P0/P1/P2 por serem grandes/difíceis em P2P puro.

- [ ] **Grupos** — fan-out P2P + ratchet de grupo (Megolm/`vodozemac` group session). Repensar o modelo de conversa (hoje `convo` = 1 contato ou thread do mesmo contato). **Grande.**
- [ ] **Multi-dispositivo** — sincronizar identidade + histórico entre aparelhos do mesmo dono (hoje a chave = 1 instalação). **Grande.**
- [ ] **Chamada de voz/vídeo** — WebRTC/iroh streams; fora de escopo por ora. **Muito grande.**
- [ ] **Editar mensagem enviada** — mensagem de controle `{k:"edit", targetTs, body}`; combina com o soft-delete do item 4.
- [ ] **Mensagens que somem** (efêmeras) / temporizador — opcional, alinhado à pegada de privacidade.

---

## Ordem sugerida de ataque

1. **v0.2.0 inteira** (notificações + não-lidos persistentes + busca) — desbloqueia o uso diário. Já resolve L3.
2. **v0.3.0**: itens 5, 6, 7 (rápidos: links, emoji, typing) → depois 4 (menu por mensagem, o maior) → 8, 9.
3. **v0.4.0**: itens rápidos de lógica primeiro (12, 13, 11), depois paginação (10), depois mídia (14–17).
4. Polimento de desktop entremeado.
5. Backlog grande por último, um de cada vez, com plano próprio.

_Marcar `[x]` aqui a cada item entregue e refletir na memória._
