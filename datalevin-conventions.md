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