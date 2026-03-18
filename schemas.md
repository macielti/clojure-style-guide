# Schema Definitions

This guide documents the preferred pattern for defining schemas across three layers: domain models, wire (in/out), and the Datalevin/Datomic DB layer.

---

## Layer Structure

Each entity has schemas in four namespaces:

| Layer | Namespace | Purpose |
|---|---|---|
| `models/` | Domain model | Core types, native Java/Clojure types |
| `wire/in/` | Incoming wire | Permissive types for deserialization |
| `wire/out/` | Outgoing wire | Serialized types for API responses |
| `wire/datalevin/` | DB layer | Namespaced keys; contains both the Datalevin attribute map and plumatic schema |

---

## Tolerant Reader

Follow the **Tolerant Reader** pattern when consuming external data: only validate what you need and ignore unknown fields. This decouples your service from the full structure of the producer — additive changes on the producer side won't break your consumers.

Plumatic Schema map validation is **strict by default**: extra keys cause validation failure. Use `common-clj.schema.core/loose-schema` to make a schema tolerant — it walks the schema recursively and adds `{s/Keyword s/Any}` to every map, so unknown keys are accepted at any nesting level.

This applies to the `wire/in/` layer (incoming HTTP payloads from external clients or services). Wrap the schema in `loose-schema` at the `s/defschema` call site:

```clojure
(ns myapp.wire.in.platform-a.product
  (:require [common-clj.schema.core :as common.schema]
            [common-clj.schema.extensions :as schema.extensions]
            [schema.core :as s]))

(def fetched-product-platform-a
  {:itemName      s/Str
   :itemCategory  (s/enum "type-a" "type-b" "type-c")
   :isAvailable   s/Bool
   :basePrice     s/Num
   :itemCode      s/Str
   :itemId        s/Int
   :minQuantity   s/Num
   :taxRate       s/Num
   :discountRate  s/Num})

(s/defschema FetchedProductPlatformA
  (common.schema/loose-schema fetched-product-platform-a))
```

The plain `def` map stays strict — `loose-schema` is applied only at the `s/defschema` boundary. Do **not** use `loose-schema` in `wire/out/` or `models/` — those are producer-side schemas where strictness is desirable.

---

## 1. Models Layer

The `models/` layer defines the canonical domain shape using native Clojure/Java types.

### Enum / Domain Value Sets

Define raw Clojure sets first, then derive schema types from them. The raw set can be reused in business logic; the schema type is for validation.

```clojure
(ns myapp.models.product
  (:require [common-clj.schema.extensions :as schema.extensions]
            [schema.core :as s])
  (:import (java.time Instant)))

(def categories #{:category-a :category-b :category-c})
(def tiers #{:tier-a :tier-b})
(def platforms #{:platform-a :platform-b})

(def Category (apply s/enum categories))
(def Tier (apply s/enum tiers))
(def Platform (apply s/enum platforms))
```

- Enum sets: lowercase plural (`categories`, `tiers`)
- Enum schema types: PascalCase singular (`Category`, `Tier`)

### Base Map → Platform Variants → Conditional Schema

Define a `base-{entity}` plain `def` map, then derive platform-specific variants via `merge` + a platform discriminator using `(s/eq :platform-kw)`. Wrap each variant in `s/defschema`. Define a unified conditional schema using `s/conditional` that dispatches on the discriminator.

```clojure
(def base-product
  {:id         s/Uuid
   :category   Category
   :name       s/Str
   :tier       Tier
   :price      BigDecimal
   :available? s/Bool
   :sku        s/Str
   :quantity   BigDecimal
   :created-at Instant
   :ref-date   schema.extensions/CalendarDateWire})

(def product-platform-a
  (merge base-product
         {:platform (s/eq :platform-a)}))
(s/defschema ProductPlatformA
  product-platform-a)

(def product-platform-b
  (merge base-product
         {:platform (s/eq :platform-b)}))
(s/defschema ProductPlatformB
  product-platform-b)

(s/defschema Product
  (s/conditional #(= (-> % :platform) :platform-a) ProductPlatformA
                 #(= (-> % :platform) :platform-b) ProductPlatformB))
```

### Optional Keys

Use `(s/optional-key :key)` for nullable or absent fields.

