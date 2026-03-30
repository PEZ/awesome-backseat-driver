# Global Babashka Tasks (bbg)

Complete recipe for setting up a personal global Babashka task system — utilities available from any directory via a `bbg` command. Based on https://github.com/PEZ/my-bbg

Reference: https://clojureverse.org/t/help-utilizing-babashka-tasks-globally/8026/2

## Architecture

```
~/.config/bbg/
├── bbg                # Wrapper script (symlinked from ~/bin/bbg)
├── bb.edn             # Thin task wrappers → scripts/
├── scripts/*.clj      # Task implementation modules
└── completions.zsh    # Zsh tab completions
```

Same pattern as project-level tasks: thin `bb.edn` wrappers delegating to `scripts/*.clj` modules. The only difference is the wrapper script that makes it globally accessible.

## Setup Steps

### 1. Create the directory

```sh
mkdir -p ~/.config/bbg/scripts
```

### 2. Create the wrapper script

```bash
# ~/.config/bbg/bbg
#!/usr/bin/env bash

bb --config ~/.config/bbg/bb.edn "$@"
```

```sh
chmod +x ~/.config/bbg/bbg
```

### 3. Symlink into PATH

```sh
ln -sf ~/.config/bbg/bbg ~/bin/bbg
```

Requires `~/bin` on `$PATH`. Add to `.zshrc` if needed: `export PATH="$HOME/bin:$PATH"`

### 4. Create bb.edn

```clojure
;; ~/.config/bbg/bb.edn
{:paths ["scripts"]
 :deps {org.babashka/cli {:mvn/version "0.2.23"}}
 :tasks
 {:requires ([babashka.cli :as cli]
             [loc])

  loc {:doc "Count lines of code (wraps cloc)"
       :task (loc/count!)}}}
```

### 5. Create a starter task module

```clojure
;; ~/.config/bbg/scripts/loc.clj
(ns loc
  (:require [babashka.process :as p]))

(defn count! []
  (p/shell
   "cloc"
   "--vcs=git"
   "--exclude-lang=Markdown"
   "."))
```

### 6. Add zsh completions (optional)

Use the zsh completion pattern from the main babashka-tasks skill, replacing `bb` with `bbg` and `_bb_complete` with `_bbg`. Source in `.zshrc`:

```sh
source ~/.config/bbg/completions.zsh
```

## Useful Starter Tasks

### bb — Manage Babashka binaries

Download, switch between, and manage `bb` versions. Useful when you need a pre-release build.

```clojure
;; bb.edn addition
bb {:doc "Manage bb binary [--download <ref> | --use <ref> | --unuse]"
    :task (bb-man/exec! (cli/parse-opts *command-line-args*
                                        bb-man/cli-spec))}
```

The module uses the GitHub API (via `gh auth token`) to download artifacts from CI workflows by PR number, SHA, branch, or tag.

### bb-nrepl — Start a Babashka nREPL

```clojure
;; scripts/bb_nrepl.clj
(ns bb-nrepl
  (:require [babashka.nrepl.server :as nrepl]))

(def cli-spec {:coerce {:port :int}
               :alias {:p :port}})

(defn start! [{:keys [port]}]
  (let [port (or port (+ 1024 (rand-int 64000)))
        server (nrepl/start-server! {:host "localhost" :port port})]
    (spit ".bb/.nrepl-port" (str port))
    (println (str "nREPL server started on port " port))
    @(promise)))
```

## Design Patterns

The CLI spec per module pattern and private task conventions are covered in the main babashka-tasks skill.

### Internal/private tasks

Prefix with `-` to hide from `bb tasks` listing:

```clojure
-my:test:unit {:doc "Run tests"
               :task (do (require 'my-test :reload)
                         (let [{:keys [fail error]} (clojure.test/run-tests 'my-test)]
                           (System/exit (if (zero? (+ fail error)) 0 1))))}
```
