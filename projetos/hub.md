# TaylorHub — plano de implementação

> Plano de 2026-07-06. Complementa o §5 do [projetos.md](projetos.md). **Decisão: o Hub é dirigido por catálogo** — adicionar app novo = 1 entrada num JSON, zero código, zero rebuild. Por isso ele pode ser feito AGORA, antes de LocalData/LocalPDF existirem: eles entram no catálogo no dia em que tiverem release.

## ESTADO (2026-07-06): Fases 1 e 2 IMPLEMENTADAS em `hub/` (v0.1.0)

Repo: https://github.com/Anon5T4R/TaylorHub (público, MIT). Build TS + `cargo check` + testes verdes; UI verificada no preview do Vite. **Falta testar clique-a-clique no app Tauri de verdade** (instalar um app real, aplicar associações, duplo-clique num arquivo) — o assistente não consegue manter janela GUI viva.

Diferenças do plano original (decididas na implementação):
- **Windows E Linux desde o dia 1** (pedido do João) — install AppImage → `~/Applications` + `.desktop`; associações via mime/xdg. A fase "Linux depois" caiu.
- **macOS: fica de fora por enquanto** — NENHUM app da suíte tem build de Mac hoje (verificado nas releases: todos só têm `.exe` + `.AppImage`/`.deb`). ⚠️ **PENDÊNCIA ANOTADA: gerar builds macOS dos apps (e do Hub) — resolver em sessão futura, só quando o João pedir explicitamente.**
- Detecção de instalado no Windows: **varre TODOS os itens de Uninstall casando por DisplayName** (Tauri NSIS e electron-builder usam chaves diferentes; casar pelo nome é robusto pros dois).
- Catálogo por ora **embutido** em `src/catalog.ts` (fase 3 = mover pra `catalog.json` remoto com cache).
- Porta dev do Vite: **1425** (writer/sheets/slides usam 1420).
- TaylorMind: assets da release v0.1.1 se chamam `0.1.0` → glob `TaylorMind.Setup.*.exe` resolve, mas arrumar na próxima release do TaylorMind.

## O que o Hub é (e não é)

**É:** um app pequeno (Tauri 2 + React, mesmo stack da suíte, pasta `hub/` no Office) com três funções:
1. **Instalar/atualizar** os apps da suíte a partir das releases do GitHub (download + instalação silenciosa).
2. **Associar tipos de arquivo** e **despachar**: clicar num arquivo abre o app certo automaticamente.
3. Tela mínima com a grade de apps (instalar / abrir / atualizar). Não é dock nem launcher elaborado.

**Não é:** loja com conta/login, updater em background agressivo, nem substituto dos instaladores individuais (cada app continua instalável sozinho).

**Encaixe futuro já previsto no design:** gerenciador de modelos GGUF (a monetização) e o runtime de IA compartilhado entram depois como itens do MESMO catálogo (`kind: "model"` / `"service"`), sem mudar a arquitetura.

## Pré-requisito — VERIFICADO OK (2026-07-06)

Todos os 6 apps existentes têm release pública com instalador Windows:

| App | Release | Asset Windows | Instalador |
|---|---|---|---|
| LocalOffice | v0.14.6 | `LocalOffice_0.14.6_x64-setup.exe` | NSIS (Tauri) |
| LocalSheets | v0.4.0 | `LocalSheets_0.4.0_x64-setup.exe` | NSIS (Tauri) |
| LocalSlides | v0.5.1 | `LocalSlides_0.5.1_x64-setup.exe` | NSIS (Tauri) |
| LocalCode | v0.7.1 | `LocalCode_0.7.1_x64-setup.exe` | NSIS (Tauri) |
| TaylorMind | v0.1.1 | `TaylorMind.Setup.0.1.0.exe` ⚠️ | NSIS (electron-builder) |
| OpenObsidian | v0.7.1 | `OpenObsidian.Setup.0.7.1.exe` | NSIS (electron-builder) |

Ambos os tipos de NSIS aceitam instalação silenciosa (`/S`). ⚠️ TaylorMind: a tag é v0.1.1 mas os assets se chamam 0.1.0 — por isso o padrão de asset no catálogo usa glob, não a versão; ainda assim vale arrumar na próxima release do TaylorMind.

## Arquitetura

