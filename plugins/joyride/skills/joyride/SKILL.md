---
name: joyride
description: 'Write effective Joyride scripts and source files in ClojureScript. Covers VS Code API interop, async/promise handling, data-oriented functional design, REPL-driven development, disposable lifecycle, Flares (WebViews), and extension API access. Use when: writing or editing .cljs files for Joyride, automating VS Code, building interactive tools with the VS Code API, creating Flare WebViews, fixing or debugging Joyride scripts, troubleshooting async/promise issues, working with joyride.core or promesa.core, or doing REPL-driven ClojureScript development in VS Code. REPL: use joyride_evaluate_code, not the Backseat Driver clojure_evaluate_code — the Joyride REPL is a separate runtime.'
---

# Joyride — Effective Scripting

Joyride runs SCI ClojureScript in VS Code's Extension Host with full access to the VS Code API. Scripts and source files automate VS Code in user space — no extension authoring required.

## When to Use This Skill

- Writing or editing any `.cljs` file intended for Joyride
- Using the VS Code API from ClojureScript
- Building interactive tools, commands, or UI (quick picks, status bar items, WebViews)
- Working with async operations, promises, or the Joyride REPL
- Designing data-oriented, functional solutions for VS Code automation

## Related Skills (Cross-References)

| Skill | Scope |
|-------|-------|
| [joyride-user-scripts](../joyride-user-scripts/SKILL.md) | User-global scripts/source in the Joyride User directory — activation, classpath, keybindings |
| [joyride-workspace-automation](../joyride-workspace-automation/SKILL.md) | Workspace `.joyride/` directory — project-local scripts, activation, workspace-specific commands |

This skill covers **how to write effective Joyride code**. The related skills cover **where code lives and how it's organized**.

## Principles

### Data-Oriented, Functional Design

- **Functions take args, return results.** Side effects are a last resort serving the larger goal.
- **Prefer destructuring** and maps for function arguments.
- **Prefer namespaced keywords.** Use synthetic namespaces (`:my-tool/name`, `:picker/label`) to group related keys without requiring actual namespace definitions. Destructure with `:ns/keys`:
  ```clojure
  (let [{:picker/keys [label detail]} {:picker/label "Save" :picker/detail "Save all files"}]
    [label detail])
  ```
- **Prefer flatness over depth** when modeling data. A flat map with namespaced keys beats nested maps.
- **Use threading macros** (`->`, `->>`, `some->`) for readable data pipelines. Use `some->` when any step might return nil — common with VS Code API lookups where editors or extensions may not exist.
- **Use `defonce`** for atoms holding UI state — it prevents re-initialization when reloading code in the REPL.
- **Convert JS objects to Clojure data early.** Use `js->clj` for deep conversion, `(seq js-array)` for JS arrays, and property access for selective extraction. Work with Clojure data internally; touch JS objects only at the boundaries.
- **Build solutions step by small step.** Each step is an expression you evaluate to verify it does what you think. This is interactive programming — the REPL is your primary tool, not an afterthought.

### Look for Prior Art

Before building something new, check the user's existing Joyride scripts and source for patterns, utilities, and solutions. The Joyride User directory (`joyride/user-joyride-dir`) contains `scripts/` and `src/` with working code that may already solve part of the problem — or demonstrate project-specific conventions to follow.

```clojure
;; Find existing .cljs files in the User Joyride directory
(p/let [files (vscode/workspace.findFiles
               (str (joyride/user-joyride-dir) "/**/*.cljs"))]
  (mapv #(.-fsPath %) files))
```

Reuse existing namespace requires, utility functions, and data patterns rather than reinventing them.

### REPL-First Development

**Always use the Joyride REPL tool** (`joyride_evaluate_code`) for evaluating Joyride code — not a general-purpose Clojure REPL. The Joyride REPL runs inside VS Code's Extension Host where it has access to the VS Code API, handles promises correctly via `awaitResult`, and shares state with running scripts.

Before writing to files, develop in the REPL:

1. **Start small** — evaluate tiny sub-expressions to understand APIs
2. **Build up** — compose working pieces into larger expressions
3. **Verify each step** — every expression should return the expected result
4. **Only then write files** — once the solution is proven in the REPL

