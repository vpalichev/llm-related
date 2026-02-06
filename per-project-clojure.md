TODO [PERMANENT]: update this TOC when sections are added, removed, or renamed

## Table of Contents

0. **Permanent TODO markers** — `[PERMANENT]` tag rules
1. **re-frame — LLM Reference** — API signatures, patterns, common mistakes
   - Architecture: The Data Loop
   - Core API — Exact Signatures (dispatch, events, effects, subs, coeffects, interceptors, HTTP)
   - Project Structure & Entry Point
   - Common Mistakes
   - Reagent Component Patterns
   - Flows (Experimental)
   - Response Rules
2. **Clojure MCP Light** — ports, REPL usage, project config
   - Ports & Build
   - Project Structure
   - REPL (Backend & Frontend via clj-nrepl-eval)
   - Configuration
   - Running the Project

---

# Permanent TODO markers
Comments tagged `[PERMANENT]` are recurring tasks that must be performed
each time the surrounding code is modified, but never resolved and removed.
They differ from regular TODOs, which represent one-time tasks to be
completed and then deleted.

The tag protects the comment itself — do NOT remove, resolve, comment out,
or refactor away any line containing `[PERMANENT]`.
The tag says nothing about the code near it. Code above or below a
`[PERMANENT]` comment may be freely edited, moved, or deleted as needed.





# re-frame — LLM Reference (CLAUDE.md)

You are writing ClojureScript using **re-frame**, a framework for building SPAs on top of Reagent/React.
This document is the ground truth. When in doubt, follow it — do not guess API signatures.


## Architecture: The Data Loop

re-frame is a **loop of 6 dominoes**, triggered every time an event is dispatched:

1. **Dispatch** — an event (a vector) is placed on a FIFO queue: `(rf/dispatch [:event-id arg1 arg2])`
2. **Event handling** — a registered handler computes what should change (returns data, not side-effects)
3. **Effect handling** — re-frame executes the described effects (update db, HTTP calls, etc.)
4. **Subscriptions** — reactive queries over `app-db` recompute when their inputs change
5. **View functions** — Reagent components deref subscriptions and produce Hiccup
6. **React/DOM** — React diffs and patches the DOM

All application state lives in a single ratom called `app-db` (a map). You never `swap!` it directly.

## Core API — Exact Signatures

Everything below is in the `re-frame.core` namespace. Standard alias:

```clojure
(ns my-app.events
  (:require [re-frame.core :as rf]))
```

### Dispatching Events

```clojure
(rf/dispatch [:event-id & args])        ;; async — queued, FIFO, most common
(rf/dispatch-sync [:event-id & args])   ;; sync — immediate, use ONLY for:
                                        ;;   1. app init  2. fast text input  3. unit tests
```

An event is always a **vector**. First element is the event id (keyword). Remaining elements are payload.

### Registering Event Handlers

#### reg-event-db — pure db-to-db handler (no side effects)

```clojure
(rf/reg-event-db
  :event-id                 ;; keyword id
  (fn [db event]            ;; db = current app-db map, event = full event vector
    (assoc db :key "val"))) ;; MUST return the new db map
```

With interceptors and idiomatic destructuring:

```clojure
(rf/reg-event-db
  :event-id
  [interceptor1 interceptor2]            ;; optional interceptor chain (vector)
  (fn [db [_ arg1 arg2]]                 ;; _ skips event-id
    (assoc db :key arg1)))
```

Handler signature: `(fn [db event-vector] -> new-db)`

#### reg-event-fx — effectful handler (can describe side effects)

```clojure
(rf/reg-event-fx
  :event-id
  (fn [{:keys [db] :as cofx} [_ arg1]]  ;; 1st arg is COEFFECTS map (always has :db)
    {:db (assoc db :key arg1)            ;; MUST return an EFFECTS map
     :fx [[:dispatch [:other-event]]]})) ;; :fx is a vector of [effect-id payload] tuples
```

Handler signature: `(fn [coeffects-map event-vector] -> effects-map)`

**CRITICAL**: The first arg to an `-fx` handler is a **coeffects map** (not bare `db`).
Always destructure: `{:keys [db]}` or `{:keys [db] :as cofx}`.
The return value is an **effects map** (not a bare `db`).

### Built-in Effects (returned in effects maps)

