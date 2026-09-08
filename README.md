Vou reler o README inteiro do Tyny Pulse para reescrever sem perder fatos e sem inventar feature.Segue um README reescrito em inglês (o idioma do repo), sem inventar docs ou features. O binário headless fica como `tyny-cli` — o texto antigo misturava isso com `tyny-pulse run`.

```markdown
# Tyny Pulse

![Tauri](https://img.shields.io/badge/Tauri-v2-24C8D5?style=for-the-badge&logo=tauri&logoColor=white)
![Rust](https://img.shields.io/badge/Rust-2021-000000?style=for-the-badge&logo=rust&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?style=for-the-badge&logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5.8-style=for-the-badge&logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-7.0-646CFF?style=for-the-badge&logo=vite&logoColor=white)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

Local-first API client for Windows, macOS and Linux. Built with **Tauri v2**, **Rust** and **React 19**.

Workspaces live as plain JSON on disk. Secrets stay in a local AES-256-GCM vault. The same Rust runner powers the desktop UI and the headless CLI.

Site: [https://tyny.ca](https://tyny.ca)

> Add a screenshot or short GIF of the main window here. A desktop client without a preview wastes the first scroll.

---

## Table of Contents

- [What it is](#what-it-is)
- [Guarantees](#guarantees)
- [Subsystems](#subsystems)
- [Features](#features)
- [How a request runs](#how-a-request-runs)
- [Project structure](#project-structure)
- [Tech stack](#tech-stack)
- [Requirements](#requirements)
- [Getting started](#getting-started)
- [CLI usage](#cli-usage)
- [Commands](#commands)
- [Protocol support](#protocol-support)
- [Automation sandbox](#automation-sandbox)
- [Testing and coverage](#testing-and-coverage)
- [Roadmap](#roadmap)
- [Documentation and license](#documentation-and-license)

---

## What it is

Tyny Pulse is a lightweight alternative to cloud-heavy API clients: REST, GraphQL, WebSockets and gRPC mock testing in one local app, plus OpenAPI SpecHub, a `pm.*` JavaScript sandbox, Git-native workspaces and a CI-friendly CLI.

Typical memory footprint is about **50–80 MB**. There is no required account and no proprietary cloud.

> **Honesty note:** gRPC is a **mock hub**, not a full production gRPC client. Cloud sync via `tyny.ca` is **not shipped**.

---

## Guarantees

These are the contracts the project is built to keep. Each row names the mechanism, not the slogan.

| Guarantee | Mechanism |
| --- | --- |
| Secrets never leave the machine by default | Local vault, AES-256-GCM at rest, Argon2 key derivation. Cloud sync is opt-in and still on the roadmap. |
| A workspace is auditable without the app | Collections, environments and specs are versionable JSON on disk. A native file watcher reloads external edits. |
| UI and Rust cannot silently drift | Every Tauri command payload is derived from Rust domain models. TypeScript bindings are generated into `src/types/generated` by `cargo test`; CI fails on drift. |
| User scripts cannot reach the host | Isolated QuickJS sandbox. `require()` only sees bundled MIT libraries, toggleable per workspace. |
| A collection runs the same in the UI and in CI | One Rust runner. `tyny-cli` emits JSON/JUnit/HTML reports and pipeline exit codes. |
| Offline work still works | Viewing, editing and organizing collections does not require the network. Git push/pull waits until a remote is reachable. |

---

## Subsystems

| Area | Responsibility |
| --- | --- |
| Protocol hub | REST (HTTP/1–2), GraphQL queries/mutations, WebSockets, gRPC mock hub |
| SpecHub | Author, preview and lint OpenAPI 3.0 / 3.1 |
| Vault | Local secret storage, encryption at rest |
| Sandbox | Pre-request / post-response scripts with the `pm.*` API |
| Workspace | JSON on disk, file watcher, in-app Git (stage, commit, branch, push/pull, JSON diffs) |
| Runner / CLI | Collection runner shared by the GUI and `tyny-cli`; load testing on Tokio |

---

## Features

- **Local-first, Git-native workspaces** — readable JSON; collaborate with the repo you already own (GitHub, GitLab or self-hosted).
- **Live disk sync** — edits made outside the app show up without a manual reload.
- **SpecHub** — OpenAPI 3.0 / 3.1 with live governance linting.
- **JavaScript automation** — pre-request logic, assertions and environment writes via `pm.*`.
- **Bundled script libraries** — `lodash`, `dayjs`, `crypto-js`, `uuid`, enabled per workspace.
- **Load testing** — 1–500 VUs, ramp-up, live RPS/latency charts, JSON and Markdown export.
- **Collection reports** — HTML/Markdown from the same Rust renderers used by the CLI.
- **Command palette** — `Ctrl+P` / `Cmd+P`, dark/light themes.

---

## How a request runs

```
┌──────────────┐     typed IPC      ┌──────────────────────────────┐
│  React UI    │ ─────────────────▶ │  application (Rust use cases)│
│  Zustand     │ ◀───────────────── │                              │
└──────────────┘                    └──────────────┬───────────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    ▼                              ▼                              ▼
            ┌───────────────┐              ┌───────────────┐              ┌───────────────┐
            │ protocol hub  │              │ local vault   │              │ workspace JSON│
            │ REST / GQL    │              │ AES-256-GCM   │              │ + file watcher│
            │ WS / gRPC mock│              └───────────────┘              │ + git         │
            └───────────────┘                                             └───────────────┘
                    │
                    ▼
            ┌───────────────┐
            │ runner / CLI  │  same engine as `tyny-cli`
            │ + load test   │
            └───────────────┘
