# awesome-backseat-driver

Plugin marketplace for Clojure AI context in GitHub Copilot — agents, skills, and workflows for REPL-first interactive programming with [Calva Backseat Driver](https://github.com/BetterThanTomorrow/calva-backseat-driver).

## Install

In VS Code, run **Chat: Install Plugin From Source** and enter:

```
https://github.com/BetterThanTomorrow/awesome-backseat-driver
```

Or add to your settings:

```json
"chat.plugins.marketplaces": ["BetterThanTomorrow/awesome-backseat-driver"]
```

## Plugins

| Plugin | Description |
|---|---|
| `clojure` | REPL-first Clojure development — agent and skill for any dialect and runtime |
| `clojure-editor` | Subagent for editing Clojure files using Backseat Driver structural editing tools |
| `babashka` | Babashka scripting and bb.edn task skills for idiomatic Babashka development |
| `joyride` | Joyride skills for VS Code automation with ClojureScript — scripting, user scripts, and workspace automation |
| `epupp` | Browser tampering and userscript development with Epupp (ClojureScript/Scittle in the browser) |

## Relationship to Calva Backseat Driver

This marketplace provides the **optional philosophy layer** — agents with Clojure methodology, community-contributed skills for patterns and idioms.

The **Calva Backseat Driver extension** provides the foundational tooling layer — REPL evaluation, structural editing, symbol lookup, and other tools. Install it from the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=betterthantomorrow.calva-backseat-driver).

## License

MIT