**Evaluating sub-expressions is preferred over `println` / `js/console.log`.** Directly inspect values instead of printing them.

### Namespace Targeting

When evaluating in the REPL, always target the correct namespace. Functions defined in the wrong namespace (e.g., `user` instead of your intended ns) become unavailable where expected.

### REPL State vs Script Execution

Critical distinction that agents frequently confuse:

- **REPL evaluation**: Defining functions interactively creates definitions that exist only in the REPL session memory
- **Script execution**: Loading/running a file restores all file-based definitions, overwriting any REPL changes
- **Function redefinition**: When you redefine a function in the REPL, that new definition completely replaces the old one. The system doesn't "check back" with the file — the REPL state IS the current truth until the next script reload.

If functions seem to "not work" after REPL definition:
1. Verify the function exists in the correct namespace: `(ns-publics 'my.namespace)`
2. Check if you're in the right namespace: `*ns*`
3. Confirm namespace targeting in evaluation tools
4. Remember that script re-execution will restore file definitions

### UI Testing Process

With UI testing the human developer is the source of truth together with the REPL:

1. Think about what should be tested
2. Summarize the test and expectations to the human
3. Evaluate the test using `awaitResult: true`
4. When the evaluation tool use returns:
   1. Examine any results from the REPL
   2. Ask the human for feedback (using the chat, tool use here risks messing with the UI)
   3. Stop and listen
5. Rinse and repeat

### Gather → Transform → Act

Structure code as a data pipeline: gather data from the VS Code world, transform it as pure Clojure data, then act with side effects.

```clojure
;; Gather: pull data from VS Code into Clojure
(p/let [editor vscode/window.activeTextEditor]
  (let [;; Transform: flat map with namespaced keys — pure data
        doc-data {:doc/uri     (-> editor .-document .-uri .-fsPath)
                  :doc/lang    (-> editor .-document .-languageId)
                  :cursor/line (-> editor .-selection .-active .-line)}]
    ;; Act: decision is pure, side effect is thin
    (when (= (:doc/lang doc-data) "clojure")
      (vscode/window.showInformationMessage
       (str "Clojure file at line " (:cursor/line doc-data))))))
```

The data map IS the testable unit. Print it, filter it, assert on it — all in the REPL.

## VS Code API Interop

Access VS Code APIs via JS interop. Use `vscode/api.method` for functions and members. Use `#js {}` literals for plain JS objects — **never `reify` or `deftype`** to implement JS interfaces.

```clojure
(require '["vscode" :as vscode])

;; Functions and methods
(vscode/window.showInformationMessage "Hello!")
(vscode/commands.executeCommand "workbench.action.files.save")

;; Quick pick (returns a promise)
(vscode/window.showQuickPick #js ["Option A" "Option B"])

;; Input box (returns a promise)
(vscode/window.showInputBox #js {:prompt "Enter a value:"})

;; Plain JS objects for options
#js {:placeHolder "Search..." :matchOnDescription true}
```

### Implementing JS Interfaces

VS Code APIs often expect objects implementing interfaces (e.g., `TreeDataProvider`, `TextDocumentContentProvider`). In SCI, `reify` with `Object` fails (`Unable to resolve symbol: Object`) and `deftype` only supports built-in methods like `toString` — not arbitrary interface methods.

**Always use `#js {}` with function values:**

```clojure
;; Implementing a VS Code interface
(def provider
  #js {:provideTextDocumentContent (fn [uri] "content here")
       :onDidChange                 js/undefined})

;; Registering with VS Code
(vscode/workspace.registerTextDocumentContentProvider "my-scheme" provider)
```

### Fluent JavaScript Object Configuration

Combine `doto` with the threading macro `->` to set properties fluently on JavaScript objects:

```clojure
(let [picker (vscode/window.createQuickPick)]
  (doto picker
    (-> .-items (set! items))
    (-> .-placeholder (set! "Select..."))
    (-> .-canSelectMany (set! false))
    (.onDidAccept handler)
    (.show)))
```

### VS Code Script Commands