```clojure
(ns myapp.models.order
  (:require [common-clj.schema.extensions :as schema.extensions]
            [schema.core :as s])
  (:import (java.time Instant)))

(def statuses #{:placed :scheduled :failed})
(def Status (apply s/enum statuses))

(def order
  {:id                            s/Uuid
   :customer-id                   s/Uuid
   :product-id                    s/Uuid
   :amount                        BigDecimal
   :reference-date                schema.extensions/CalendarDateWire
   :status                        Status
   :scheduled-at                  Instant
   (s/optional-key :placed-at)    Instant
   (s/optional-key :failed-at)    Instant})

(s/defschema Order order)
```

### Type Rules for `models/`

- Time: `java.time.Instant` (requires `(:import (java.time Instant))`)
- Money: `BigDecimal`
- IDs: `s/Uuid`
- Booleans: `s/Bool`
- Calendar dates (date-only strings): `schema.extensions/CalendarDateWire`

---

## 2. Wire In Layer

The `wire/in/` layer validates incoming data. Types are more permissive to accommodate what clients send.

```clojure
(ns myapp.wire.in.product
  (:require [myapp.models.product :as models.product]
            [common-clj.schema.extensions :as schema.extensions]
            [schema.core :as s]))

(def base-product
  {:id         s/Uuid
   :category   models.product/Category
   :tier       models.product/Tier
   :price      s/Num
   :name       s/Str
   :ref-date   schema.extensions/CalendarDateWire
   :created-at schema.extensions/InstantWire
   :platform   models.product/Platform
   :available? s/Bool
   :sku        s/Str
   :quantity   s/Num})

(def product-platform-a
  (merge base-product {:platform (s/eq :platform-a)}))
(s/defschema ProductPlatformA
  product-platform-a)

(def product-platform-b
  (merge base-product {:platform (s/eq :platform-b)}))
(s/defschema ProductPlatformB
  product-platform-b)

(s/defschema Product
  (s/conditional #(= (-> % :platform keyword) :platform-a) ProductPlatformA
                 #(= (-> % :platform keyword) :platform-b) ProductPlatformB))
```

### Type Rules for `wire/in/`

- Time (instants): `schema.extensions/InstantWire` (string representation)
- Numbers: `s/Num` instead of `BigDecimal` (accepts any numeric type from JSON)
- Enum types: reuse from `models/` namespace (keywords)
- Calendar dates: `schema.extensions/CalendarDateWire`
- Unknown keys: wrap with `common.schema/loose-schema` at `s/defschema` (Tolerant Reader — see above)

---

## 3. Wire Out Layer

The `wire/out/` layer validates outgoing API responses. Enums are re-derived as string names since JSON serializes keywords as strings.

```clojure
(ns myapp.wire.out.product
  (:require [myapp.models.product :as models.product]
            [common-clj.schema.extensions :as schema.extensions]
            [schema.core :as s]))

(def Category (->> models.product/categories
                   (map #(name %))
                   (apply s/enum)))

(def Tier (->> models.product/tiers
               (map #(name %))
               (apply s/enum)))

(def base-product
  {:id         schema.extensions/UuidWire
   :category   Category
   :name       s/Str
   :tier       Tier
   :price      BigDecimal
   :available? s/Bool
   :sku        s/Str
   :quantity   BigDecimal
   :created-at schema.extensions/InstantWire
   :ref-date   schema.extensions/CalendarDateWire})

(def product-platform-a
  (merge base-product
         {:platform (s/eq "platform-a")}))
(s/defschema ProductPlatformA
  product-platform-a)

(def product-platform-b
  (merge base-product
         {:platform (s/eq "platform-b")}))
(s/defschema ProductPlatformB
  product-platform-b)

(s/defschema Product
  (s/conditional #(= (-> % :platform) "platform-a") ProductPlatformA
                 #(= (-> % :platform) "platform-b") ProductPlatformB))
```

### Type Rules for `wire/out/`

- IDs: `schema.extensions/UuidWire` (string)
- Time (instants): `schema.extensions/InstantWire` (string)
- Enums: re-derived from the raw model set via `(map name ...)` + `apply s/enum` → produces string enum
- Platform discriminator: `(s/eq "platform-a")` — string, not keyword
- Numbers: `BigDecimal` (exact, for financial data)

---

## 4. Datalevin Layer

The `wire/datalevin/` layer has two parts per entity in the same namespace:

1. **`{entity}-skeleton`** — a plain `def` map of Datalevin attribute definitions
2. **Plumatic schema** — `s/defschema` using namespaced keys that mirror the skeleton

### Skeleton (Datalevin Attribute Map)

