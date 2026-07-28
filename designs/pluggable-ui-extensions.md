# Pluggable UI Extensions

Status: draft · Related: [#729](https://github.com/omnigent-ai/omnigent/issues/729) · Sibling: [harness-plugin-interface.md](./harness-plugin-interface.md)

## Problem

Community UI ideas (Kanban boards, split panes, agent config managers, themes,
white-label skins) currently require forking `web/` or landing opinionated  
chrome in core. We already have a clean plugin path for harnesses /  
sandboxes (`omnigent.community.*` + entry points). The web SPA has none — routes,
settings sections, command palette, and block renderers are closed enums /
switches.

## Goals

- Install an extension from a **separate repo** without touching Omnigent source.
- Extension manageable via CLI: `omni extension [list|install|enable|disable|status|uninstall|config]`.
- Stable, versioned host contract so core can evolve without breaking every plugin.
- One bad extension must not blank the SPA.

## Non-goals (v1)

- Marketplace / signing / remote untrusted installs.
- Process/iframe sandbox (defer until marketplace trust is needed).
- Replacing core chat transcript / auth chrome (extensions **sit beside** or
  **act on** the conversation via rail tabs and actions — see Core seams).
- Micro-frontends, Module Federation, or letting plugins import `web/src/**` internals.
- Making native TUI harnesses pluggable (see harness-modular-registry-proposal.md).
- IDE plugins that only **embed** the SPA (#1891) — different product surface.

## Design in one sentence

**Obsidian-inspired package shape** (manifest + bundled ESM + `activate`/`deactivate`)
with **VS Code-inspired contribution point shape**, loaded by a thin SPA Extension Host
from a catalog the **Omnigent instance** serves — optionally paired with a Python
companion under the existing community namespace when server hooks are needed.

Install scope follows an **A → C roadmap** (instance store first, then per-user on
that instance). Other topologies are documented below but deferred.

## Where extensions live

The runnable UI is always the SPA from an Omnigent origin. Electron, mobile, and
the VS Code iframe are thin shells around that SPA — there is no separate client
UI bundle. So “server vs client” is the wrong frame. The real questions are:
**who may install**, and **whose catalog does that SPA load?**

### Topologies


| Topology                                   | What users need                                                                                |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Local self-host / desktop → localhost      | Personal plugins; instance store ≈ “my machine”                                                |
| Browser or desktop → shared central server | Org-wide chrome *and* personal UI without admin for every experiment                           |
| Desktop → remote URL                       | Temptation to keep plugins only on the device — but then browser users of the same URL diverge |


### Options

**A — Instance store (v1)**  
Packages in `$OMNIGENT_DATA_DIR/extensions/`, served same-origin
(`GET /v1/extensions`, `/extensions/<id>/*`).  

- One path for every client; org white-label; `omni extension` on the host;
Python companions co-locate.  
− On a locked-down central deploy, personal Kanban needs operator rights.  
Fits self-host / single-user and org-wide defaults.

**B — Device / client store (deferred)**  
Electron `userData` (or similar) injects into the webview.  

- Personal UI against a remote you don’t admin.  
− Browser parity breaks; remote-origin inject is CSP/CORS/trust-heavy;
Omnigent is not a desktop-only app like Obsidian.  
Escape hatch only if someone must customize a remote they cannot install on.

**C — User-scoped packages on the instance (next)**  
User installs into their account; bytes still served by that origin; SPA loads
instance defaults ∪ that user’s enabled set.  

- Browser and desktop stay equal; personal UI on central servers; white-label
remains instance defaults.  
− Needs accounts + authenticated install/storage API.  
Natural evolution of A when multi-user hurts — not a second product.

**D — Hybrid A+B (deferred)**  
Merge instance catalog with an Electron overlay. Max coverage, max complexity.

### Decision: A → C

1. **Host contract (stable for A and C):** SPA Extension Host loads a catalog
 from the instance it is talking to (same-origin modules + contribution merge).
2. **v1 = A.** Prove install/load with instance data dir. Honest limitation: shared
 servers need operator install until C.
3. **Next = C.** Per-user install and/or enable on the instance when accounts +
 central deploys demand personal UI. Prefer C over B so browser users are not
 second-class.
4. **Defer B/D** unless a concrete “personal UI on an unadminable remote” CUJ
 appears.

Storage location does not make extensions safer — once loaded they share the
SPA session. Isolation/sandbox is a separate phase.

```text
omni extension install <path|git-url>
        │
        ▼
 v1:  $OMNIGENT_DATA_DIR/extensions/<id>/     (A: instance)
 later: …/extensions/users/<user>/<id>/     (C: still on instance)
   manifest.json + dist/main.js (+ assets)
        │
        ▼
 Instance: GET /v1/extensions  +  /extensions/<id>/*
        │
        ▼
 Any client of that URL (browser / Electron / …)
   SPA ExtensionHost → dynamic import → activate(api)
        │
        ▼
 Contribution registries (routes, nav, commands, settings, themes)
```

## Why this shape (and not others)


| Pattern                   | Takeaway for Omnigent                                                                                                                                                           |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **VS Code**               | Contribution points + activation + versioned API are the durable core. Marketplace and extension-host isolation are distribution/trust layers — add later.                      |
| **Obsidian**              | Manifest + `main.js` + typed App API is the fastest path for trusted community plugins. Matches “build in another repo, install locally.”                                       |
| **Harness plugins today** | Entry points + `omnigent.community.*` namespace is the right model for **Python** companions, not for React UI bytes. Reuse the mental model; don’t force UI through pip alone. |


## Example Package format

```
omnigent-kanban/
  manifest.json
  dist/main.js          # ESM, self-contained bundle
  dist/main.css         # optional
```

`manifest.json` (illustrative):

```json
{
  "id": "kanban",
  "name": "Kanban",
  "version": "0.1.0",
  "engines": { "omnigent": ">=0.5.0", "extensionApi": "1" },
  "main": "dist/main.js",
  "activationEvents": ["onStartup"],
  "contributes": {
    "routes": [{ "path": "kanban", "title": "Kanban" }],
    "sidebar": [{ "id": "kanban", "label": "Kanban", "route": "kanban" }],
    "commands": [{ "id": "kanban.open", "title": "Open Kanban" }],
    "workspacePanels": [{ "id": "kanban-session", "title": "Board" }],
    "themes": []
  }
}
```

`main` exports:

```ts
export async function activate(api: ExtensionAPI): Promise<void> { /* … */ }
export async function deactivate(): Promise<void> { /* … */ }
```

Optional Python companion (only if needed): ship under
`omnigent.community.ui.<id>.*` with an entry point group analogous to
`omnigent.community.harness` — same namespace discipline, no overriding builtins.

## Host API (keep tiny)

Versioned facade (`extensionApi: "1"`), not a dump of React internals:

- **UI:** register route views, workspace-rail panels, composer/header actions,
  command handlers, theme tokens.
- **Data:** call existing HTTP/SSE surfaces the SPA already uses (`/v1/sessions`, …) — no direct Zustand mutation.
- **Config:** get/set extension-scoped settings (`omni extension config`).
- **Lifecycle:** disposables cleaned on `deactivate` / disable.

Core owns the shell; extensions **fill slots**.

## Core seams to open (minimal)

Replace closed lists with mergeable registries. **Fill slots; don’t replace the
chat transcript owner** (see Non-goals).

**App chrome**

1. Routes — [`web/src/App.tsx`](../web/src/App.tsx)
2. Sidebar / activity entries
3. Command palette — [`web/src/shell/CommandPalette.tsx`](../web/src/shell/CommandPalette.tsx)
4. Settings sections — [`web/src/shell/settingsNav.tsx`](../web/src/shell/settingsNav.tsx)
5. Theme tokens (CSS variables / appearance)

**Conversation surface** (needed for #729 split pane; still not “replace chat”)

[`AppShell`](../web/src/shell/AppShell.tsx) already owns a right **workspace rail**
(Files / viewer / terminals) beside the main [`ChatPage`](../web/src/pages/ChatPage.tsx)
outlet. That rail is the natural split-pane seam:

6. **Workspace rail tabs / panels** — contribute an extra tab or pane beside the
   conversation (split pane, agent config inspector, custom tool UI) without
   forking the transcript.
7. **Composer / conversation header actions** — small actions next to send or in
   the chat header (e.g. “Open board for this session”).

**Explicitly later (higher coupling)**

- **Block / transcript item renderers** — custom `item.kind` UI; touches reducer
  parity with `sdks/python-client`.
- **Center-pane view modes** — swap the linear chat body for an alternate view
  of the *same* session while keeping the composer. Powerful, easy to break
  streaming/elicitation; only after rail + action slots prove out.

Core keeps ownership of message streaming, elicitations, and send/interrupt.
Extensions decorate or sit beside that surface.

## Install & trust (v1 = A)

- Store: `$OMNIGENT_DATA_DIR/extensions/` (default `~/.omnigent/extensions/`).
- Sources: local path, git URL. Enable/disable is instance metadata next to the package.
- **Trust:** installed code is served same-origin with the SPA. On a shared  
instance, install is an operator decision for everyone until Option C adds per-user  
scope. Sandbox + marketplace only if/when we accept untrusted remotes.

## Phased rollout

1. **In-tree registries + empty ExtensionHost** — prove contribution merge.
2. **A: local path install + dynamic** `import()` — unblocks ideas like #729 (Kanban, themes).
3. **A: git URL install + Settings “Extensions” UI** (instance enable/disable).
4. **C: per-user install and/or enable** on the instance (accounts-aware catalog).
5. **Sandbox + curated index** if/when we want a marketplace; reconsider B/D only
 with a concrete remote-overlay CUJ.

## Open questions for discussion

1. Should theme-only packs be a separate lighter format (CSS + manifest, no JS)?
2. Should Python companions be required to declare a UI manifest id, or stay fully independent plugins?
3. For Option C: is per-user **enable** of instance-installed packages enough at first, or
 do we need per-user **install** (private packages other accounts never see)?
4. Embed hosts (Databricks island): inherit the instance catalog, or disable
 third-party UI extensions in embed mode initially?

## Example CUJ

`omni extension install https://github.com/zcking/omnigent-kanban` → files land under
`~/.omnigent/extensions/kanban/` → server lists it enabled → SPA imports
`/extensions/kanban/dist/main.js` → contributes a sidebar item + `/kanban` route
that renders a board over session/task APIs — **zero Omnigent source changes**.