### 1. Catálogo (`catalog.json`) — o coração

Um JSON com a lista de apps. O Hub embute uma cópia (funciona offline/primeira execução) e busca a versão mais nova de uma URL raw do GitHub (repo do Hub) com cache local — **app novo aparece pra todo mundo sem atualizar o Hub**.

```json
{
  "version": 1,
  "apps": [
    {
      "id": "writer",
      "name": "LocalOffice",
      "description": "Documentos (Word) com IA local",
      "kind": "app",
      "repo": "Anon5T4R/LocalOffice",
      "assets": { "win": "*_x64-setup.exe", "linux": "*_amd64.AppImage" },
      "silentArgs": ["/S"],
      "uninstallKey": "LocalOffice",
      "exe": "LocalOffice.exe",
      "extensions": ["md", "markdown", "txt", "docx", "odt", "rtf", "html", "htm"]
    }
  ]
}
```

Campos por app: `id`, `name`, `description`, `kind` (`app` | futuro `model`/`service`), `repo` (GitHub), `assets` (glob por plataforma), `silentArgs`, `uninstallKey` (nome da chave no registro p/ detectar instalado+versão), `exe` (pra abrir/despachar), `extensions` (o que ele atende). Ícone: baixado do repo ou embutido no catálogo.

**Adicionar LocalData/LocalPDF/Taylor Chat no futuro = colar um bloco desses.** É esse o critério de extensibilidade atendido.

### 2. Instalar / detectar / atualizar (Rust)

- **Detectar instalado + versão:** ler `HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall\<uninstallKey>` (tanto Tauri NSIS quanto electron-builder gravam `DisplayVersion`, `InstallLocation`, `DisplayIcon`). Fallback: `HKLM` (instalação por máquina).
- **Última versão:** `GET api.github.com/repos/<repo>/releases/latest` (sem token; rate limit de 60/h anônimo é suficiente — 1 chamada por app, com cache). Comparar com `DisplayVersion` → badge "atualizar".
- **Instalar/atualizar:** baixar o asset que casa com o glob → salvar em `%LOCALAPPDATA%\TaylorHub\downloads` → rodar com `/S` → aguardar exit code → reler registro → **reaplicar associações** (ver §3, porque o instalador do app acabou de sobrescrevê-las).
- Progresso de download via evento Tauri (stream com `reqwest`).
- **Nota SmartScreen:** os instaladores não são assinados. Quem baixa o Hub manualmente vê o aviso UMA vez; os apps instalados PELO Hub não passam pelo prompt do Explorer (execução via processo, não via duplo-clique do shell). Esse é, inclusive, o argumento de venda do Hub ("instalação facilitada").

### 3. Associações de arquivo — modelo dispatcher

O Hub se registra como handler e **despacha** pro app certo:

- Pra cada extensão do catálogo: criar ProgID `Taylor.<ext>` em `HKCU\Software\Classes` com `shell\open\command` = `"taylorhub.exe" --open "%1"` e `DefaultIcon` apontando pro exe do app de destino (ícone certo por tipo). Setar `HKCU\Software\Classes\.<ext>\(Default)` = `Taylor.<ext>`.
- No modo `--open`: Hub lê a **tabela de rotas** (settings: extensão → app id), resolve o exe via registro (`InstallLocation`) e faz `spawn(exe, [arquivo])`. Sem janela (ou splash de 200ms). Se o app não estiver instalado → abre o Hub na tela de instalação daquele app.
- **Por que dispatcher e não registrar o exe do app direto:** trocar a rota (ex.: `md` → Writer ou Code) vira um clique na UI, sem tocar registro; e o conflito atual Writer×Code por `md`/`html` fica resolvido POR USUÁRIO, no Hub. Default proposto: documento (md/html/docx) → Writer.
- **Realidade do Windows 10/11:** o `UserChoice` é protegido por hash — pra extensões que JÁ têm dono (md aberto pelo VS Code, pdf pelo Edge), o Windows pode mostrar o diálogo "Como você quer abrir?" uma vez; o Taylor.<ext> estará na lista e a escolha persiste. Pra extensões novas (`.tslides`, `.tmind`) funciona direto. É o comportamento padrão do ecossistema — aceitar.
- Botão **"Reparar associações"** na UI (re-registra tudo; também roda automaticamente após cada install/update).

