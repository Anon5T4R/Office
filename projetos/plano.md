# Taylor Chat — plano de implementação (o Mensageiro)

> Plano de 2026-07-07. Detalha e **fecha as decisões abertas** do §4.4 do [projetos.md](projetos.md) ("Taylor Chat — mensageria P2P"). É o análogo do [hub.md](hub.md), mas pro mensageiro P2P sem servidor.
>
> **Nome: TaylorChat** (nome público E codinome). Decisão do João (2026-07-07): fugir do padrão `Local*` porque **um mensageiro não é "local"** — o ponto dele é justamente falar com outra máquina; "LocalChat" soaria contraditório. Fica sob a marca Taylor direto. Pasta `chat/` no Office; repo próprio `Anon5T4R/TaylorChat`, **MIT**.

> **➡️ Trabalho a partir daqui: [taylorchat-roadmap.md](taylorchat-roadmap.md)** (criado 2026-07-09). Este `plano.md` é o design + histórico até o MVP; o roadmap é a checklist do que falta pra **paridade com o WhatsApp Desktop** (P0/P1/P2 + achados de lógica), a partir da **v0.1.11** (lista de contatos, canal quente P2P com mDNS LAN + heartbeat ping/pong + presença em tempo real). Revisão vs WhatsApp: 62/100 de prontidão pra uso diário; maior furo = **notificações de desktop** (P0).

## ESTADO (2026-07-07): FASES 1–6 IMPLEMENTADAS em `chat/` (v0.1.0-dev)

Scaffold do app criado e as duas primeiras fases codadas: **identidade** (chave ed25519 gerada no 1º uso, guardada no keyring do SO) + **banco SQLite** (contatos/conversas/mensagens) + **pareamento** (gerar/mostrar convite e QR, colar/escanear pra adicionar contato) + **shell de UI** (lista de conversas, painel, composer). A **Fase 3 (rede iroh)** já **compila** (`cargo check --features p2p` verde com iroh 0.35); falta só o teste ponta a ponta com 2 instâncias. Ícone próprio feito. Ver §ESTADO detalhado no fim.

## O que o Taylor Chat é (e não é)

**É:** um app desktop pequeno (Tauri 2 + React 19 + Rust, stack padrão da suíte) de **mensagens P2P diretas entre pares, sem servidor próprio, sem cadastro, sem número de telefone**. Substitui de vez a ideia de e-mail (descartada em 2026-07-06). Três coisas:
1. **Conversar** — texto, arquivos comprimidos e mídia, direto ponta a ponta, cifrado.
2. **Parear** por QR code / código de convite (troca de chaves fora de banda) — identidade é uma chave, não uma conta.
3. **Guardar histórico local** cifrado, que o usuário sincroniza por conta própria (Syncthing/OneDrive), como o resto da suíte.

**Não é:** um servidor de chat, um app com conta/login central, um clone de Discord com salas públicas, nem depende de infraestrutura da casa. Sem telemetria. As únicas máquinas de terceiros que ele toca são os **relays públicos de descoberta/NAT-traversal** (ver §5.7) — equivalentes a STUN, não guardam mensagem. Não é multi-dispositivo no MVP (a chave = 1 dispositivo; multi-device fica pra depois).

## Princípios (herdados da filosofia — projetos.md §1)

- **100% offline por natureza:** não há "nuvem do Taylor". O que existe é rede direta entre os dois pares.
- **Sync é do usuário:** o histórico é um arquivo local; o app não sabe que Syncthing existe.
- **OpenSource na v1**, MIT; reusar OSS permissivo (evitar copyleft incompatível com MIT — ver §6.4 do projetos.md).
- **Sem servidor PRÓPRIO.** Usar relays públicos de terceiros (n0) pra hole-punching/fallback é aceitável e coerente — não é infra nossa, não vê conteúdo (tráfego é cifrado ponta a ponta). Self-host de relay é opção futura, não requisito.
- **Sem cadastro:** identidade é um par de chaves gerado no primeiro uso; pareamento é troca de chave fora de banda.

## Stack decidido