| Effect key        | Payload                      | Notes                                                         |
|-------------------|------------------------------|---------------------------------------------------------------|
| `:db`             | new-db map                   | Always actioned **first** (guaranteed since v1.1.0).          |
| `:fx`             | vector of `[effect-id val]`  | Actions effects sequentially. `nil` entries are ignored.      |
| `:dispatch`       | event vector                 | Dispatch a single event. **Use inside `:fx` for multiples.**  |
| `:dispatch-later` | `{:ms 200 :dispatch [...]}`  | Delayed dispatch.                                             |

#### Using :fx (the modern, preferred approach)

```clojure
{:db new-db
 :fx [[:dispatch [:event-a]]
      [:dispatch [:event-b "arg"]]
      (when condition [:some-effect payload])  ;; nil is safely ignored
      [:dispatch-later {:ms 500 :dispatch [:delayed]}]]}
```

**DEPRECATED since v1.1.0**: `:dispatch-n` (use `:fx` with multiple `:dispatch` tuples instead).

Do NOT put `:dispatch` as a top-level key alongside `:fx` — put all dispatches inside `:fx`.

### Registering Subscriptions

#### reg-sub — the standard way to create subscriptions

**Variation 1: No input signal (reads directly from app-db)**

```clojure
(rf/reg-sub
  :query-id
  (fn [db query-vector]     ;; db = current app-db map
    (:some-key db)))         ;; returns derived value
```

**Variation 2: With `:<-` input signals (sugar)**

Single input — computation fn receives a **single value**:
```clojure
(rf/reg-sub
  :active-count
  :<- [:todos]                    ;; input signal (another subscription)
  (fn [todos query-vector]        ;; todos = value of :todos sub (NOT a vector)
    (count (remove :done? todos))))
```

Multiple inputs — computation fn receives a **vector of values**:
```clojure
(rf/reg-sub
  :visible-todos
  :<- [:todos]
  :<- [:visibility-filter]
  (fn [[todos vis-filter] query-vector]    ;; destructure the vector
    (filter (partial matches-filter? vis-filter) todos)))
```

**CRITICAL `:<-` rule**: One `:<-` = singleton value. Two or more `:<-` = vector of values.
This is the #1 source of bugs. If you have ONE `:<-`, the computation fn gets a bare value, not a 1-element vector.

**Variation 3: Signal function (most general)**

```clojure
(rf/reg-sub
  :query-id
  (fn [query-vector _]                    ;; signal fn — returns subscription(s)
    (rf/subscribe [:other-sub (:param query-vector)]))  ;; returns a single signal
  (fn [other-value query-vector]          ;; computation fn
    (transform other-value)))
```

Signal fn returns: singleton → comp fn gets singleton. Vector → comp fn gets vector. Map → comp fn gets map.

**Sugar: `:->` and `:=>`**

```clojure
(rf/reg-sub :todos :-> :todos)           ;; equivalent to (fn [db _] (:todos db))
(rf/reg-sub :sorted :-> (partial sort-by :date))
(rf/reg-sub :parameterized :=> my-2-arity-fn) ;; receives [input-values & query-params]
```

`:->` passes **only** the input (1-arity). `:=>` passes input AND query rest-args (multi-arity).

### Using Subscriptions in Views

```clojure
(defn my-component []
  (let [todos @(rf/subscribe [:todos])]   ;; MUST deref with @ in the render body
    [:ul (for [t todos]
           ^{:key (:id t)} [:li (:text t)])]))
```

**CRITICAL**: `subscribe` returns a Reagent Reaction. You MUST `deref` (`@`) it.
Only deref subscriptions inside **Reagent component render functions** — never in event handlers.

### Custom Effects

```clojure
;; Registration (once, at startup):
(rf/reg-fx
  :my-effect                      ;; effect id (keyword)
  (fn [payload]                   ;; handler — side-effecting, return value ignored
    (do-something-with payload)))

;; Usage (in an event handler's returned effects map):
{:db new-db
 :fx [[:my-effect {:some "data"}]]}
```

### Coeffects

Inject external data (time, localStorage, etc.) into event handlers without impurity:

```clojure
;; Register a coeffect handler:
(rf/reg-cofx
  :now
  (fn [cofx _]                           ;; cofx map in, cofx map out
    (assoc cofx :now (js/Date.))))

;; Inject into an event handler:
(rf/reg-event-fx
  :my-event
  [(rf/inject-cofx :now)]                ;; interceptor that injects :now
  (fn [{:keys [db now]} [_ arg]]         ;; now available in cofx map!
    {:db (assoc db :timestamp now)}))
```

