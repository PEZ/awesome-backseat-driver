---
name: joyride-user-scripts
description: 'Organize and manage Joyride User scripts and source files — the global Joyride directory that applies across all VS Code workspaces. Covers directory structure (scripts/ vs src/), user_activate.cljs, deps.edn and package.json for dependencies, keybinding setup, clojure-lsp configuration, and classpath resolution. Use when: creating or editing files in the Joyride User directory, setting up user activation, adding global keybindings for Joyride functions, configuring user-level dependencies, or organizing user-scoped scripts and source.'
---

# Joyride User Scripts

The Joyride User directory holds scripts and source files that apply **globally** across all VS Code workspaces. This is the place for personal tools, editor customizations, and reusable utilities.

**Requires: [joyride](../joyride/SKILL.md)** — read the foundational skill first for coding principles, API patterns, and async handling.

## When to Use This Skill

- Creating or editing files in the Joyride User directory
- Setting up `user_activate.cljs` for VS Code startup automation
- Adding global keybindings that invoke Joyride functions
- Configuring user-level dependencies (`deps.edn`, `package.json`)
- Deciding whether something belongs in User vs Workspace scope

## Directory Structure

The User Joyride directory path is available at runtime via `joyride/user-joyride-dir`. Typical structure:

```
<user-joyride-dir>/
├── deps.edn                   # Clojure dependencies + classpath
├── package.json               # npm dependencies
├── scripts/
│   ├── user_activate.cljs     # Auto-runs when Joyride activates
│   └── my_tool.cljs           # Runnable from "Joyride: Run User Script"
└── src/
    └── my_lib.cljs            # Library code called by keybindings or scripts
```

## Scripts vs Source Files

| Aspect | `scripts/` | `src/` |
|--------|-----------|--------|
| **Purpose** | Runnable files — appear in *Joyride: Run User Script* menu | Library functions called by shortcuts, other scripts, or activation |
| **When to use** | Direct execution from menus or activation | Reusable functionality triggered by keybindings or `require` |
| **Namespace** | Flat — file name becomes namespace | Can use nested directories for namespace hierarchy |
| **Typical pattern** | Script execution guard at bottom | Functions only — no guard needed |

### Script Example

```clojure
;; scripts/toggle_font.cljs
(ns toggle-font
  (:require ["vscode" :as vscode]
            [joyride.core :as joyride]))

(defn toggle-large-font []
  (let [config  (vscode/workspace.getConfiguration "editor")
        current (.get config "fontSize")]
    (if (> current 16)
      (.update config "fontSize" 14 vscode/ConfigurationTarget.Global)
      (.update config "fontSize" 20 vscode/ConfigurationTarget.Global))))

(when (= (joyride/invoked-script) joyride/*file*)
  (toggle-large-font))
```

### Source Example

```clojure
;; src/my_utils.cljs
(ns my-utils
  (:require ["vscode" :as vscode]))

(defn toggle-large-font []
  (let [config  (vscode/workspace.getConfiguration "editor")
        current (.get config "fontSize")]
    (if (> current 16)
      (.update config "fontSize" 14 vscode/ConfigurationTarget.Global)
      (.update config "fontSize" 20 vscode/ConfigurationTarget.Global))))

;; Called from keyboard shortcut:
;; {
;;   "key": "ctrl+alt+f",
;;   "command": "joyride.runCode",
;;   "args": "(require 'my-utils :reload) (my-utils/toggle-large-font)"
;; }
```

The `:reload` in the keybinding ensures the latest saved code runs.

## User Activation Script

`scripts/user_activate.cljs` runs automatically when Joyride activates (VS Code startup). Use it for global setup: registering commands, creating status bar items, setting up event listeners.

### Reloadable Activation Pattern

```clojure
(ns user-activate
  (:require ["vscode" :as vscode]
            [joyride.core :as joyride]
            [promesa.core :as p]))

(defonce !db (atom {:disposables []}))

(defn- clear-disposables! []
  (run! #(.dispose %) (:disposables @!db))
  (swap! !db assoc :disposables []))

(defn- push-disposable! [disposable]
  (swap! !db update :disposables conj disposable)
  (.push (.-subscriptions (joyride/extension-context)) disposable))

(defn- my-main []
  (clear-disposables!)
  ;; Register commands, status bar items, event listeners here
  ;; Use push-disposable! for each one
  )

(when (= (joyride/invoked-script) joyride/*file*)
  (my-main))
```

Key points:
- `defonce !db` survives script re-runs — holds disposable references
- `clear-disposables!` cleans up previous registrations, making the script idempotent
- `push-disposable!` both tracks locally and registers with VS Code for cleanup on extension deactivation

## Keybinding Setup

Source functions in `src/` are invoked via VS Code keybindings:

```json
{
  "key": "ctrl+alt+j ctrl+alt+x",
  "command": "joyride.runCode",
  "args": "(require '[my-ns :as m] :reload) (m/my-command!+)"
}
```

Multi-key chords (e.g., `ctrl+alt+j` followed by another key) create a personal keybinding namespace that avoids conflicts. Document the keybinding in the source file for discoverability.

## Dependencies

### Clojure Dependencies — `deps.edn`

```clojure
{:deps {org.clojure/clojurescript {:mvn/version "1.11.54"}
        funcool/promesa {:mvn/version "9.0.471"}}
 :paths ["src" "scripts"]}
```

This file also configures the classpath for clojure-lsp analysis (see below).

### npm Dependencies — `package.json`

Install npm packages in the User Joyride directory:

```bash
cd "$(joyride/user-joyride-dir)"  # or the actual path
npm install lodash
```

Then require in ClojureScript:

```clojure
(require '["lodash" :as _])
```

## Classpath Resolution

Joyride resolves `require` in this order:

1. `<workspace>/.joyride/src`
2. `<workspace>/.joyride/scripts`
3. `<user-joyride-dir>/src`
4. `<user-joyride-dir>/scripts`

Workspace files take precedence. This means a workspace can override user-level code by providing a file with the same namespace.

## clojure-lsp Configuration

To get clojure-lsp analysis for the User Joyride directory:

1. Add a `:source-alias` to the User Joyride directory's `.lsp/config.edn`:
   ```clojure
   {:source-aliases #{:joyride-user}}
   ```

2. Add `:joyride-user` to your global/user `deps.edn` aliases:
   ```clojure
   {:aliases
    {:joyride-user {:extra-deps {joyride/user {:local/root "<user-joyride-dir>"}}}}}
   ```
   Replace `<user-joyride-dir>` with your actual Joyride User directory path.

## VS Code Commands

| Command | Purpose |
|---------|---------|
| `Joyride: Run User Script` | Pick and run a script from `scripts/` |
| `Joyride: Create User Activate Script` | Scaffold `user_activate.cljs` |
| `Joyride: Create Hello Joyride User Script` | Create example script |
| `Joyride: Create User Source File...` | Create a file in `src/` |

## Version Control

Put the User Joyride directory under version control. It contains your personal VS Code automation — treat it like dotfiles.

## Related Skills

| Skill | Scope |
|-------|-------|
| [joyride](../joyride/SKILL.md) | How to write effective Joyride code — principles, APIs, patterns |
| [joyride-workspace-automation](../joyride-workspace-automation/SKILL.md) | Project-local scripts in `<workspace>/.joyride/` |
