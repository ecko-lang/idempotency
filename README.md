# idempotency - Ecko Std Lib Package

An Idempotency-Key store for mutating HTTP APIs, backed by [Ecko](https://ecko.sh)'s
embedded `std.sql`. Stripe-style semantics: the use case this exists for is
create-refund, or any other "run this exactly once, however many times the
client sends it" endpoint - a double-click, a client retry after a dropped
connection, or a proxy replaying a request must not double-pay.

No native code: it composes `std.sql`, `std.json`, `std.hash` and `std.time`.
You bring the database connection; this package brings the state machine.

## Install

```bash
ecko get github.com/ecko-lang/idempotency
```

This package is pure - it never opens, closes, or scopes a filesystem path
itself (the caller's already-open `db` handle does that), so it needs no
`grant` in your `ecko.json`.

```ecko
import idempotency
```

## Usage

```ecko
import std.sql
import idempotency

db = sql.open("app.db")          # or ":memory:" - any std.sql connection
idempotency.init(db)             # creates the idempotency_keys table once

fp = idempotency.fingerprint("POST", "/v1/refunds", { order_id: 42, amount: 500 })

result = idempotency.run(db, "idem-key-from-header", fp, fn() {
    charge_refund(42, 500)       # runs exactly once per key
})
```

Call `run` again with the same key and the same fingerprint - a double-click,
a client retry, a load balancer replaying a request - and it returns the
stored `result` without calling the function again:

```ecko
same = idempotency.run(db, "idem-key-from-header", fp, fn() charge_refund(42, 500))
# same == result; charge_refund was NOT called a second time
```

Reuse the same key for a genuinely different request and `run` refuses,
rather than guess which request you meant:

```ecko
other_fp = idempotency.fingerprint("POST", "/v1/refunds", { order_id: 42, amount: 999 })
idempotency.run(db, "idem-key-from-header", other_fp, fn() charge_refund(42, 999))
# throws { kind: "idempotency", reason: "mismatch" }
```

## API

| Function | Description |
|---|---|
| `init(db)` | Create the `idempotency_keys` table if it does not exist. Call once per connection, before the first `run`. |
| `run(db, key, request_fingerprint, action, opts?)` | Run `action()` exactly once per `key`; replay, refuse, or run as described below. Returns `action()`'s result. |
| `fingerprint(method, path, body)` | A sha256 hex digest of `method`, `path` and `body` (JSON-encoded). Use it to build `request_fingerprint`. |
| `middleware(db, opts?)` | A `fn(req, next)` for `std.web`'s `web.router(routes, [middleware])`. See "Middleware" below. |

### `run` semantics

- **First call for a key**: `action()` runs, its return value is stored, and
  it is returned.
- **Repeat, same fingerprint**: the stored value is returned; `action` is
  **not** called again.
- **Repeat, different fingerprint**: throws
  `{ kind: "idempotency", reason: "mismatch", message: "..." }`. The key was
  reused for a different request - refusing is safer than guessing which
  request should win.
- **Repeat while the first call is still running**: throws
  `{ kind: "idempotency", reason: "in_progress", message: "..." }` rather than
  running `action` concurrently for the same key.
- **`action` throws**: the key's row is deleted before the error is
  re-thrown, so a retry with the same key runs `action` again. This package
  does not store failures - matching Stripe, which does not treat a
  5xx-like failure as a result worth replaying. If your own failures should
  sometimes be remembered (a well-formed 4xx, say), catch them inside
  `action` and return a normal value describing the failure instead of
  throwing.
- **`action`'s return value must be JSON-serializable** - it round-trips
  through `json.encode`/`json.decode` to survive a replay. A map, list,
  string, number, bool or null all work; a function, a cell, bytes or a
  stream do not.

### `opts`

Both `run` and `middleware` take the same options map, every key optional:

| Key | Default | Meaning |
|---|---|---|
| `ttl_seconds` | `86400` (24h) | How long a key is remembered. `0` means never expire. |
| `now` | `std.time.now` | A zero-argument function returning the current time in unix milliseconds. Inject a fixed or stepped clock in tests instead of sleeping on a real one. |

An expired key is treated as if it never existed: the next `run` call for it
starts over, running `action` and overwriting the old row.

### Middleware

`middleware(db, opts?)` returns a `fn(req, next)` for `web.router`'s
middleware list:

```ecko
import std.http
import std.web
import idempotency

app = web.router(
    [web.post("/v1/refunds", fn(req) http.json(create_refund(req.json)))],
    [idempotency.middleware(db)],
)
http.serve(8080, app)
```

For `POST`, `PATCH` and `DELETE` requests carrying an `Idempotency-Key`
header, it fingerprints the request (`method`, `path`, `req.body`) and wraps
`next(req)` with `run`, so a repeat request replays the earlier response
instead of re-running the handler. Requests with no `Idempotency-Key` header,
and every `GET`/`PUT`/`HEAD` request, pass straight through to `next`
untouched - idempotency-by-key is opt-in per request, the way Stripe's own
API treats it.

**The handler's return value must be JSON-serializable**, same as `action`
above. A plain `http.json(...)` or `http.text(...)` response - a map of
status/headers/body - round-trips fine. A streamed response (`stream: ch`) or
one with a raw bytes body does not survive `json.encode`/`json.decode`, so
don't put this middleware in front of a route that returns one; write that
route's own idempotency handling with `run` directly instead, storing
whatever smaller, serializable receipt makes sense for it.

## Notes

**Concurrency is handled at the database, not in Ecko.** `run` claims a key
with `insert or ignore`, which either commits or is silently skipped by
`std.sql`/SQLite depending on whether the row already exists - there is no
window where two calls can both believe they won. The loser re-reads the row
and replays or refuses exactly as a later, sequential call would.

**Why failures aren't stored.** Storing a stored *success* is the entire
point - it is what makes a replay safe. Storing a stored *failure* would mean
a transient error (a timeout, a dropped connection) permanently poisons the
key, and the caller's only way out is a new key - worse than the double-click
this package exists to prevent. Releasing the key on failure means a retry
gets a clean attempt.

**Table shape.** `init` creates one table, `idempotency_keys`, columns `key`,
`fingerprint`, `status` (`"in_progress"` or `"completed"`), `response` (JSON
text, `null` while in progress), `created_at` and `expires_at` (unix
milliseconds, `0` meaning never). It is an implementation detail, not API -
don't query it directly; open an issue if you need something `run` doesn't
expose.

## Testing

```bash
ecko test
```

Offline and deterministic. Every test opens its own `:memory:` SQLite
database. Expiry is tested with an injected `now` function stepped by hand,
never a real sleep. The "call while in progress" case is exercised
single-threaded, by having `action` itself make a second `run` call for the
same key before returning - at that point the row is genuinely still
`"in_progress"`, which is what a concurrent caller would also see.

## License

MIT - see [LICENSE](LICENSE).
