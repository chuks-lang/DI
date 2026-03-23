# di

A dependency injection container for Chuks. Boot-time wiring with singleton, transient, and instance bindings. Circular dependency detection. Zero runtime overhead after boot.

## Installation

```bash
chuks add di
```

## Quick Start

```chuks
import { Container } from "pkg/di"

var c = new Container()

// Register a sync singleton
c.singleton("config", function(c: Container): any {
    return { "port": 3000, "env": "production" }
})

// Register an async singleton (e.g. database)
c.singletonAsync("db", async function(c: Container): Task<any> {
    return await connectToDatabase(c.resolve("config"))
})

// Register a plain value
c.instance("appName", "my-service")

// Boot — resolves all singletons, detects circular deps
await c.bootAsync()

// Resolve — single map read, zero overhead
var db = c.resolve("db")
var port: int = c.resolve("config").port
```

## API Reference

### `new Container()`

Create a new empty container.

```chuks
var c = new Container()
```

### `.singleton(name, factory)`

Register a singleton with a synchronous factory. The factory receives the container so it can resolve other dependencies. Runs **once** during `bootAsync()`, result cached forever.

```chuks
c.singleton("logger", function(c: Container): any {
    return new Logger(c.resolve("config"))
})
```

### `.singletonAsync(name, factory)`

Register a singleton with an async factory. Use this for dependencies that require `await` (database connections, external services).

```chuks
c.singletonAsync("userRepo", async function(c: Container): Task<any> {
    var repo = new UserRepo()
    await repo.useConnection(c.resolve("db"))
    return repo
})
```

### `.transient(name, factory)`

Register a transient binding. A **new instance** is created on every `resolve()` call.

> **Warning:** Do not resolve transients in request handlers. Resolve once at boot and pass the reference.

```chuks
c.transient("requestId", function(c: Container): any {
    return generateUuid()
})
```

### `.instance(name, value)`

Register a plain value — no factory, no overhead.

```chuks
c.instance("port", 3000)
c.instance("version", "1.0.0")
```

### `.bootAsync()`

Resolve all `singleton` and `singletonAsync` bindings in registration order. After boot:

- All singleton factories are dropped (GC reclaims closures)
- `resolve()` is a single map read
- No further registrations are allowed

```chuks
await c.bootAsync()
```

### `.resolve(name)`

Resolve a dependency by name. After `bootAsync()`, this is a **single map read**.

```chuks
var db = c.resolve("db")
var port = c.resolve("port")
```

Throws if the binding does not exist:

```
Error: Container: 'unknown' is not registered
```

### `.resolveAsync(name)`

Resolve a dependency asynchronously. Required for `singletonAsync` bindings before `bootAsync()` is called.

```chuks
var db = await c.resolveAsync("db")
```

### `.has(name)`

Check if a binding exists.

```chuks
if (c.has("db")) {
    println("db is registered")
}
```

### `.keys()`

List all registered binding names.

```chuks
var names: []string = c.keys()
// ["config", "db", "userRepo", "port"]
```

## Circular Dependency Detection

The container detects circular dependencies during `bootAsync()` and throws with the full cycle path:

```chuks
var c = new Container()

c.singleton("a", function(c: Container): any {
    return c.resolve("b")   // a needs b
})

c.singleton("b", function(c: Container): any {
    return c.resolve("a")   // b needs a — cycle!
})

await c.bootAsync()
// Error: Circular dependency detected: a → b → a
```

Three-way cycles are also caught:

```chuks
c.singleton("a", function(c: Container): any { return c.resolve("b") })
c.singleton("b", function(c: Container): any { return c.resolve("c") })
c.singleton("c", function(c: Container): any { return c.resolve("a") })

await c.bootAsync()
// Error: Circular dependency detected: a → b → c → a
```

## Real-World Example

```chuks
import { Container } from "pkg/di"
import { createServer, Request, Response } from "std/http"
import { json } from "std/json"
import { dbConnection } from "./db/dbConnection"
import { UserRepo } from "./repository/userRepository"
import { UserService } from "./services/userService"
import { loggerMiddleware } from "./middleware/logger.chuks"
import { authMiddleware } from "./middleware/auth.chuks"
import { corsMiddleware } from "./middleware/cors.chuks"
import { registerHealthRoutes } from "./routes/healthRoutes.chuks"
import { registerUserRoutes } from "./routes/userRoutes.chuks"

async function main(): Task<any> {
    var c = new Container()

    // Infrastructure
    c.singleton("db", function(c: Container): any {
        return dbConnection()
    })
    c.instance("port", 3000)

    // Repositories
    c.singletonAsync("userRepo", async function(c: Container): Task<any> {
        var repo = new UserRepo()
        await repo.useConnection(c.resolve("db"))
        return repo
    })

    // Services
    c.singletonAsync("userService", async function(c: Container): Task<any> {
        var repo = await c.resolveAsync("userRepo")
        return new UserService(repo)
    })

    // Boot — resolve everything, detect cycles, drop factories
    await c.bootAsync()

    // Grab direct references — container's job is done
    var userService = c.resolve("userService")

    // Server setup
    var app = createServer()
    app.use(corsMiddleware)
    app.use(loggerMiddleware)
    app.use(authMiddleware)

    app.get("/", function(req: Request, res: Response) {
        res.json(json.stringify({ "service": "userservice", "version": "1.0.0" }))
    })

    registerHealthRoutes(app)
    registerUserRoutes(app, userService)

    println("Starting on port " + string(c.resolve("port")))
    app.listen(c.resolve("port"))
    return null
}
```

## Performance

The container is a **boot-time wiring tool**. After `bootAsync()`:

| Operation        | Cost                                        |
| ---------------- | ------------------------------------------- |
| `resolve("key")` | 1 map read (~5ns)                           |
| Request handling | 0 container ops — services are local vars   |
| Memory           | One `map[string]any` with cached singletons |
| GC pressure      | Factory closures dropped after boot         |

The container adds **zero overhead** to request throughput. All services are resolved once at startup and passed as direct references.

## Binding Lifetimes

| Method           | When factory runs      | Result cached | Use case                          |
| ---------------- | ---------------------- | ------------- | --------------------------------- |
| `singleton`      | Once at `bootAsync()`  | Yes           | Services, repos, DB connections   |
| `singletonAsync` | Once at `bootAsync()`  | Yes           | Async init (DB, external APIs)    |
| `transient`      | Every `resolve()` call | No            | Utility instances (use sparingly) |
| `instance`       | Never (direct value)   | Yes           | Config, constants, ports          |