```

The topology is a desktop app with a strict inward dependency rule. Extracting a subsystem is an adapter change, not a rewrite of the domain.

---

## Project structure

```
tyny-pulse/
├── src/                          # FRONTEND (React 19 + TypeScript)
│   ├── components/
│   ├── store/                    # Zustand
│   ├── types/                    # includes generated bindings
│   ├── App.tsx
│   └── main.tsx
│
└── src-tauri/                    # BACKEND & DESKTOP (Rust + Tauri v2)
    ├── Cargo.toml
    ├── tauri.conf.json           # App ID: ca.tyny.pulse
    └── src/
        ├── domain/               # entities & outbound ports — no framework imports
        ├── application/          # use cases, depend only on ports
        ├── infrastructure/       # reqwest, QuickJS, vault, git, CLI adapters
        └── main.rs               # bootstrap & command bindings
```

**Boundary rules**

- `domain/` and `application/` in Rust never import infrastructure or Tauri UI bindings.
- Frontend state talks to Rust only through typed command adapters.
- Workspace state lives in user-controlled JSON files — no required cloud backend.

---

## Tech stack

| Concern | Choice |
| --- | --- |
| Desktop runtime | Tauri v2 — native webview, not a full Chromium shell |
| Engine / I/O | Rust 2021 + Tokio (`reqwest`, load test, file watcher) |
| UI | React 19 + TypeScript 5.8 + Vite 7 |
| State | Zustand, kept off the IPC boundary |
| Styling | CSS Modules, glassmorphic UI, Framer Motion, Lucide |
| Secrets | AES-256-GCM at rest, Argon2 key derivation |
| Scripts | QuickJS + `pm.*`; libs bundled under `src-tauri/assets/script-libs/` |
| Collaboration | Git on local JSON — no proprietary sync server |
| Targets | Windows, macOS, Linux |

---

## Requirements

- **Node.js** 18+
- **Rust** 1.75+ via `rustup`
- Coverage tools (only for `scripts/coverage-rust.sh`):
  - `rustup component add llvm-tools-preview`
  - `cargo install cargo-llvm-cov --locked`
- **Linux / WSL2**

```bash
sudo apt update
sudo apt install -y libwebkit2gtk-4.1-dev build-essential curl wget file libssl-dev libayatana-appindicator3-dev librsvg2-dev
```

---

## Getting started

### 1. Clone

```bash
git clone https://github.com/daniel-castilho/tyny-pulse.git
cd tyny-pulse
git config core.hooksPath .githooks   # pre-commit formatter
```

### 2. Install and run the desktop app

```bash
npm install
npm run tauri dev
```

Native installers (`.exe`, `.msi`, `.dmg`, `.AppImage`, `.deb`) come from:

```bash
npm run tauri build
```

Output: `src-tauri/target/release/bundle/`.

### 3. First success without the UI

```bash
cargo build --release --manifest-path src-tauri/Cargo.toml --bin tyny-cli

