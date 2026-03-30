---
name: joyride-workspace-automation
description: 'Set up and manage Joyride workspace automation in the .joyride/ directory of a project. Use when working with workspace-level Joyride scripts and source files, configuring workspace activation, managing dependencies, and deciding between User vs Workspace scope for Joyride code. Use when using Backseat Driver tools with Joyride.'
---

# Joyride Workspace Automation

The `<workspace>/.joyride/` directory holds scripts and source files scoped to a **specific project**. Use this for project-local automation, workspace-specific commands, and team-shareable tooling.

**Requires: [joyride](../joyride/SKILL.md)** — read the foundational skill first for coding principles, API patterns, and async handling.

## When to Use This Skill

- Creating or editing files in a workspace's `.joyride/` directory
- Setting up `workspace_activate.cljs` for project-specific startup
- Adding project-local scripts or utilities
- Configuring workspace-level dependencies
- Deciding whether something belongs in Workspace vs User scope

## Directory Structure

```
<workspace>/
├── .joyride/
│   ├── deps.edn                       # Workspace Clojure dependencies
│   ├── package.json                    # Workspace npm dependencies
│   ├── scripts/
│   │   ├── workspace_activate.cljs     # Auto-runs when workspace opens
│   │   └── project_tool.cljs           # Runnable from "Run Workspace Script"
│   └── src/
│       └── project_utils.cljs          # Project utility functions
└── ... (project files)
```

## Scripts vs Source — Same Rules, Workspace Scope

The scripts/source distinction is identical to user-level (see [joyride-user-scripts](../joyride-user-scripts/SKILL.md)):

- **`scripts/`** — appear in *Joyride: Run Workspace Script* menu, use script execution guard
- **`src/`** — library code, called via keybindings or `require`

## Workspace Activation Script

`scripts/workspace_activate.cljs` runs automatically when Joyride activates in a workspace that contains a `.joyride/` directory. Use it for project-specific setup.

### Reloadable Activation Pattern

```clojure
(ns workspace-activate
  (:require [joyride.core :as joyride]
            [promesa.core :as p]
            ["vscode" :as vscode]))

(defonce !db (atom {:disposables []}))

(defn- clear-disposables! []
  (run! #(.dispose %) (:disposables @!db))
  (swap! !db assoc :disposables []))

(defn- push-disposable! [disposable]
  (swap! !db update :disposables conj disposable)
  (.push (.-subscriptions (joyride/extension-context)) disposable))

(defn- main []
  (clear-disposables!)
  ;; Project-specific setup here:
  ;; - Register commands for this project
  ;; - Set up file watchers for project file types
  ;; - Connect to project-specific services
  )

(when (= (joyride/invoked-script) joyride/*file*)
  (main))
```

### Typical Workspace Activation Tasks

- Opening project documentation or dashboards
- Connecting to project-specific REPLs or services
- Registering project-specific commands
- Setting up file watchers for build outputs or config changes

## Classpath Precedence

Workspace files take precedence over user files:

1. **`<workspace>/.joyride/src`** ← checked first
2. **`<workspace>/.joyride/scripts`**
3. `<user-joyride-dir>/src`
4. `<user-joyride-dir>/scripts`

This means a workspace can **override** user-level namespaces by providing a file with the same name. Use this intentionally — for example, to provide project-specific implementations of shared utilities.

## Dependencies

### Clojure Dependencies — `.joyride/deps.edn`

```clojure
{:deps {org.clojure/clojurescript {:mvn/version "1.11.54"}
        funcool/promesa {:mvn/version "9.0.471"}}
 :paths ["src" "scripts"]}
```

### npm Dependencies — `.joyride/package.json`

```bash
cd <workspace>/.joyride
npm install some-package
```

npm packages installed here are available to workspace Joyride code. Packages in the project root (`<workspace>/package.json`) are also on the resolution path.

## clojure-lsp Configuration

To get clojure-lsp to analyze workspace Joyride code:

1. Add a `:source-alias` to `.joyride/.lsp/config.edn`:
   ```clojure
   {:source-aliases #{:joyride}}
   ```

2. Add a `:joyride` alias to the project root `deps.edn`:
   ```clojure
   {:aliases {:joyride {:extra-deps {joyride/workspace {:local/root ".joyride"}}}}}
   ```

To also include user-level Joyride code in analysis, see the [joyride-user-scripts](../joyride-user-scripts/SKILL.md) skill for the `:joyride-user` alias setup.

## VS Code Commands

| Command | Purpose |
|---------|---------|
| `Joyride: Run Workspace Script` | Pick and run a script from `.joyride/scripts/` |
| `Joyride: Create Workspace Activate Script` | Scaffold `workspace_activate.cljs` |
| `Joyride: Create Workspace Script...` | Create a file in `.joyride/scripts/` |
| `Joyride: Create Workspace Source File...` | Create a file in `.joyride/src/` |

## Workspace vs User — Decision Guide

| Criterion | Workspace (`.joyride/`) | User (`<user-joyride-dir>/`) |
|-----------|------------------------|------------------------------|
| Applies to | This project only | All workspaces |
| Shareable with team | Yes — commit to repo | No — personal setup |
| Overrides | Wins over user-level code | Provides defaults |
| Typical use | Project tooling, build helpers | Personal editor customizations |
| Activation | `workspace_activate.cljs` | `user_activate.cljs` |

**Rule of thumb:** If it's useful in every workspace, put it in user. If it's project-specific or team-shareable, put it in workspace.

## Team Sharing

The `.joyride/` directory can be committed to the project repo. Consider adding a README in `.joyride/` explaining:
- What the scripts do
- Any required npm dependencies (`npm install` in `.joyride/`)
- Keybindings team members should add

Team members need the Joyride extension installed. Scripts will auto-run via `workspace_activate.cljs` and appear in the workspace script menu.

## Related Skills

| Skill | Scope |
|-------|-------|
| [joyride](../joyride/SKILL.md) | How to write effective Joyride code — principles, APIs, patterns |
| [joyride-user-scripts](../joyride-user-scripts/SKILL.md) | User-global scripts in the Joyride User directory |