| Camada | Escolha | Por quê |
|---|---|---|
| App | **Tauri 2 + React 19 + Vite + TS** (front) + Rust (back) | padrão da suíte |
| Transporte P2P | **iroh** (`iroh` + `iroh-gossip` + `iroh-blobs`) | ver §5.1 |
| Cripto de mensagem | **QUIC/TLS do iroh** (transporte) + **double ratchet via `vodozemac`** (Apache-2.0) pro conteúdo | ver §5.3 |
| Histórico | **rusqlite (bundled)**, cifrado em repouso | mesmo padrão do LocalData |
| Compressão de arquivo | **`zstd`** antes do envio | §4.4 do projetos.md |
| QR / convite | `qrcode` (Rust, gera) + `jsQR`/webcam no front (lê) | pareamento fora de banda |
| IA | **llama.cpp sidecar** (porta 8103, ver §5.8) | padrão da suíte |

## Arquitetura

### 5.1 Transporte P2P — **iroh** (decidido)

Escolha entre as duas opções do esboço: **iroh vence libp2p** para este app.

- **iroh** (n0, Rust, QUIC): conexões diretas autenticadas por chave (`NodeId` = ed25519), **hole-punching** e relays de fallback embutidos, e — decisivo — traz de graça os dois protocolos que este app precisaria construir à mão:
  - **`iroh-blobs`**: transferência de conteúdo **endereçado por hash (BLAKE3), resumível em chunks** → é exatamente o "arquivos/mídia retomável" do §4.4, sem reinventar.
  - **`iroh-gossip`**: pub/sub leve pra presença e entrega de mensagens curtas.
- **libp2p**: mais ecossistema, porém mais peças pra montar (multiplexer, transport, discovery separados) e mais superfície de bug pra um app de 1 dev. Sem ganho que compense aqui.
- **Consequência:** o "transporte autenticado + cifrado" já vem pronto (QUIC/TLS com a chave do nó). A camada de aplicação por cima é o protocolo de mensagens (§5.5) e o ratchet (§5.3).

> A conexão iroh já é **mutuamente autenticada e cifrada** (cada ponta conhece o `NodeId` do outro). Isso sozinho já dá confidencialidade "na fita"; o ratchet (§5.3) acrescenta **sigilo futuro/forward secrecy** por mensagem.

### 5.2 Identidade e pareamento

- **Identidade = chave.** No primeiro uso o Rust gera o `iroh` secret key (ed25519); o `NodeId` público é a "identidade Taylor" do usuário. Guardada no **cofre do SO** (keyring: DPAPI no Windows, Secret Service no Linux) — mesmo padrão do token do LocalCode.
- **Pareamento fora de banda (sem servidor de descoberta de contatos):**
  - Cada usuário mostra um **convite** = `NodeId` + dica de relay + **segredo de pareamento efêmero** (nonce), codificado em texto curto e **QR code**.
  - O outro **escaneia o QR (webcam)** ou cola o código. Handshake: ambos provam posse da chave e confirmam o segredo → viram contatos. A verificação de identidade fica a cargo do canal fora de banda (mostrar o QR pessoalmente / mandar por outro meio) — sem CA, sem servidor.
  - Contato salvo = `{NodeId, apelido, chave pública, sessão do ratchet}`.
- **Descoberta de endereço** (achar por onde falar com um `NodeId` já conhecido): discovery do iroh (pkarr/DNS público do n0) + **mDNS na LAN** pra pares na mesma rede. Nada disso revela conteúdo nem lista de contatos.

### 5.3 Criptografia ponta a ponta

- **Duas camadas:** (1) QUIC/TLS do iroh entre `NodeId`s autenticados — cifra tudo no fio; (2) **double ratchet** (`vodozemac`, a impl. Rust auditada do olm/megolm do Matrix, Apache-2.0 → compatível com nosso MIT) sobre o conteúdo das mensagens → **forward secrecy + post-compromise security**.
- **Handshake inicial de sessão:** X3DH-like a partir das chaves trocadas no pareamento (§5.2) estabelece o estado do ratchet; daí cada mensagem avança a chave.
- **MVP pragmático:** se o ratchet atrasar o começo, a **camada (1) sozinha já entrega E2E** entre pares autenticados (é o piso aceitável); (2) entra logo em seguida como fase própria. Registrar isso pra não travar o MVP na cripto perfeita.
- **Repouso:** histórico e mídia cifrados no disco (ver §5.4).

### 5.4 Histórico e dados locais