`inject-cofx` returns an **interceptor**. Place it in the interceptor chain (2nd arg to reg-event-fx).

### Interceptors (built-in)

| Interceptor           | Purpose                                                        |
|-----------------------|----------------------------------------------------------------|
| `(rf/path [:a :b])`  | Narrow db to a sub-path. Handler sees/returns only that slice. |
| `rf/debug`            | `console.log` before/after db. Dev only.                       |
| `rf/trim-v`           | Remove event-id from event vector (handler gets `[arg1 arg2]` not `[:id arg1 arg2]`). |
| `rf/unwrap`           | Unwrap single-map payload: handler gets the map directly as 2nd arg. |
| `(rf/after f)`        | Run `(f db event)` after handler for side-effects. Return ignored. |
| `(rf/enrich f)`       | Run `(f db event)` after handler. Returns replacement db.      |
| `(rf/on-changes f out-path in1 in2 ...)` | Recompute `(f v1 v2)` when values at `in-paths` change. |

### HTTP Requests (day8.re-frame/http-fx)

This library registers an effect with key `:http-xhrio`. It wraps `cljs-ajax`.

```clojure
(ns my-app.events
  (:require [re-frame.core :as rf]
            [ajax.core :as ajax]
            [day8.re-frame.http-fx]))    ;; <-- require for side-effect (registers the effect handler)

(rf/reg-event-fx
  :fetch-data
  (fn [{:keys [db]} _]
    {:db (assoc db :loading? true)
     :http-xhrio {:method          :get
                   :uri             "https://api.example.com/data"
                   :timeout         8000                                        ;; ms, optional
                   :response-format (ajax/json-response-format {:keywords? true}) ;; REQUIRED
                   :on-success      [:fetch-data-success]                       ;; event vector
                   :on-failure      [:fetch-data-failure]}}))                    ;; event vector

(rf/reg-event-db
  :fetch-data-success
  (fn [db [_ result]]          ;; result is the response body (last element conj'd to event vector)
    (assoc db :loading? false :data result)))

(rf/reg-event-db
  :fetch-data-failure
  (fn [db [_ error]]           ;; error is a map with :status, :status-text, :failure, etc.
    (assoc db :loading? false :error error)))
```

**CRITICAL**: You MUST provide `:response-format` — it is NOT inferred.
The effect key is `:http-xhrio` (not `:http`). The `:http` key belongs to the alpha/experimental `http-fx-alpha` library.
For POST, also provide `:format` (e.g., `(ajax/json-request-format)`) and `:params`.

For multiple concurrent requests, pass a **vector** of option maps to `:http-xhrio`.

`:on-success` and `:on-failure` are **event vectors**. The response/error is `conj`'d as the last element.
You can pre-populate them with extra args: `:on-success [:my-handler 42 "extra"]` →
handler receives `[_ 42 "extra" response-data]`.

## Project Structure

```
my-app/
├── deps.edn
├── shadow-cljs.edn
├── package.json
├── resources/public/
│   └── index.html
└── src/my_app/
    ├── core.cljs       ;; entry point, mount-root, init
    ├── db.cljs          ;; default-db definition
    ├── events.cljs      ;; reg-event-db, reg-event-fx
    ├── subs.cljs        ;; reg-sub
    ├── views.cljs       ;; Reagent components
    └── fx.cljs          ;; reg-fx, reg-cofx (custom effects/coeffects)
```

### CRITICAL: Namespace Loading

`events.cljs` and `subs.cljs` register handlers **as side effects** at load time.
Google Closure Compiler will **tree-shake them away** unless they are explicitly required.

```clojure
;; core.cljs — MUST require these for their side effects:
(ns my-app.core
  (:require [my-app.events]          ;; required for side-effect registration
            [my-app.subs]            ;; required for side-effect registration
            [my-app.views :as views]
            [re-frame.core :as rf]
            [reagent.dom.client :as rdc]))
```

If your subscriptions or events silently do nothing, this is likely the cause.

### Entry Point Pattern

