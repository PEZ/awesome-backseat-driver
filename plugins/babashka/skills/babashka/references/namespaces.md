# Babashka Namespace Reference

## Built-in (available without :deps)

| Namespace | Purpose | Key Functions |
|---|---|---|
| `babashka.fs` | File/path operations | `glob`, `copy`, `copy-tree`, `move`, `delete-tree`, `create-dirs`, `exists?`, `which`, `modified-since`, `with-temp-dir` |
| `babashka.process` | External commands | `shell`, `process`, `destroy-tree`, `pipeline` |
| `babashka.cli` | CLI argument parsing | `parse-opts`, `parse-args`, `format-opts` |
| `babashka.http-client` | HTTP requests | `get`, `post`, `put`, `delete` |
| `babashka.http-server` | Local HTTP serving | `serve` (returns stop-fn) |
| `cheshire.core` | JSON (built-in) | `generate-string`, `parse-string` |
| `clojure.data.json` | JSON (alternative) | `read-str`, `write-str` |
| `clojure.data.csv` | CSV parsing/writing | `read-csv`, `write-csv` |
| `clojure.data.xml` | XML parsing | `parse`, `emit` |
| `clojure.edn` | EDN reading | `read-string` |
| `clojure.java.io` | Stream I/O | `writer`, `reader`, `file`, `copy` |
| `clojure.string` | String operations | `split`, `join`, `trim`, `blank?`, `replace` |
| `clojure.set` | Set operations | `union`, `intersection`, `difference` |
| `clojure.walk` | Tree traversal | `postwalk`, `prewalk`, `keywordize-keys` |
| `clojure.pprint` | Pretty printing | `pprint`, `print-table` |
| `clojure.test` | Unit testing | `deftest`, `is`, `testing`, `run-tests` |
| `clojure.tools.cli` | CLI parsing (older) | `parse-opts` |
| `cognitect.transit` | Transit format | `read`, `write`, `reader`, `writer` |
| `clj-yaml.core` | YAML parsing | `parse-string`, `generate-string` |
| `clojure.core.async` | Async channels | `go`, `chan`, `<!!`, `>!!`, `alt!` |
| `org.httpkit.client` | HTTP client (alt) | `get`, `post`, `request` |
| `org.httpkit.server` | HTTP server (alt) | `run-server` |
| `babashka.curl` | HTTP with streaming | `get`, `post`, `head` |

## Shell command replacements

| Shell | Babashka |
|---|---|
| `find . -name "*.clj"` | `(fs/glob "." "**.clj")` |
| `cp -r src/ dst/` | `(fs/copy-tree "src" "dst")` |
| `rm -rf dir/` | `(fs/delete-tree "dir")` |
| `mkdir -p a/b/c` | `(fs/create-dirs "a/b/c")` |
| `which cmd` | `(fs/which "cmd")` |
| `mv old new` | `(fs/move "old" "new")` |
| `curl URL` | `(:body (http/get URL))` |
| `python -m http.server` | `(server/serve {:port 8080 :dir "."})` |

## babashka.fs extended examples

```clojure
;; Temporary directories with automatic cleanup
(fs/with-temp-dir [tmp {:prefix "my-script"}]
  (process-files tmp))  ; tmp deleted on exit

;; Compression
(fs/gunzip "data.xml.gz" ".")
(fs/gzip "large-file.txt")

;; Streaming compressed XML
(with-open [in (java.util.zip.GZIPInputStream. (io/input-stream path))]
  (xml/parse in))
```

## babashka.http-client patterns

```clojure
;; GET with headers and params
(http/get url {:headers {"Authorization" "Bearer token"}
               :query-params {:limit 10}})

;; POST JSON
(http/post url {:headers {:content-type "application/json"}
                :body (json/generate-string data)})
```

Use `http-client` for API calls. Use `babashka.curl` for large file downloads.

## Addable deps (require :deps in bb.edn)

| Dep | Coordinates | Use Case |
|---|---|---|
| `org.babashka/cli` | `{:mvn/version "0.8.x"}` | Newer CLI version |
| `medley/medley` | `{:mvn/version "1.4.0"}` | Extra collection utilities |
| `selmer/selmer` | `{:mvn/version "1.12.x"}` | Template rendering |
| `metosin/malli` | `{:mvn/version "0.15.x"}` | Data validation |
| `borkdude/rewrite-edn` | `{:mvn/version "0.4.8"}` | Programmatic EDN rewriting |
| `djblue/portal` | `{:mvn/version "0.40.0"}` | Data navigation/visualization |
| `com.github.seancorfield/honeysql` | `{:mvn/version "2.4.x"}` | SQL generation |
| `hiccup/hiccup` | `{:mvn/version "2.0.0-RC4"}` | HTML generation from data |
| `aero/aero` | `{:mvn/version "1.1.6"}` | Configuration with profiles |
| `camel-snake-kebab/camel-snake-kebab` | `{:mvn/version "0.4.2"}` | Case conversion utilities |