- **rusqlite (bundled)**, um banco local por perfil (`chat.db` na config dir). Tabelas: `contatos`, `conversas`, `mensagens` (id, conversa, autor, corpo, ts, estado de entrega), `anexos` (hash BLAKE3 do iroh-blobs + metadados), `sessoes` (estado serializado do ratchet).
- **Cifrado em repouso:** SQLCipher (via `rusqlite` feature) **ou** cifra a nível de app com chave derivada da identidade guardada no keyring — decidir na implementação (SQLCipher é mais simples se a build fechar; senão, app-level). A chave nunca vai pro disco em claro.
- **Sync do usuário:** a pasta do perfil é o que ele joga no Syncthing/OneDrive. Como é 1 dispositivo por chave no MVP, sync = backup; multi-device real (mesma identidade em 2 máquinas) é problema separado (§9).

### 5.5 Protocolo de mensagens

- **Mensagem** = envelope JSON/CBOR pequeno: `{tipo, id, ts, corpo?, blobHash?, ...}` cifrado pelo ratchet e mandado pela conexão iroh (stream direto quando ambos online; `iroh-gossip` pra presença/ACK).
- **Tipos:** texto, anexo (aponta pra um blob), recibo (entregue/lido), presença (online/digitando), controle (rotação de chave). Extensível.
- **Estados de entrega:** enfileirada → enviada → entregue → lida. Enquanto o par está offline, fica `enfileirada` no SQLite e reenvia quando ele aparece (presença via gossip/discovery).

### 5.6 Arquivos e mídia

- **`zstd`** comprime o arquivo → entra no **`iroh-blobs`** (endereçado por hash, **transferência resumível em chunks**) → manda-se só o hash na mensagem; o par busca o blob. Retomar download interrompido é grátis (é o que o iroh-blobs faz).
- Miniaturas/preview gerados no front; mídia grande fica no blob store local, referenciada pela mensagem.
- Limite de tamanho configurável; nada de servidor de mídia (é P2P puro).

### 5.7 Entrega, presença e o problema do offline

- **MVP: só quando ambos online.** Coerente com "sem servidor próprio". Presença por discovery + gossip; se o par está fora, a mensagem espera no SQLite (§5.5).
- **NAT traversal:** hole-punching do iroh; quando falha, **relay público do n0** encaminha os bytes **já cifrados** (não vê conteúdo). Isso NÃO é servidor nosso e não guarda nada — é o "STUN/TURN" do iroh. Aceito pela filosofia.
- **Store-and-forward (FUTURO, não-MVP):** três caminhos possíveis, a decidir quando doer: (a) **relay opcional** que segura o pacote cifrado até o destino voltar (custa infra → só se houver receita); (b) **entre os próprios contatos** (um contato online repassa quando puder — sem infra, mas exige protocolo de roteamento); (c) **`iroh-docs`** como log replicado que sincroniza quando os pares se encontram. Registrar as três; nenhuma no MVP.

### 5.8 IA local

- **Sidecar llama.cpp** padrão da suíte (Vulkan + fallback CPU), só `127.0.0.1`, **porta 8103** (preferência; faixa 8103–8123 com busca de porta livre — ver mapa em §6). Zero telemetria.
- **Casos de uso (tudo local, sobre a conversa que já está na máquina):**
  - **Redigir/melhorar resposta** (tom, encurtar, formalizar) — como o bubble menu do Writer.
  - **Resumir conversa longa** (map-reduce, reusa o `chunkDocument` do Writer).
  - **Traduzir** mensagem recebida/enviada.
  - **Smart-compose / sugestões de resposta** curtas.
- **Regra da suíte mantida:** a IA nunca manda mensagem sozinha nem executa ação de rede; ela só devolve **texto proposto** que o usuário revisa e envia. Sem "agente" que fala pelos outros.

### 5.9 UI (React — layout de mensageiro)

- **Coluna de conversas** (lista de contatos com última mensagem + presença) | **painel da conversa** (bolhas, entrega/lido, digitando) | **composer** (texto, anexar arquivo, botão IA).
- **Tela de pareamento:** mostrar meu QR / escanear QR / colar convite.
- **Identidade Taylor** (§6.5 do projetos.md): ícone próprio, cor de domínio (mensageria — sugestão verde-água/roxo, definir no branding), "Taylor" cursivo como easter egg.
- Tema claro/escuro; sem StrictMode duplicando efeitos de rede (lição da suíte).