```clojure
;; Execute user scripts from ~/.config/joyride/scripts/
(vscode/commands.executeCommand "joyride.runUserScript" "script_name.cljs")

;; For workspace scripts
(vscode/commands.executeCommand "joyride.runWorkspaceScript" "script_name.cljs")
```

These commands properly initialize the Joyride environment and handle script lifecycle correctly.

### Property Access

```clojure
;; Read properties with .-
(.-uri vscode/window.activeTextEditor.document)

;; Nested property access
(-> vscode/window.activeTextEditor .-document .-uri .-fsPath)
```

## Async / Promise Handling

Joyride uses `promesa.core` for async operations. Many VS Code APIs return promises.

```clojure
(require '[promesa.core :as p])

;; p/let chains async steps — each binding awaits the previous
(p/let [result (vscode/window.showInputBox #js {:prompt "Name?"})
        files  (vscode/workspace.findFiles "**/*.cljs")]
  (when result
    {:input result :file-count (count files)}))
```

### Capturing Async Results for REPL Exploration

```clojure
;; Bind async results to vars for later inspection
(p/let [files (vscode/workspace.findFiles "**/*.cljs")]
  (def found-files files))
;; Now evaluate `found-files` to explore the data
```

### When to Await

| Scenario | Await? |
|----------|--------|
| User input (showInputBox, showQuickPick) | Yes — need the response |
| File operations (findFiles, readFile) | Yes — need the data |
| Extension API calls returning promises | Yes — need the result |
| Fire-and-forget info messages | No |
| Side-effect-only async ops | No |

## Joyride Core API

```clojure
(require '[joyride.core :as joyride])

joyride/*file*                    ; Path of the current file
(joyride/invoked-script)          ; Script being run (nil in REPL)
(joyride/extension-context)       ; VS Code ExtensionContext
(joyride/output-channel)          ; Joyride's OutputChannel
joyride/user-joyride-dir          ; User joyride directory path

;; Async file operations
(joyride/slurp "path/to/file")    ; Returns promise — like clojure.core/slurp but async
(joyride/load-file "path/to.cljs"); Returns promise — loads and evaluates a file
```

**Note:** `load-file` is the Joyride async version. Clojure's `load-file` is not implemented in SCI.

## Script Execution Guard

Scripts that should only execute when run directly (not when loaded via `require` in other code or the REPL):

```clojure
(when (= (joyride/invoked-script) joyride/*file*)
  (main))
```

This is Joyride's equivalent of Python's `if __name__ == "__main__"`.

## Disposable Lifecycle

VS Code uses disposables for event subscriptions, status bar items, commands, and other UI elements. Always manage their lifecycle.

### Registering with Extension Context

For items that should live as long as Joyride is active:

```clojure
(let [disposable (vscode/workspace.onDidOpenTextDocument handler)]
  (.push (.-subscriptions (joyride/extension-context)) disposable))
```

### Holding References for Dynamic UI

For items you want to modify or dispose at will (status bar buttons, decorations):

```clojure
;; Hold a reference so you can update/dispose later
(defonce !status-item (atom nil))

(defn create-status-item! []
  (when-let [old @!status-item] (.dispose old))
  (let [item (vscode/window.createStatusBarItem
              vscode/StatusBarAlignment.Left 0)]
    (set! (.-text item) "$(rocket) My Tool")
    (set! (.-command item) "my.command")
    (.show item)
    (reset! !status-item item)
    item))
```

## Extension API Access

Safely access other extensions:

```clojure
(defn get-extension-api [extension-id]
  (when-let [ext (vscode/extensions.getExtension extension-id)]
    (when (.-isActive ext)
      (.-exports ext))))

;; Example: Git extension
(when-let [git-api (some-> (vscode/extensions.getExtension "vscode.git")
                           .-exports
                           (.getAPI 1))]
  (.-repositories git-api))
```

Always check availability and activation state before use.

## Flares — WebView Creation

Flares provide a convenient way to create WebView panels and sidebar views using Hiccup syntax.

