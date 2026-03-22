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