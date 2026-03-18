# Datalevin Operations

This guide documents the preferred patterns for writing Datalevin query and mutation files. It covers namespace setup, function signatures, query forms, result processing, and the transact!/db-after mutation pattern.

---

## Namespace & Requires

Query files live under `diplomat/db/datalevin/` and follow the naming convention `myapp.diplomat.db.datalevin.{entity}`.

Required namespaces:

```clojure
(ns myapp.diplomat.db.datalevin.item
  (:require [myapp.adapters.item :as adapters.item]
            [myapp.models.item :as models.item]
            [common-clj.time.parser :as time.parser]
            [datalevin.component]
            [datalevin.core :as d]
            [schema.core :as s])
  (:import (java.time Instant)))
```

- Always require the entity's **model** namespace and **adapter** namespace.
- `datalevin.component` is required without an alias (used only as a type).
- `datalevin.core :as d` — all Datalevin operations go through this alias.
- `common-clj.time.parser` and `(:import (java.time Instant))` are only needed when the entity has time fields being written.

---

## Function Signatures (Plumatic Schema)

All functions use `s/defn` with explicit return type annotations and annotated parameters.

```clojure
(s/defn insert! :- models.item/Item
  [item :- models.item/Item
   database-connection :- datalevin.component/DatalevinComponent]
  ...)
```

Rules:
- `s/defn` with `:- ReturnType` on the same line as the function name.
- Every parameter is annotated with `:-`.
- `database-connection` is always the **last** parameter.
- `database-connection` type is always `datalevin.component/DatalevinComponent`.
- Nullable single-result functions return `(s/maybe EntityType)`.
- Multi-result functions return `[EntityType]` (a vector schema).

---

## Query Form (`d/q`)

Always use the quoted Datalog vector form. Always pull all attributes with `(pull ?entity [*])` — never select individual attributes.

```clojure
(d/q '[:find (pull ?item [*])
       :in $ ?id
       :where [?item :item/id ?id]]
     (d/db database-connection)
     id)
```