```clojure
(defonce root (rdc/create-root (.getElementById js/document "app")))

(defn ^:dev/after-load mount-root []    ;; shadow-cljs hot-reload hook
  (rf/clear-subscription-cache!)        ;; clear stale subs on reload
  (rdc/render root [views/app]))

(defn init []                           ;; called once on page load
  (rf/dispatch-sync [:initialize-db])   ;; dispatch-sync for init is correct
  (mount-root))
```

### Initialize DB Event

```clojure
;; events.cljs
(rf/reg-event-db
  :initialize-db
  (fn [_ _]              ;; ignore both args
    db/default-db))      ;; return the default db map

;; db.cljs
(def default-db
  {:todos   []
   :filter  :all})
```

## Common Mistakes — DO NOT DO THESE

| Mistake | What's Wrong | Correct |
|---------|-------------|---------|
| `(fn [db [_ x]] ...)` in `reg-event-fx` | First arg is **cofx map**, not db | `(fn [{:keys [db]} [_ x]] ...)` |
| Returning `db` from `reg-event-fx` handler | Must return an **effects map** | `{:db new-db}` |
| `{:dispatch [...] :dispatch [...]}` | Duplicate map key — second wins | `{:fx [[:dispatch [...]] [:dispatch [...]]]}` |
| `@(rf/subscribe [...])` in event handler | Memory leak, conceptual error | Use `db` from coeffects instead |
| `(rf/subscribe [:foo])` without `@` | Gets the reaction, not its value | `@(rf/subscribe [:foo])` |
| Not requiring events/subs namespaces | Handlers never register → silent failure | Require in `core.cljs` |
| Using `:http` as effect key | That's `http-fx-alpha` (experimental) | `:http-xhrio` with `http-fx` |
| Using `:dispatch-n` | Deprecated since v1.1.0 | Use `:fx` with `:dispatch` tuples |
| Single `:<-` expecting a vector | One `:<-` gives a bare value | Destructure as value, not `[value]` |
| `(rf/reg-sub :foo :some-key)` without `:->` | If `:some-key` not in db, returns the query-vector (!!!) | `(rf/reg-sub :foo :-> :some-key)` |

## Reagent Component Patterns

### Form-1 (simple, most common)

```clojure
(defn todo-item [id]
  (let [todo @(rf/subscribe [:todo id])]     ;; subscribe in let, deref with @
    [:li {:class (when (:done? todo) "done")}
     (:text todo)]))
```

### Form-2 (when you need setup before render)

```clojure
(defn editable-field []
  (let [local-val (reagent.core/atom "")]     ;; local state created once
    (fn []                                     ;; inner fn is the render function
      [:input {:value @local-val
               :on-change #(reset! local-val (-> % .-target .-value))}])))
```

### Event Handlers in Hiccup

```clojure
;; Anonymous fn wrapping dispatch (most common for parameterized):
[:button {:on-click #(rf/dispatch [:delete-todo id])} "Delete"]

;; For events with no extra args:
[:button {:on-click #(rf/dispatch [:clear-completed])} "Clear"]
```

## Flows (Experimental — re-frame.alpha)

Flows are an **experimental** feature for derived data within `app-db`. They require `re-frame.alpha`.

```clojure
(ns my-app.flows
  (:require [re-frame.alpha :as rf]))

(rf/reg-flow
  {:id     :total-price
   :inputs {:items [:cart :items]
            :tax   [:settings :tax-rate]}
   :output (fn [{:keys [items tax]}]
             (* (reduce + (map :price items)) (+ 1 tax)))
   :path   [:cart :total-price]})
```

Do NOT use Flows unless the user explicitly requests them. They are not yet part of stable `re-frame.core`.

## Response Rules

1. **Never invent API functions.** If you're unsure whether a function exists, say so.
2. **Always destructure `-fx` handlers correctly**: `(fn [{:keys [db]} event]`, NOT `(fn [db event]`.
3. **Always return the right shape**: `reg-event-db` → new db map. `reg-event-fx` → effects map.
4. **Use `:fx` for multiple effects**, not duplicate top-level keys.
5. **Require side-effect namespaces** (events, subs, http-fx) in the entry point.
6. **The effect key for HTTP is `:http-xhrio`**, not `:http`.
7. **`:response-format` is mandatory** in `:http-xhrio` — it is never inferred.
8. **Never deref subscriptions outside Reagent render context** (not in event handlers, not in `go` blocks).
9. When showing project setup, use **deps.edn + shadow-cljs**, not Leiningen.
10. Prefer **namespaced keywords** for event and subscription ids in real projects (e.g., `::fetch-data` or `:my-app.events/fetch-data`).