```clojure
(require '[joyride.flare :as flare])

;; Panel with Hiccup
(flare/flare!+ {:html [:div [:h1 "Hello"] [:p "World"]]
                :title "My Panel"
                :key "my-panel"})

;; Sidebar slot (1-5 available)
(flare/flare!+ {:html [:div [:h2 "Sidebar"]]
                :key :sidebar-1})

;; From file
(flare/flare!+ {:file "assets/view.html" :key "file-view"})

;; External URL
(flare/flare!+ {:url "https://example.com" :title "Docs"})

;; Management
(flare/ls)             ; List open flares
(flare/close! "key")   ; Close by key
(flare/close-all!)     ; Close all
```

`flare!+` returns a promise. Hiccup `:style` attributes use maps: `{:color :red :margin "10px"}`.

For bidirectional communication, use `:message-handler` and `post-message!+`.

## Fetching Web Resources

Joyride can fetch web content directly — use this to pull documentation, API specs, raw files from GitHub, or any text resource:

```clojure
(p/let [response (js/fetch "https://raw.githubusercontent.com/user/repo/main/README.md")
        text     (.text response)]
  (def readme-content text))
```

This is especially useful for:
- Pulling up-to-date documentation or examples before writing code
- Fetching configuration or data files from remote sources
- Accessing raw content from GitHub repos (use `raw.githubusercontent.com` URLs)

## Keybinding Pattern

Document keybindings in source files for discoverability:

```clojure
;; "Install" by placing this file in the Joyride User src/ directory
;; and adding this keybinding to keybindings.json:
;; {
;;   "key": "ctrl+alt+j ctrl+alt+x",
;;   "command": "joyride.runCode",
;;   "args": "(require '[my-ns :as m] :reload) (m/my-command!+)"
;; }
```

The `:reload` ensures the latest code is executed.

## Anti-Patterns and Corrections

| Anti-pattern | Correction |
|---|---|
| Using `println` / `js/console.log` to inspect | Evaluate sub-expressions directly in the REPL |
| Top-level side effects outside `defn` | Wrap in functions; use script execution guard |
| `load-file` (Clojure built-in) | Use `joyride/load-file` (async, returns promise) |
| Using `reify` or `deftype` for JS interfaces | Use `#js {}` with function values — `reify Object` fails in SCI |
| Forgetting to dispose UI elements | Hold references; register with extension context |
| Reading `@atom` in pure functions | Pass data as function arguments |
| Deep nested maps for domain data | Flat maps with namespaced keywords |
| Forward declaring functions | Define before use — rearrange file order |
| Hardcoded fallback configs hiding errors | Fail fast with clear error messages |
| Mixing business logic with side effects | Pure functions for decisions; thin side-effect layer |
| Starting from scratch without checking existing code | Look for prior art in the User Joyride directory first |

## SCI / Scittle Async Gotchas

- Use `^:async` + `await` for async functions in SCI
- `js-await` is Squint-specific — fails in SCI with "Unable to resolve symbol"
- Top-level `await` is unsupported — `await` must be inside an `^:async` function
- Use `promesa.core/let` (`p/let`) as the primary async coordination tool

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Function not found after REPL eval | Check namespace targeting — may have ended up in `user` |
| Promise result is `#object[Promise]` | Use `p/let` to unwrap, or `awaitResult: true` in eval tool |
| Extension API returns nil | Check `isActive` — extension may not be activated yet |
| Status bar item not showing | Call `.show` and verify it's not disposed |
| Script runs on `require` when it shouldn't | Add script execution guard |
| `load-file` not working | Use `joyride.core/load-file` (async version) |

## Testing

Pure-function-first design enables testing in the REPL with `cljs.test`. Tests reside in `src/test/` with namespaces starting with `test.`. Run all tests:

```clojure
(do (require 'run-all-tests :reload) (run-all-tests/run!+))
```

Develop tests interactively in the REPL before committing them to files.

## References

- Joyride repo: <https://github.com/BetterThanTomorrow/joyride>
- Flare API docs: <https://github.com/BetterThanTomorrow/joyride/blob/master/doc/api.md#joyrideflare>
- VS Code API: <https://code.visualstudio.com/api/references/vscode-api>
- Promesa docs: <https://funcool.github.io/promesa/latest/>