## 6. Mapa de portas (atualiza a tabela do projetos.md §1)

| App | Porta |
|---|---|
| Writer | 8088 fixa |
| Code | 8090–8099 |
| Sheets | 8099–8148 |
| Slides | 8100–8120 |
| LocalData | 8101–8121 |
| LocalPDF (planejado) | 8102 |
| **Taylor Chat (este)** | **8103–8123 (pref. 8103)** |

(As faixas se sobrepõem de propósito — todos buscam porta livre; o runtime de IA compartilhado, quando vier, unifica numa porta só. §6.1 do projetos.md.)

## 7. Fases

1. **Esqueleto + identidade:** app Tauri, geração de chave iroh no 1º uso, keyring, banco SQLite (cifrado). *Critério: abre, gera identidade estável, mostra meu `NodeId`/QR.*
2. **Pareamento:** gerar/mostrar QR e convite; escanear (webcam) / colar; virar contatos com sessão estabelecida. *Critério: duas máquinas viram contatos por QR, sem digitar nada de rede.*
3. **Mensagens de texto online:** conectar por iroh, mandar/receber texto, recibos entregue/lido, presença. *Critério: conversa de texto entre dois pares na mesma LAN e entre redes diferentes (via hole-punch/relay).*
4. **Cripto forte:** double ratchet (`vodozemac`) sobre o conteúdo; rotação de chave. *Critério: mensagens com forward secrecy; sessão sobrevive a reinício.*
5. **Arquivos/mídia:** zstd + iroh-blobs, transferência resumível, preview. *Critério: mandar um arquivo grande, cortar a conexão no meio, retomar e concluir.*
6. **IA local:** sidecar 8103; redigir/resumir/traduzir sobre a conversa. *Critério: "resumir esta conversa" e "melhorar minha resposta" funcionam offline.*
7. **Fila offline + polimento:** reenvio quando o par volta, estados de entrega robustos, backup, UI. *Critério: mandar pra par offline, ele recebe ao voltar (ambos precisando ficar online em algum momento).*
8. **(Futuro, pós-v1)** store-and-forward de verdade (§5.7), multi-device, self-host de relay. Cada um vira seu próprio mini-plano quando doer.

## 8. Decisões tomadas neste plano

- **iroh** como transporte (não libp2p) — traz blobs resumíveis e gossip de graça (§5.1).
- Nome **TaylorChat** (não `Local*` — mensageiro não é "local"); pasta `chat/`, repo `Anon5T4R/TaylorChat`, **MIT**.
- Cripto: QUIC/TLS do iroh (piso E2E) + **double ratchet `vodozemac`** (Apache-2.0, compatível MIT) como camada de conteúdo — ratchet pode entrar como fase 4, sem travar o MVP.
- Histórico em **rusqlite bundled** cifrado; identidade no **keyring do SO**.
- **Relays públicos do n0 são OK** (NAT traversal, não veem conteúdo, não são infra nossa). Store-and-forward fica pra depois.
- **Porta de IA: 8103** (pref.), faixa 8103–8123.
- MVP **1 dispositivo por identidade**; multi-device e store-and-forward são fases futuras explícitas.
- IA **só propõe texto**, nunca envia nem age na rede (regra da suíte).

## 9. Riscos e perguntas em aberto (a resolver na implementação)

- **Multi-device / sync da identidade:** a mesma chave em 2 máquinas via Syncthing quebra o ratchet (estado diverge). Solução real é protocolo de dispositivos (tipo Signal) — **fora do MVP**. Enquanto isso: 1 chave = 1 device, sync é só backup do histórico.
- **SQLCipher × cripto app-level:** decidir qual fecha a build mais limpa no Tauri/Windows+Linux (SQLCipher às vezes complica o bundle do rusqlite).
- **Estabilidade das APIs do iroh:** iroh e os protocolos (`iroh-blobs`/`iroh-gossip`) evoluem rápido; fixar versões e revisar antes de começar (o retrato aqui pode envelhecer).
- **Verificação de identidade sem servidor:** o pareamento por QR presencial é forte; convite por texto por outro canal herda a segurança daquele canal. Documentar isso pro usuário (não prometer mais do que dá).
- **Recibos de "lido" e privacidade:** deixar desligável (padrão da categoria).

## 10. CI / Release (padrão da suíte — §6.6 do projetos.md)

