# Clojure Style Guide

# Library Preferences

## UUID String Validation

Use `common-clj.schema.extensions/uuid-string?` from the `net.clojars.macielti/common-clj` library instead of adding a separate `clj-uuid` dependency.

### Rule

When you need to validate a UUID string, import and use the predicate from `common-clj`:

```clojure
(require '[common-clj.schema.extensions :as schema.extensions])

(schema.extensions/uuid-string? "550e8400-e29b-41d4-a716-446655440000")
;; => true
```

Do not add `clj-uuid` as a separate dependency if `common-clj` is already on the classpath.

### Why

The `net.clojars.macielti/common-clj` library provides a `uuid-string?` predicate via `common-clj.schema.extensions`. If your project already depends on `common-clj`, adding `clj-uuid` as a separate dependency introduces unnecessary bloat and dependency duplication.

### How to Apply

- When you encounter UUID string validation needs in tests or source code, check whether `common-clj` is already in the dependencies
- If yes, use `common-clj.schema.extensions/uuid-string?`
- If no, add `common-clj` rather than adding a separate UUID validation library

# Integrant Patterns

## JVM Shutdown Hook Requirement

Every Clojure service using Integrant must register a JVM shutdown hook immediately after `ig/init` to gracefully halt all components on shutdown (SIGTERM/SIGINT).

### Rule

Always wrap `ig/init` in a function that registers a shutdown hook before returning the system:

```clojure
(defn start-system! []
  (let [system (ig/init components)]
    (.addShutdownHook (Runtime/getRuntime)
                      (Thread. #(ig/halt! system)))
    system))
```

Never return `ig/init` directly — always register the shutdown hook first.

### Why

Without a shutdown hook, Integrant components do not halt gracefully when the process receives a termination signal. This causes resource leaks (unclosed DB connections, HTTP sockets, message consumers, schedulers) and can lead to data loss if writes are in-flight when the process dies.

### How to Apply

- When building a new Clojure service with Integrant, wrap `ig/init` with the pattern above
- When reviewing a service startup, verify that the shutdown hook is present and calls `ig/halt!` on the initialized system
- Do not rely on implicit cleanup or finalizers — always use an explicit shutdown hook

# Datalevin/Datomic Conventions

## Wire Schema Maps

Do not explicitly define `:db/id` in Datalevin/Datomic wire schema maps (e.g., `wire.datomic.*` namespaces).

### Rule

Only include domain attributes in wire schema maps. Omit `:db/id` entirely.

**Don't:**

```clojure
(def DocumentSchema
  {(s/required-key :db/id) s/Int
   (s/required-key :document/id) s/Str
   (s/required-key :document/file-name) s/Str})
```

**Do:**

```clojure
(def DocumentSchema
  {(s/required-key :document/id) s/Str
   (s/required-key :document/file-name) s/Str})
```

### Why