Named `{entity}-skeleton`. Keys use namespaced format `:{entity}/attr`. The identity attribute gets `:db/unique :db.unique/identity`.

```clojure
(def product-skeleton
  {:product/id         {:db/valueType :db.type/uuid
                         :db/unique    :db.unique/identity}
   :product/category   {:db/valueType :db.type/keyword}
   :product/name       {:db/valueType :db.type/string}
   :product/tier       {:db/valueType :db.type/keyword}
   :product/price      {:db/valueType :db.type/bigdec}
   :product/available? {:db/valueType :db.type/boolean}
   :product/platform   {:db/valueType :db.type/keyword}
   :product/created-at {:db/valueType :db.type/instant}
   :product/sku        {:db/valueType :db.type/string}
   :product/quantity   {:db/valueType :db.type/bigdec}
   :product/ref-date   {:db/valueType :db.type/string}})
```

### Plumatic Schema (Namespaced Keys)

Mirrors the skeleton keys exactly. The same base map → platform variants → conditional schema pattern applies, using namespaced keys throughout.

```clojure
(ns myapp.wire.datalevin.product
  (:require [myapp.models.product :as models.product]
            [common-clj.schema.extensions :as schema.extensions]
            [schema.core :as s]))

(def base-product
  {:product/id         s/Uuid
   :product/category   models.product/Category
   :product/name       s/Str
   :product/tier       models.product/Tier
   :product/price      BigDecimal
   :product/available? s/Bool
   :product/created-at s/Inst
   :product/sku        s/Str
   :product/quantity   BigDecimal
   :product/ref-date   schema.extensions/CalendarDateWire})

(def product-platform-a
  (merge base-product
         {:product/platform (s/eq :platform-a)}))
(s/defschema ProductPlatformA
  product-platform-a)

(def product-platform-b
  (merge base-product
         {:product/platform (s/eq :platform-b)}))
(s/defschema ProductPlatformB
  product-platform-b)

(s/defschema Product
  (s/conditional #(= (-> % :product/platform) :platform-a) ProductPlatformA
                 #(= (-> % :product/platform) :platform-b) ProductPlatformB))
```

### Order Example (with Optional Keys and Namespaced Enums)

```clojure
(def order-skeleton
  {:order/id           {:db/valueType :db.type/uuid
                         :db/unique    :db.unique/identity}
   :order/customer-id  {:db/valueType :db.type/uuid}
   :order/product-id   {:db/valueType :db.type/uuid}
   :order/amount       {:db/valueType :db.type/bigdec}
   :order/ref-date     {:db/valueType :db.type/string}
   :order/status       {:db/valueType :db.type/keyword}
   :order/scheduled-at {:db/valueType :db.type/instant}
   :order/placed-at    {:db/valueType :db.type/instant}
   :order/failed-at    {:db/valueType :db.type/instant}})

(def statuses #{:order.status/placed
                :order.status/scheduled
                :order.status/failed})
(def Status (apply s/enum statuses))

(def order
  {:order/id                            s/Uuid
   :order/customer-id                   s/Uuid
   :order/product-id                    s/Uuid
   :order/amount                        BigDecimal
   :order/ref-date                      schema.extensions/CalendarDateWire
   :order/status                        Status
   :order/scheduled-at                  s/Inst
   (s/optional-key :order/placed-at)    s/Inst
   (s/optional-key :order/failed-at)    s/Inst})

(s/defschema Order order)
```

### Type Rules for `wire/datalevin/`

- Time (instants): `s/Inst` (not `java.time.Instant`)
- IDs: `s/Uuid`
- Numbers: `BigDecimal`
- Enums: reuse from `models/` namespace (keywords); in Datalevin layer, enum values may be namespaced (`:order.status/placed`)
- Optional fields: `(s/optional-key :ns/attr)` in the plumatic schema (skeleton always declares all attributes)

---

## Naming Conventions Summary

| Artifact | Convention | Example |
|---|---|---|
| Enum raw sets | lowercase plural `def` | `categories`, `statuses` |
| Enum schema types | PascalCase `def` | `Category`, `Status` |
| Base map | `{entity}` plain `def` | `base-product`, `order` |
| Platform variants | `{entity}-{platform}` plain `def` | `product-platform-a` |
| Public schema types | PascalCase `s/defschema` | `ProductPlatformA`, `Order` |
| Datalevin attribute map | `{entity}-skeleton` plain `def` | `product-skeleton`, `order-skeleton` |