- Tag `v*` → matriz Windows (NSIS) + Ubuntu 22.04 (AppImage) via `tauri-action`.
- Sidecar llama.cpp baixado no CI por `scripts/fetch-llama.{ps1,sh}`; `binaries/llama/placeholder.txt` versionado (gotcha de suíte).
- Instaladores não assinados (SmartScreen avisa uma vez); assinatura fica pra quando houver receita.
- **Entra no catálogo do Hub** com 1 entrada JSON quando tiver release (nasce já com `fileAssociations` se fizer sentido — provavelmente nenhuma extensão associada, como o TaylorAI Studio). Nota do hub.md: apps novos já nascem no padrão de CI/catálogo.

## 11. ESTADO da implementação (2026-07-07) — `chat/` v0.1.0-dev

Scaffold criado e **Fases 1–6 codadas**. Front build verde; `cargo test` (default, 4 testes) e `cargo test --features p2p` (5 testes) verdes; `tauri dev --features p2p` boota o app + endpoint iroh sem panic. Estrutura no padrão da suíte (copiada de `hub/`+`data/`), pasta `chat/`. Falta o teste ponta a ponta (2 instâncias) e testar a IA com um modelo real.

**Feito:**
- **Identidade** (`src-tauri/src/identity.rs`): par ed25519 (`ed25519-dalek`) gerado no 1º uso, segredo no **keyring** do SO; `node_id` = hex das 32 bytes (mesmos bytes do `NodeId` do iroh).
- **Banco** (`db.rs`): SQLite `bundled`, tabelas `contacts`/`messages`/`sessions`/`_meta`, CRUD + `enqueue`/`set_state`/`record_incoming`. **Corpo das mensagens e pickles do ratchet cifrados em repouso** (Fase 4, ver abaixo).
- **Pareamento** (`pairing.rs`): `my_identity` (id + convite `taylorchat:<hex>` + **QR em SVG** via crate `qrcode`) e `parse_invite`. UI: modal com meu QR/código + adicionar por convite/apelido.
- **UI** (React, `src/`): sidebar de contatos, painel de conversa (bolhas, estados de entrega, autoscroll), composer (Enter envia). Tema claro/escuro. Sem StrictMode.
- **IA** (`llm.rs`): sidecar llama.cpp na **porta 8103** (mesmo motor da suíte) — comandos prontos; UI de IA é Fase 6.
- **CI/Release**: workflows no padrão; release ainda builda o **default** (sem p2p) pra ficar verde.

**Fase 3 — rede (iroh):** `src-tauri/src/net.rs` **atrás da feature `p2p`** (não-default). API do **iroh 0.35** (endpoint + ALPN `taylorchat/msg/0`; handshake e mensagens em **streams bidirecionais**; emite evento `message-in`). `cargo check`/`cargo test --features p2p` verdes e o app **boota** com o endpoint no ar (node id = a chave da identidade). Bug de runtime resolvido: usa `tauri::async_runtime` (não tokio à parte); `send_text`/`send_message` são async. Sem a feature: app roda, pareia, guarda histórico; mensagens de saída ficam `queued`.

**Fase 4 — cripto (FEITO, testado):**
- **Cifra em repouso** (`crypto.rs`, sempre ligada): AEAD **XChaCha20-Poly1305**, chave derivada (HKDF-SHA256) do segredo da identidade (nunca em claro no disco). Corpo das mensagens vira BLOB cifrado; pickles do ratchet idem (`_meta`/`sessions`). Metadados (ts/peer/estado) em claro por ora — cifra do arquivo inteiro (SQLCipher) fica como opção futura. **2 testes vitest-Rust passam** (roundtrip + chave errada + derivação determinística).
- **Double ratchet** (`ratchet.rs`, feature p2p): **vodozemac 0.8** (Olm do Matrix, Apache-2.0). Núcleo puro e testável: **1 teste passa** — X3DH (bundle de pré-chave) + troca ida-e-volta com avanço do ratchet + sobrevivência a pickle/restore. Integrado no `net.rs`: 1ª mensagem pra um par faz `req_prekey → bundle → msg_prekey` (X3DH sobre o canal iroh já autenticado), depois usa a sessão; estado persistido cifrado. **Forward secrecy no conteúdo.**