`:db/id` is an internal Datalevin detail managed by the database, not a domain attribute. Including it in the schema muddies the line between wire format (what's sent over the network or stored) and internal database representation. Clean schemas make the domain model clearer and prevent confusion about which attributes belong to the application domain.

### How to Apply

When creating or reviewing `wire.datomic.*` Prismatic Schema maps (or any Datalevin wire schemas), only include the domain attributes. The database will manage `:db/id` automatically.

---

## Component Parameter Annotation

Any `s/defn` parameter that receives a Datalevin component must be annotated with `datalevin.component/DatalevinComponent`.

### Rule

Always require `[datalevin.component :as datalevin.component]` and annotate the parameter.

**Don't:**

```clojure
(s/defn insert!
  [document :- wire.document/DocumentSchema
   datalevin]
  ...)
```

**Do:**

```clojure
(s/defn insert!
  [document :- wire.document/DocumentSchema
   datalevin :- datalevin.component/DatalevinComponent]
  ...)
```

### Why

Consistent schema annotations make data flow explicit and allow Prismatic Schema to validate component types at runtime. Unannotated component parameters are invisible to schema validation and obscure the function's contract.

### How to Apply

Check every `s/defn` that accepts a Datalevin connection or component. Add the annotation and the `datalevin.component` require if missing.

---

## Domain vs Wire Enum Namespace Convention (Datalevin/Datomic only)

This rule applies only to the Datalevin/Datomic wire layer. Other wire layers (HTTP, RabbitMQ, etc.) follow their own conventions.

At the **models (domain) layer**, enum values must be un-namespaced keywords — e.g., `:pending`, `:extracted`, `:failed`.

At the **wire/datalevin (or wire/datomic) layer**, enum values must be namespaced — e.g., `:document.status/pending`.

Adapters (`adapters/`) are responsible for translating between the two representations.

### Rule

- `models/` schema: use plain un-namespaced keywords
- `wire/datalevin/` or `wire/datomic/` schema: use fully-qualified namespace form
- `adapters/`: map between them (`:extracted` → `:document.status/extracted` and back)

### Why

For Datalevin/Datomic, namespacing enum values is a persistence detail tied to the attribute's namespace, not a domain concern. Domain schemas should stay clean and portable across persistence layers.

### How to Apply

When defining an enum value:
- `models/` → `:pending`, `:extracted`, `:failed`
- `wire/datalevin/` → `:document.status/pending`, `:document.status/extracted`, `:document.status/failed`
- In adapters, explicitly map each value in both directions

# Testing Conventions

## Integration Test Fixtures

Use `common-test-clj.helpers.schema/generate` for all integration test fixtures. Never use static maps.

### Rule

Always generate fixture data from the schema. Override only the fields relevant to the test.

**Don't:**

```clojure
(def document-fixture
  {:document/id   "abc123"
   :document/name "file.pdf"
   :document/status :pending})
```

**Do:**

```clojure
(require '[common-test-clj.helpers.schema :as schema.helpers])

(schema.helpers/generate wire.document/DocumentSchema
                         {:document/status :pending})
```

### Why

Static maps become stale as schemas evolve, silently omitting required fields or carrying outdated structure. Generated fixtures always match the current schema and make the test's intent clearer — only the overridden fields are meaningful.

### How to Apply

Replace any static map fixture with `schema.helpers/generate`, passing the wire or domain schema and a map of only the fields the test cares about.

### Dependency

Add to `:dev :dependencies` in `project.clj`:

```clojure
[net.clojars.macielti/common-test-clj "7.1.0"]
```

---

## Datalevin DB Function Tests

Test functions in `diplomat/db/datalevin/` with unit tests using `datalevin.mock/database-connection-for-unit-tests!`. Do not use integration tests for these.

### Rule

- Tests for `diplomat/db/datalevin/` live in `test/unit/`, not `test/integration/`
- Use `datalevin.mock/database-connection-for-unit-tests!` to get an isolated connection
- No full Integrant system is needed

**Do:**

```clojure
(ns unit.diplomat.db.datalevin.document-test
  (:require [clojure.test :refer [deftest is]]
            [datalevin.mock :as datalevin.mock]
            [diplomat.db.datalevin.document :as db.document]))

(deftest insert!-test
  (let [conn (datalevin.mock/database-connection-for-unit-tests!)]
    (db.document/insert! {:document/id "abc"} conn)
    (is (some? (db.document/find-by-id "abc" conn)))))
```

### Why

Datalevin DB functions are pure data-layer operations. Testing them via full integration tests (with HTTP, RabbitMQ, etc.) adds unnecessary overhead and couples DB logic tests to unrelated infrastructure. The mock creates a real isolated Datalevin connection at a random `/tmp/datalevin/<uuid>` path, providing full fidelity with minimal setup.

### How to Apply

- Place DB unit tests under `test/unit/diplomat/db/datalevin/`
- Call `datalevin.mock/database-connection-for-unit-tests!` in the test body or a fixture
- Reserve `test/integration/` for higher-level flows (HTTP endpoints, consumer pipelines)