### 4. UI (React — uma tela)

Grade de cards (ícone, nome, descrição, versão instalada × última): botões **Instalar / Abrir / Atualizar**. Segunda aba "Arquivos": tabela extensão → app (dropdown com os apps que declaram a extensão) + Reparar. Configurações mínimas (tema, catálogo remoto on/off). Identidade Taylor.

### 5. Auto-update do próprio Hub

Fase 3: o Hub se trata como mais um item do catálogo (checa a própria release, baixa e roda o setup novo). Simples, sem tauri-updater/assinatura por ora.

## Fases

1. **MVP (Windows):** catálogo embutido com os 6 apps; detectar instalados (registro); instalar/atualizar silencioso; abrir app; UI da grade. *Critério: instalar LocalSheets do zero e atualizar um app desatualizado, pelo Hub.*
2. **Dispatcher:** ProgIDs + `--open` + tabela de rotas + reaplicar pós-install + "Reparar associações". *Critério: duplo-clique num `.tslides` e num `.md` abre o app certo; trocar rota do `md` pro Code funciona sem re-registrar na mão.*
3. **Catálogo remoto + auto-update do Hub:** fetch com cache do `catalog.json` no repo do Hub; Hub se atualiza. *Critério: adicionar um app fake no catálogo remoto e vê-lo aparecer sem rebuild.*
4. **Linux:** baixar AppImage pra `~/Applications`, `chmod +x`, gerar `.desktop` + MIME (`xdg-mime`). Dispatcher via .desktop do Hub.
5. **(Futuro, pós-suíte)** `kind: "model"` (gerenciador/loja de modelos GGUF — pasta única, aponta os apps pra ela) e `kind: "service"` (runtime de IA compartilhado). Só entra quando existir.

## Decisões tomadas neste plano

- Hub em Tauri 2 (padrão da suíte), pasta `hub/` no Office, repo próprio `Anon5T4R/TaylorHub`, MIT.
- Catálogo-driven desde o dia 1 (a condição do João pra começar já).
- Dispatcher próprio em vez de associação direta ao exe do app.
- Windows primeiro; Linux na fase 4.
- Sem telemetria; as únicas chamadas de rede são GitHub (releases + catálogo), disparadas por ação do usuário ou abertura do Hub.

## Backlog (pedidos do João em 2026-07-06, após testar a v0.1.0)

- ~~**Ícones reais nos cards**~~ FEITO (v0.2.0): `iconUrl` do catálogo com fallback pra letra.
- ~~**Auto-update do Hub, só com consentimento**~~ FEITO (v0.2.0): banner "Nova versão do Hub disponível" + botão; nunca atualiza sozinho.
- ~~**Aba de arquivos: recentes/favoritos**~~ FEITO (v0.2.0): dispatcher loga em `recents.json` (config dir; cap 40 não fixados); aba "Recentes" com Fixados (★) + Recentes, abre pelo app da rota atual, fixar/remover/limpar. Aba "Arquivos" renomeada pra "Associações". Nota: só entra na lista o que passa pelo dispatcher (duplo-clique via associação) — arquivo aberto de dentro do app não aparece. Sem file-manager (organizar/mover fica pro Explorer/Nautilus).
- **Pasta de vaults do OpenObsidian** — a fazer, **investigado em 2026-07-06**: o OpenObsidian NÃO lê `process.argv` (vault só abre por diálogo, Ctrl+Shift+O; o último vault fica em `app-settings.json` no userData e reabre sozinho). **Pré-requisito: adicionar suporte a CLI no OpenObsidian** (~15 linhas em `src/main/index.ts`: argv[1] é pasta → abrir como vault + tratar `second-instance`). Depois disso o Hub lista as subpastas da pasta de vaults configurada e lança `OpenObsidian <caminho>`. Evitar a gambiarra de escrever `lastVault` no app-settings.json por fora (frágil, só com o app fechado).

## Pendências fora do Hub (arrumar nos apps, sem pressa)

- TaylorMind: nome dos assets desalinhado com a tag (0.1.0 × v0.1.1).
- Quando LocalData/LocalPDF nascerem: já nascer com `fileAssociations` + release CI no padrão, e adicionar ao catálogo.