**Fase 5 — arquivos/mídia (FEITO, testado):**
- `media.rs` (sempre): **zstd** (`compress`/`decompress`) + `guess_mime` + `save_attachment` (grava na pasta `attachments` do app). **2 testes passam** (pipeline comprime→cifra→decifra→descomprime; mime por extensão).
- **Transferência** (net.rs, p2p): o protocolo virou unificado — o conteúdo interno é `{k:"text"|"file"}` cifrado pelo ratchet; anexo vai **comprimido (zstd) + cifrado com chave de uso único** (a chave viaja dentro do JSON cifrado), os bytes seguem o cabeçalho no mesmo stream bi. O receptor manda um **ACK** antes do fecho (confirma processamento; base pra "entregue"). Limite 100 MiB/anexo neste corte; **retomada via iroh-blobs fica como upgrade** (§5.6).
- Comando `attach_file` (lib.rs): lê o arquivo, guarda cópia local, registra msg `file` e transfere. UI: botão 📎 no composer (diálogo nativo) + bolha de anexo com nome/tamanho e "abrir" (plugin-opener).
- DB: coluna `kind` (`text`/`file`) com **migração** pra bancos das fases anteriores; corpo `file` = JSON `{filename,mime,size,localPath}` (cifrado em repouso como o texto).

**Fase 6 — IA local (FEITA):**
- Sidecar llama.cpp já vivia no `llm.rs` (porta 8103, `list_models`/`start_llm`/`stop_llm`/`llm_status`, padrão da suíte). Fase 6 = o cliente + a UI.
- `src/lib/ai.ts`: ciclo de vida do sidecar (via `invoke`, args camelCase `modelPath`/`nGpuLayers`/`ctxSize`) + `chat()` que faz `fetch` no servidor local OpenAI-compat (`127.0.0.1:porta/v1/chat/completions`, think OFF). Ações: **sugerir resposta, resumir conversa, melhorar rascunho, traduzir** (transcript da conversa como contexto).
- `AiPanel.tsx`: terceira coluna (botão **✦ IA** no topo da conversa) — escolher pasta de modelos (diálogo nativo), dropdown de `.gguf`, iniciar/parar (CPU, ctx 4096), status; resultado editável com **"Usar no rascunho"**/copiar. **Regra da suíte mantida: a IA só PROPÕE texto**, nunca envia. Front build verde.
- **Limite honesto:** inferência real precisa do `llama-server` (baixado por `scripts/fetch-llama`) + um `.gguf` na máquina — não dá pra exercitar aqui, mas a fiação é a mesma (comprovada) dos irmãos da suíte.

**Refinos de fechamento do MVP (FEITO 2026-07-07):**
- **Recibos:** o ACK da rede agora marca a mensagem como **`delivered`** (✓✓); ao abrir uma conversa a UI manda um **recibo de leitura** (`mark_read` → conteúdo `{k:"read"}`), o par recebe (evento `receipts`) e minhas mensagens viram **`read`** (✓✓ coloridos). `db::mark_out_read` atualiza em lote.
- **UX de mensageiro:** **badges de não-lido** por contato (some ao abrir), **ordenação da lista por atividade** recente, e o listener trata mensagens de contatos não abertos (antes sumiam da UI).
- **Gerenciar contato:** renomear (✏️) e remover (🗑) na barra da conversa (usa `contact_add`/`contact_remove`).

**Próximos passos (operacionais, não de código):**
1. **Teste de 2 instâncias** trocando texto E arquivo real (LAN e entre redes) com `--features p2p` — tudo compila, testa e boota; falta exercitar handshake+transferência+recibos ponta a ponta (não dá com 1 perfil só).
2. Testar a **IA** de verdade com um `.gguf` na máquina (baixar o `llama-server` antes).
3. Upgrade da transferência pra **iroh-blobs** (retomável) e ligar `--features p2p` no CI/release depois do teste ponta a ponta.
4. ~~Criar `Anon5T4R/TaylorChat` + 1º release + catálogo do Hub~~ **FEITO (2026-07-07/08):** repo público criado, **release v0.1.0 publicado** (Windows `_x64-setup.exe` + Linux `_amd64.AppImage`, build com `--features p2p`), e TaylorChat adicionado ao catálogo do TaylorHub (**Hub v0.3.3**, 9º app).

