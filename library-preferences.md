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