Key rules:
- The query is a **quoted vector literal** `'[:find ...]`, not a map.
- `:where` conditions are top-level vectors inside the quoted form, **not** nested inside a `:where` vector. Each `[?entity :ns/attr ?var]` clause sits at the same indentation level as the `:where` keyword.
- Database value for **reads**: `(d/db database-connection)`.
- Database value for **writes** (after transact!): `db-after` — see [Mutation Pattern](#mutation-pattern-dtransact) below.
- `:in $` is always present, even when there are no additional bindings (the `$` refers to the database).

### Multi-condition `:where` example

```clojure
(d/q '[:find (pull ?item [*])
       :in $ ?customer-id ?status
       :where [?item :item/customer-id ?customer-id]
       [?item :item/status ?status]]
     (d/db database-connection)
     customer-id
     status)
```

Each `:where` clause is on its own line at the same indentation level as `[?item :item/customer-id ?customer-id]` — they are **siblings** inside the quoted vector, not children of a `:where` sub-vector.

---

## Result Processing Chains

The raw result of `d/q` with `(pull ?entity [*])` is a sequence of single-element vectors. The processing chain strips the wrapper, removes the internal `:db/id` key, and converts to the model type via the adapter.

### Single result — guaranteed present (`insert!`, `action!`)

Use `->` (thread-first):

```clojure
(-> (d/q '[:find (pull ?item [*])
           :in $ ?id
           :where [?item :item/id ?id]] db-after id)
    ffirst
    (dissoc :db/id)
    adapters.item/database->item)
```

`ffirst` = `(first (first result))` — takes the first result tuple, then the first (and only) element of that tuple.

### Single result — nullable (`lookup`)

Use `some->` to short-circuit on `nil`:

```clojure
(some-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] (d/db database-connection) id)
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)
```

If the query returns an empty sequence, `some->` returns `nil` without calling the subsequent steps.

### Multiple results (`by-{criteria}`, `all`)

Use `->>` (thread-last) with `mapv`:

```clojure
(->> (d/q '[:find (pull ?item [*])
            :in $ ?customer-id
            :where [?item :item/customer-id ?customer-id]] (d/db database-connection) customer-id)
     (mapv #(-> % first (dissoc :db/id)))
     (mapv adapters.item/database->item))
```

The anonymous function `#(-> % first (dissoc :db/id))` unwraps each result tuple and removes `:db/id`. The second `mapv` applies the adapter to every element.

---

## Mutation Pattern (`d/transact!`)

### Always query `db-after`, never `(d/db conn)`

After a write, **always** use the `db-after` value returned by `d/transact!` to read back the persisted entity. Never call `(d/db database-connection)` after a transact! — that would introduce a potential race condition and is semantically incorrect.

```clojure
(s/defn insert! :- models.item/Item
  [item :- models.item/Item
   database-connection :- datalevin.component/DatalevinComponent]
  (let [{:keys [db-after]} (d/transact! database-connection [(adapters.item/item->database item)])]
    (-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] db-after (:id item))
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)))
```

Pattern:
1. `(d/transact! database-connection [tx-data])` — wraps tx-data in a vector (Datalevin expects a collection of transactions).
2. Destructure `{:keys [db-after]}` from the result.
3. Query `db-after` (not `(d/db database-connection)`) to get the just-written entity.
4. Apply the standard single-result chain: `ffirst → (dissoc :db/id) → adapter/database->entity`.

### Partial update mutations (`action!`)

Status transitions and partial updates only write the identity key plus the changed fields — no full entity reconstruction needed:

```clojure
(s/defn confirm! :- models.item/Item
  [{:keys [id]} :- models.item/Item
   confirmed-at :- Instant
   database-connection :- datalevin.component/DatalevinComponent]
  (let [{:keys [db-after]} (d/transact! database-connection [{:item/id           id
                                                              :item/status       :item.status/confirmed
                                                              :item/confirmed-at (time.parser/instant->legacy-date confirmed-at)}])]
    (-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] db-after id)
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)))

(s/defn cancel! :- models.item/Item
  [{:keys [id]} :- models.item/Item
   cancelled-at :- Instant
   database-connection :- datalevin.component/DatalevinComponent]
  (let [{:keys [db-after]} (d/transact! database-connection [{:item/id           id
                                                              :item/status       :item.status/cancelled
                                                              :item/cancelled-at (time.parser/instant->legacy-date cancelled-at)}])]
    (-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] db-after id)
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)))
```

Rules for partial updates:
- Destructure only what you need from the entity parameter: `[{:keys [id]} :- models.item/Item ...]`.
- The transaction map contains **only** the identity key (`:item/id`) plus the fields being changed.
- Time values must be converted via `time.parser/instant->legacy-date` before writing — Datalevin does not accept `java.time.Instant` directly.

---

## Read vs Write: Database Value Summary

| Operation | Database value | Why |
|---|---|---|
| Read (`lookup`, `by-*`, `all`) | `(d/db database-connection)` | Current database snapshot |
| Write (`insert!`, `action!`) | `db-after` from `d/transact!` | Reflects the just-committed state |

Never mix these up — using `(d/db conn)` after a transact! may read stale data before the write is visible.

---

## Adapter Convention

### Write path: `entity->database`

Converts a model map (plain keys, native types) to a namespaced-key map suitable for Datalevin. Called inside `d/transact!`:

```clojure
(d/transact! database-connection [(adapters.item/item->database item)])
```

### Read path: `database->entity`

Converts a namespaced-key map (as returned by `(pull ?entity [*])`) back to a model map. Called at the end of the result chain:

```clojure
adapters.item/database->entity
```

### Multimethod adapters for platform/type variants

When an entity has platform or type variants, both `entity->database` and `database->entity` are **multimethods** dispatching on the discriminator key.

Write-path multimethod dispatches on the plain model key:

```clojure
(defmulti product->database :platform)

(s/defmethod product->database :platform-a :- wire.datalevin.product/ProductPlatformA
  [{:keys [id name category tier price available? sku quantity created-at ref-date platform]}
   :- models.product/ProductPlatformA]
  {:product/id         id
   :product/name       name
   :product/category   category
   :product/tier       tier
   :product/price      price
   :product/available? available?
   :product/sku        sku
   :product/quantity   quantity
   :product/created-at (time.parser/instant->legacy-date created-at)
   :product/ref-date   ref-date
   :product/platform   platform})

(s/defmethod product->database :platform-b :- wire.datalevin.product/ProductPlatformB
  [{:keys [id name category tier price available? sku quantity created-at ref-date platform]}
   :- models.product/ProductPlatformB]
  {:product/id         id
   :product/name       name
   :product/category   category
   :product/tier       tier
   :product/price      price
   :product/available? available?
   :product/sku        sku
   :product/quantity   quantity
   :product/created-at (time.parser/instant->legacy-date created-at)
   :product/ref-date   ref-date
   :product/platform   platform})
```

Read-path multimethod dispatches on the **namespaced** discriminator key:

```clojure
(defmulti database->product :product/platform)

(s/defmethod database->product :platform-a :- models.product/ProductPlatformA
  [{:product/keys [id name category tier price available? sku quantity created-at ref-date platform]}
   :- wire.datalevin.product/ProductPlatformA]
  {:id         id
   :name       name
   :category   category
   :tier       tier
   :price      price
   :available? available?
   :sku        sku
   :quantity   quantity
   :created-at (time.parser/legacy-date->instant created-at)
   :ref-date   ref-date
   :platform   platform})
```

The dispatch key differs between directions: `entity->database` dispatches on `:platform` (plain key); `database->entity` dispatches on `:product/platform` (namespaced key, as returned by Datalevin pull).

---

## Standard Function Set Per Entity

| Function | Signature | Threading | DB value |
|---|---|---|---|
| `insert!` | `entity × conn → entity` | `->` | `db-after` |
| `lookup` | `id × conn → (maybe entity)` | `some->` | `(d/db conn)` |
| `by-{criteria}` | `...criteria × conn → [entity]` | `->>` + mapv | `(d/db conn)` |
| `all` | `conn → [entity]` | `->>` + mapv | `(d/db conn)` |
| `{action}!` | `entity × ... × conn → entity` | `->` | `db-after` |

### `all` — no additional `:in` bindings

When fetching all records, omit extra bindings but keep `:in $`:

```clojure
(s/defn all :- [models.item/Item]
  [database-connection :- datalevin.component/DatalevinComponent]
  (->> (d/q '[:find (pull ?item [*])
              :in $
              :where [?item :item/id]] (d/db database-connection))
       (mapv #(-> % first (dissoc :db/id)))
       (mapv adapters.item/database->item)))
```

The `:where` clause `[?item :item/id]` without a binding variable matches all entities that have the `:item/id` attribute — effectively selecting all items.

---

## Complete Example: Item Entity

```clojure
(ns myapp.diplomat.db.datalevin.item
  (:require [myapp.adapters.item :as adapters.item]
            [myapp.models.item :as models.item]
            [common-clj.time.parser :as time.parser]
            [datalevin.component]
            [datalevin.core :as d]
            [schema.core :as s])
  (:import (java.time Instant)))

(s/defn insert! :- models.item/Item
  [item :- models.item/Item
   database-connection :- datalevin.component/DatalevinComponent]
  (let [{:keys [db-after]} (d/transact! database-connection [(adapters.item/item->database item)])]
    (-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] db-after (:id item))
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)))

(s/defn lookup :- (s/maybe models.item/Item)
  [id :- s/Uuid
   database-connection :- datalevin.component/DatalevinComponent]
  (some-> (d/q '[:find (pull ?item [*])
                 :in $ ?id
                 :where [?item :item/id ?id]] (d/db database-connection) id)
          ffirst
          (dissoc :db/id)
          adapters.item/database->item))

(s/defn by-customer-id :- [models.item/Item]
  [customer-id :- s/Uuid
   database-connection :- datalevin.component/DatalevinComponent]
  (->> (d/q '[:find (pull ?item [*])
              :in $ ?customer-id
              :where [?item :item/customer-id ?customer-id]] (d/db database-connection) customer-id)
       (mapv #(-> % first (dissoc :db/id)))
       (mapv adapters.item/database->item)))

(s/defn all :- [models.item/Item]
  [database-connection :- datalevin.component/DatalevinComponent]
  (->> (d/q '[:find (pull ?item [*])
              :in $
              :where [?item :item/id]] (d/db database-connection))
       (mapv #(-> % first (dissoc :db/id)))
       (mapv adapters.item/database->item)))

(s/defn confirm! :- models.item/Item
  [{:keys [id]} :- models.item/Item
   confirmed-at :- Instant
   database-connection :- datalevin.component/DatalevinComponent]
  (let [{:keys [db-after]} (d/transact! database-connection [{:item/id           id
                                                              :item/status       :item.status/confirmed
                                                              :item/confirmed-at (time.parser/instant->legacy-date confirmed-at)}])]
    (-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] db-after id)
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)))

(s/defn cancel! :- models.item/Item
  [{:keys [id]} :- models.item/Item
   cancelled-at :- Instant
   database-connection :- datalevin.component/DatalevinComponent]
  (let [{:keys [db-after]} (d/transact! database-connection [{:item/id           id
                                                              :item/status       :item.status/cancelled
                                                              :item/cancelled-at (time.parser/instant->legacy-date cancelled-at)}])]
    (-> (d/q '[:find (pull ?item [*])
               :in $ ?id
               :where [?item :item/id ?id]] db-after id)
        ffirst
        (dissoc :db/id)
        adapters.item/database->item)))
```