**v0.1.1 — refino de UX (2026-07-08):** drag & drop de arquivos na conversa + **preview de imagem inline** (asset protocol, feature `protocol-asset` + `assetProtocol.scope=$APPDATA/attachments`); **reenvio automático** da fila (`resend_queued`) quando o par volta/ao abrir a conversa; sidebar com **prévia da última msg + hora + não-lidos** (`conversations_summary`), ordenada por atividade; separadores de data (Hoje/Ontem) na conversa; avatar com cor única por contato; textarea que cresce; Esc fecha o modal de pareamento + nota explicando o QR; estados de entrega ✓/✓✓/lido + `failed`.

**+ Arquivos grandes (mesma v0.1.1):** transferência de anexo virou **streaming por chunks** (1 MiB): cada pedaço é comprimido+cifrado e enviado; o receptor grava incremental no disco. Nem remetente nem destinatário seguram o arquivo inteiro em RAM → **sem teto de tamanho** (tirado o limite de 100 MB). `net.rs` refatorado: `send_header` (handshake+cabeçalho) reusado por texto/arquivo/recibo; `send_file` lê do disco e manda chunks até um frame vazio (fim); cópia local via `fs::copy` (streaming).

**v0.1.3 — contatos/persistência/auditoria/verificação/i18n (2026-07-08):**
- **Auto-salvar contato** ao receber de peer novo (`contact_ensure` no `insert_message` quando `direction=in`) — antes a conversa ficava invisível se você não tinha adicionado o outro. Contatos e mensagens já persistiam no SQLite; o buraco era esse.
- **ts do remetente transmitido** e gravado nos 2 lados (inner JSON ganhou `ts`; `record_incoming`/`record_file` recebem ts) — corrige a hora das recebidas E é a base da auditoria.
- **Auditar** (`audit_conversation` → `db::audit_digest`): SHA-256 sobre as mensagens normalizadas (autor=quem enviou, ts, tipo, conteúdo; arquivo = nome+tamanho, sem o caminho local), ordenadas de forma determinística. Os 2 dispositivos geram o MESMO digest se os registros baterem; divergência = adulteração. Fica no painel de **Configurações**. (Não-repúdio forte com assinatura por-mensagem fica como upgrade; hoje é evidência-de-adulteração entre as 2 partes honestas.)
- **Palavra-chave por contato** (anti-MITM): cada um define a palavra combinada fora do app; o app troca só o **hash** (SHA-256) pelo canal E2E (`{k:"keyword"}`), compara e mostra confere/diverge (🔑 verde/âmbar + banner). **Não bloqueia** a conversa. Guardada cifrada em repouso (`contacts.kw`); hash do par em `contacts.peer_kw_hash`. Pega pareamento com identidade errada (impostor não sabe a palavra).
- **Configurações** (engrenagem na sidebar): tema (sistema/claro/escuro via `data-theme` + CSS), idioma, **toggle de recibo de leitura** (gate no `markRead`), e o Auditar.
- **i18n pt/es/en** em toda a UI (`src/lib/i18n.ts`, `t()`; troca ao vivo re-renderiza via `key={lang}`).
- Testes: 6 default / 7 p2p verdes, front build verde, app boota.

**+ Transferência RETOMÁVEL (2026-07-08):** decidido **NÃO usar iroh-blobs** — ele endereça por hash do conteúdo em CLARO; pra manter o E2E eu teria que blobar o ciphertext (anula dedup/content-addressing), então o único ganho real (download retomável) eu fiz no próprio protocolo, sem o refactor do Router nem a API instável do iroh-blobs. Como: cada anexo tem `transferId` estável + `fileKey` reusado (guardados no meta local, cifrado em repouso); o receptor mantém `attachments/.partial/<id>` e, ao (re)conectar, responde de qual **chunk** retomar (alinhado à fronteira de chunk); o remetente faz `seek` e manda daí. Sobrevive a queda no meio E a reinício dos dois lados; `resend_queued` reusa transferId/fileKey do meta. Testes: 7 p2p / 6 default (+`retomada_reconstroi_apos_queda_no_meio`), front build verde, app boota. iroh-blobs fica só se um dia quiser dedup entre arquivos SEM E2E.

**Pendências herdadas:** repo `Anon5T4R/TaylorChat` a criar + primeiro release pra entrar no catálogo do Hub. Ícone próprio feito (`icons/source-taylorchat.svg`; placeholders do LocalData removidos).