# Clojure MCP Light

# Ports

- BACKEND_REPL = 12888
- FRONTEND_REPL = 12889
- BACKEND_API = 12300
- FRONTEND_DEV = 12280

# Build

- SHADOW_BUILD = :app

# Project Structure

Full-stack Clojure/ClojureScript project:
- Backend (Clojure JVM): src/clj
- Frontend (ClojureScript browser): src/cljs
- Shared (CLJC): src/cljc

# REPL

Two separate REPLs - use the correct one based on what you're working on.

## Backend (Clojure JVM)

For code in src/clj:
```
clj-nrepl-eval -p <BACKEND_REPL> "<clojure-code>"
```

Example:
```
clj-nrepl-eval -p <BACKEND_REPL> "(+ 1 2 3)"
```

## Frontend (ClojureScript Browser)

IMPORTANT: `clj-nrepl-eval` does NOT persist REPL session state between calls.
This means `(shadow/repl :app)` won't help - the next call loses CLJS mode.

Use the shadow API to evaluate ClojureScript directly:
```
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"<cljs-code>\" {})"
```

Examples:
```
# Check browser is connected (should return list with browser info)
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>)"

# Get page HTML
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(.-innerHTML js/document.body)\" {})"

# Query DOM element
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(js/document.querySelector \\\"#app\\\")\" {})"

# Check app state (explore src/cljs to find the state atom location first)
```

Note: ClojureScript code is passed as a string, so escape inner quotes with \\"




### shadow-cljs nREPL via clj-nrepl-eval

#### Banner — NOT an error
The shadow-cljs nREPL middleware prints the following on every eval
from a CLJ-mode session (which is every session created by clj-nrepl-eval):
```
;; shadow-cljs repl is NOT in CLJS mode
;; use (shadow/active-builds) to list builds available
;; use (shadow/repl <build-id>) to jack into a CLJS repl
```
This is informational. It means the session is in CLJ mode, which is correct.
Check for `=>` in the output to determine success. Do NOT retry based on this banner.

#### Evaluating ClojureScript from CLJ mode
Use `shadow.cljs.devtools.api/cljs-eval` — a CLJ-side function that dispatches
to the connected JS runtime. No need to switch to CLJS mode via `(shadow/repl :app)`.
```bash
# Correct — bash double-quote outer, escaped inner
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(+ 1 2)\" {})"
# => {:results ["3"], :out "", :err "", :ns cljs.user}
```

#### Successful response format
A working cljs-eval returns: `=> {:results [...], :out "...", :err "...", :ns cljs.user}`
- `:results` — vector of stringified return values
- `:out` / `:err` — captured stdout/stderr from the JS runtime
- `=> nil` or empty results — JS runtime likely disconnected (browser tab closed or reloading)

#### String escaping in nested ClojureScript expressions
Each nesting level requires one additional layer of backslash escaping:
```bash
# Simple expression — one level of escaping
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(+ 1 2)\" {})"

# String inside CLJS — two levels of escaping
clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/cljs-eval <SHADOW_BUILD> \"(js/console.log \\\"hello\\\")\" {})"
```

3+ levels of nesting — DO NOT inline.
Define a helper function in a CLJS namespace and call it with simple arguments.

#### No result / nil result troubleshooting
If `cljs-eval` returns `nil` or no `=>` line:
1. Check runtime connectivity:
   `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/repl-runtimes <SHADOW_BUILD>)"`
   - Non-empty vector = browser connected (working)
   - Empty vector `[]` = no browser connected — open/refresh the app
2. Check build worker:
   `clj-nrepl-eval -p <FRONTEND_REPL> "(shadow.cljs.devtools.api/worker-running? <SHADOW_BUILD>)"`
   - `false` = watch not running, restart with `shadow-cljs watch app`
3. During hot-reload cycles the runtime briefly disconnects — retry after 1-2 seconds.



## General

Always use :reload when requiring namespaces.

# Configuration

- Config: config/main-config.edn
- Server runs on port <BACKEND_API>
- Frontend dev server runs on port <FRONTEND_DEV> (shadow-cljs)

# Running the Project

- Backend: `clj -M:nrepl`
- Frontend: `npx shadow-cljs watch <SHADOW_BUILD>`