src-tauri/target/release/tyny-cli run src-tauri/tests/fixtures/sample_collection.json
```

Exit `0` means the collection passed. A ready-to-copy GitHub Actions workflow is in [`.github/workflows/tyny-cli-ci.yml`](.github/workflows/tyny-cli-ci.yml).

---

## CLI usage

`tyny-cli` is a dedicated binary: no window, no GUI subsystem, usable in CI.

```bash
BIN=src-tauri/target/release/tyny-cli

$BIN run collections/smoke.json \
    -e environments/staging.json \
    -v baseUrl=https://staging.api.example.com \
    -r report.junit
```

| Flag | Purpose |
| --- | --- |
| `-e`, `--env <path>` | Load an environment JSON file |
| `-g`, `--globals <path>` | Load global variables JSON |
| `-v`, `--var <key=value>` | Override an environment variable (repeatable) |
| `-r`, `--report <path>` | Write a report (`.json` or `.xml` / `.junit` selects the writer) |
| `-f`, `--format json\|junit` | Force the writer when the extension is unknown |

Exit codes: `0` all tests passed · `1` test failures · `2` usage/input error · `3` domain error.

---

## Commands

| Purpose | Command |
| --- | --- |
| Desktop dev (Vite + Tauri) | `npm run tauri dev` |
| Web preview only | `npm run dev` |
| Type-check and web build | `npm run build` |
| Native desktop binary | `npm run tauri build` |
| Rust check | `cd src-tauri && cargo check` |
| Rust tests | `cd src-tauri && cargo test` |
| Frontend tests | `npm test` |
| Frontend coverage (fail under gate) | `npm run test:coverage` |
| Rust coverage (fail under gate) | `bash scripts/coverage-rust.sh` |

---

## Protocol support

```
                  ┌─────────────────────────────────────────┐
                  │               TYNY PULSE                │
                  └────────────────────┬────────────────────┘
                                       │
         ┌─────────────────┬───────────┴─────────┬──────────────────┐
         │                 │                     │                  │
   ┌─────▼─────┐     ┌─────┴─────┐         ┌─────▼─────┐      ┌─────▼─────┐
   │   REST    │     │  GraphQL  │         │ WebSocket │      │   gRPC    │
   │ HTTP/1,2  │     │ Query/Mut │         │ Real-time │      │ Mock Hub  │
   └───────────┘     └───────────┘         └───────────┘      └───────────┘
```

- **REST** — GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD; full header and body control.
- **GraphQL** — queries and mutations, schema introspection, variable editor.
- **WebSocket** — bidirectional real-time messages.
- **gRPC** — mock hub for testing, not a complete client stack.

---

## Automation sandbox

Scripts run in isolated QuickJS. The surface is the familiar `pm.*` API.

### Pre-request

```javascript
const timestamp = new Date().toISOString();
pm.environment.set('request_time', timestamp);

const randomId = 'test_' + Math.floor(Math.random() * 10000);
pm.environment.set('test_correlation_id', randomId);
```

### Post-response

```javascript
pm.test('Status code is 200 OK', () => {
  expect(pm.response.status).to.equal(200);
});

pm.test('Response time is under 200ms', () => {
  expect(pm.response.responseTime).to.be.below(200);
});

pm.test('Validate User Payload', () => {
  const json = pm.response.json();
  expect(json.id).to.be.a('number');
  expect(json.email).to.include('@');
  pm.environment.set('auth_token', json.token);
});
```

### Bundled libraries

```javascript
const _ = require('lodash');
const dayjs = require('dayjs');
const CryptoJS = require('crypto-js');
const { v4: uuidv4 } = require('uuid');

pm.test('Signed payload is deterministic', () => {
  const signature = CryptoJS.HmacSHA256(
    JSON.stringify({ id: uuidv4() }),
    pm.environment.get('secret'),
  );
  expect(signature.toString()).to.be.a('string');
});
```

Libraries are bundled at build time (`src-tauri/assets/script-libs/`; see `THIRD-PARTY-NOTICES.md`)