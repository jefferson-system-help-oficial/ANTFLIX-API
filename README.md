# 📡 ANTFLIX-API

> **Camada de dados remotos do ANTFLIX Desktop** — addons, catálogos kids, mensagens e versões do app, servidos via GitHub como API somente-leitura.

[![Version](https://img.shields.io/badge/version-0.0.1--beta-red?style=flat-square)](versions/app.json)
[![Format](https://img.shields.io/badge/format-JSON-blue?style=flat-square)](#-estrutura-do-repositório)
[![Mode](https://img.shields.io/badge/mode-read--only-green?style=flat-square)](#-como-o-app-consome)
[![Manager](https://img.shields.io/badge/write-manager--only-orange?style=flat-square)](#-como-publicar-gerenciador)

---

## 🎯 Visão geral

Este repositório **não contém código** — apenas `JSON` e `Markdown` curados. Ele funciona como um **CMS remoto barato e versionado**:

| Consumidor | Permissão | Como |
|---|---|---|
| 🖥️ **ANTFLIX Desktop** | 🔒 Somente leitura | `GET raw.githubusercontent.com` + `ETag` / cache local |
| 🛠️ **App Gerenciador** (projeto separado) | ✏️ Leitura + escrita | GitHub Contents API + validação por schema |

**Princípio de ouro:** o que vem do GitHub é tratado como *"vindo do servidor"*; o que o usuário alterou localmente (`desligou`, `apagou`, `liberou`) **sempre vence** e nunca é sobrescrito.

---

## 🗂️ Estrutura do repositório

```text
ANTFLIX-API/
├── versions/
│   └── app.json                  # Versão atual do app + changelog + download
├── addons/
│   ├── index.json                # Mapa leve: 1 request diz o que mudou
│   ├── big.json                  # Modelo de exemplo (duplique para criar)
│   ├── streaming-catalogs.json
│   ├── cinemeta.json
│   ├── zeus.json / unioflix.json / froststream.json
│   ├── fenixflix.json / bestcine.json / ...
│   └── youtube.json / kitsu.json / anizip.json / tmdb.json
├── kids/
│   ├── rules.json                # Tentativas, bloqueio, tamanho do PIN
│   └── catalogos/
│       ├── pre-escolar.json      # Pack 0–6 anos
│       └── herois.json           # Pack 4–10 anos
├── messages/
│   └── ativos.json               # Avisos estilo "app profissional"
├── schemas/
│   ├── addon.schema.json
│   ├── pack.schema.json
│   └── message.schema.json
└── README.md
```

---

## 📦 Formatos

### 1. `versions/app.json` — versão do app

```json
{
  "version": "0.0.1-beta",
  "minSupported": "0.0.1-beta",
  "changelog": ["Lançamento beta do ANTFLIX Desktop."],
  "downloadUrl": ""
}
```

> O app compara `version` remota × local e exibe aviso de atualização.

### 2. `addons/index.json` — mapa barato de mudanças

O app baixa **só este arquivo** (com `If-None-Match`) para saber o que mudou. Só então baixa o `*.json` individual.

```json
{
  "generatedAt": "2026-10-14T00:00:00Z",
  "addons": [
    { "id": "com.fenixflix", "file": "fenixflix.json", "version": "1.0.0", "updatedAt": "..." }
  ]
}
```

### 3. `addons/<nome>.json` — um arquivo por addon

```json
{
  "id": "com.fenixflix",
  "name": "FenixFlix",
  "kind": "stream",
  "manifestUrl": "https://fenixflix.fenixhub.online/manifest.json",
  "manifestSnapshot": { "id": "com.fenixflix", "name": "FenixFlix", "version": "1.0.0" },
  "version": "1.0.0",
  "changelog": ["Fonte de streams dublado/legendado."],
  "enabled": true,
  "priority": 50,
  "timeoutMs": 8000,
  "updatedAt": "2026-10-14T00:00:00Z"
}
```

| Campo | Efeito no app |
|---|---|
| `version` ↑ | 🔔 Toast discreta *"Atualização no FenixFlix"* + update em background |
| `enabled: false` | ⛔ Kill-switch remoto — addon sai da fila em horas, sem rebuild |
| `priority` | Ordem na fila de streams/catálogos |
| `manifestUrl` | Revalidação do manifesto original |

### 4. `kids/rules.json` + `kids/catalogos/*.json` — modo kids

```json
// rules.json
{ "maxFails": 3, "lockMs": 60000, "pinLength": 4 }
```

```json
// catalogos/pre-escolar.json
{
  "id": "pre-escolar",
  "name": "Pré-escolar",
  "faixa": "0-6 anos",
  "version": "1.0.0",
  "items": [{ "id": "buscar:Bluey", "name": "Bluey", "type": "series" }]
}
```

> `buscar:Nome` é resolvido ao vivo contra o catálogo real — **nenhum ID é inventado**. O adulto libera com PIN e o item entra na allowlist local existente.

### 5. `messages/ativos.json` — avisos profissionais

```json
{
  "messages": [
    {
      "id": "boas-vindas-beta",
      "tipo": "info",
      "titulo": "Bem-vindo ao beta",
      "texto": "Conecte as fontes em Administração → Fontes.",
      "de": "2026-10-14T00:00:00Z",
      "ate": null,
      "kidsSafe": true,
      "acao": null
    }
  ]
}
```

| `tipo` | Uso |
|---|---|
| `info` | Novidade / dica |
| `aviso` | Atenção (ex: fonte instável) |
| `manutencao` | Janela de manutenção |

> `kidsSafe: false` **nunca** aparece no perfil infantil. Cada `id` é exibido 1×.

---

## 🔄 Como o app consome

```mermaid
index.json (ETag) → mudou? → baixa *.json → valida → merge → SQLite
```

1. **Poll leve** a cada ~6h (ou botão *Atualizar*): `GET index.json` com `If-None-Match`.
2. **Download seletivo**: só arquivos com `version` maior que `remote_version` local.
3. **Merge seguro** (nunca destrutivo):
   - ➕ Remoto novo → adiciona com `origin = remote`.
   - 🔌 Remoto `enabled: false` → desliga, preserva config local.
   - 🙈 Usuário desligou/apagou (`local_disabled` / `local_deleted`) → remoto **não ressuscita**.
   - ⚙️ Config local sempre vence default remoto.
4. **Offline-first**: sem rede, o app serve 100% do SQLite — zero tela de erro.

---

## 🛠️ Como publicar (gerenciador)

> O ANTFLIX Desktop **nunca escreve** aqui. Só o app gerenciador commita.

1. Edite o JSON no gerenciador (com preview).
2. Valide contra `schemas/*.schema.json`.
3. Bump de `version` + entrada no `changelog` + `updatedAt` ISO-8601.
4. Commit via Contents API (`PUT /repos/{owner}/{repo}/contents/{path}` com `sha`).
5. Regenere `addons/index.json` (`generatedAt` = agora).

**Para adicionar um addon novo:** duplique `addons/big.json`, preencha, adicione 1 linha no `index.json`.

---

## 🔐 Segurança

1. 🚫 **Nunca** commitar token, PAT, PIN, hash, salt ou senha aqui.
2. 🔑 Token de leitura (se o repo for privado): PAT fine-grained `contents:read` **só neste repo**, via `.env` (`GITHUB_READ_TOKEN`) — nunca bundlado no `dist/`.
3. ✏️ Token de escrita vive **server-side, só no gerenciador** (`contents:write`).
4. 🌍 Recomendado: repo **público** = zero token, `raw.githubusercontent.com` direto, rate-limit 60/h por IP (suficiente com ETag + cache).

---

## 🔢 Versionamento

- **App**: `versions/app.json` — `semver` + sufixo (`0.0.1-beta`). `minSupported` força update.
- **Addon**: `version` dentro de cada `addons/*.json` — bump + `changelog` + `updatedAt`.
- **Pack kids / mensagem**: `version` ou novo `id` (mensagem nova = `id` novo).

---

## 🤝 Contribuindo

1. Valide o JSON (`node -e "JSON.parse(...)"` ou o gerenciador).
2. 1 addon = 1 arquivo + 1 linha no `index.json`.
3. Teste o manifesto real antes de publicar (`/manifest.json` responde `id` + `name`?).
4. Descreva no `changelog` o que mudou para o usuário final.

---

## 📄 Licença

Dados de configuração © ANTFLIX. Manifestos de terceiros pertencem aos seus autores.

> Dúvidas? Abra uma issue em [ANTFLIX-API](https://github.com/jefferson-system-help-oficial/ANTFLIX-API).
