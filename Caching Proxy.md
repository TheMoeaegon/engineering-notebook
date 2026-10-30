# Caching Proxy — Build Notes

Notes from building a small HTTP caching proxy with **Bun + TypeScript**. The goal
was to learn how HTTP caching actually works, not just to make requests faster.

Repo: [TheMoeaegon/caching-proxy](https://github.com/TheMoeaegon/caching-proxy)
Global command: `caching-proxy`

```bash
caching-proxy serve -o https://dummyjson.com -p 3000
caching-proxy clear-cache -p 3000
```

---

## 1. What I built

A reverse proxy that sits between a client (browser, curl) and an upstream origin
(API, CDN, static server).

It:

- forwards every request to the origin
- caches `GET` and `HEAD` responses
- respects `Cache-Control` (`no-store`, `no-cache`, `private`, `max-age`,
  `s-maxage`, `Expires`)
- computes `Age`
- revalidates stale entries with `ETag` / `Last-Modified`
- returns `304` to the client only when the client's own conditional matches
- adds `x-cache: HIT | MISS`
- overrides `server` to `caching-proxy`
- can clear its cache through `caching-proxy clear-cache -p <port>`

---

## 2. The big picture: three caches, not one

```
┌─────────────────────┐
│ Browser             │  private cache
│ (or curl)           │  own copy, own clock
└──────────┬──────────┘
           │  maybe If-None-Match / If-Modified-Since
           ▼
┌─────────────────────┐
│ caching-proxy       │  shared cache  ← this project
│                     │  own copy, own clock
└──────────┬──────────┘
           │  maybe If-None-Match / If-Modified-Since
           ▼
┌─────────────────────┐
│ CDN / upstream      │  shared cache
│                     │  own copy, own clock
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Origin server       │  source of truth
└─────────────────────┘
```

The important idea:

> Every cache in the chain stores its **own copy** and decides **independently**
> whether that copy is fresh.

A conditional request from the browser does **not** mean the proxy's copy is
stale. It only means the browser's copy reached the end of *its* lifetime (or the
user forced a reload). The proxy still checks its own clock.

The `Age` header exists to help these clocks agree, but they still diverge in
practice (more on this below).

---

## 3. Request lifecycle in this project

```
GET /users?limit=1
        │
        ▼
key = "GET /users?limit=1"
        │
        ▼
   cache.get(key)
        │
        ├── no entry ──────────────────────────────────────────┐
        │                                                      │
        │                                          fetch origin as GET
        │                                                      │
        │                                          cacheable?
        │                                            ├─ yes → store
        │                                            └─ no  → don't store
        │                                                      │
        │                                          200 + body, x-cache: MISS
        │
        ├── entry exists
        │        │
        │        ├── fresh (now < expiresAt)
        │        │        │
        │        │        ├── client sent matching validator → 304, HIT
        │        │        └── otherwise                       → 200 + body, HIT
        │        │
        │        └── stale
        │                 │
        │                 ▼
        │          revalidate with origin
        │          (If-None-Match or If-Modified-Since)
        │                 │
        │                 ├── 304 → refresh metadata
        │                 │         ├── client validator matches → 304, HIT
        │                 │         └── otherwise                → 200 body, HIT
        │                 │
        │                 └── 200 → replace entry
        │                           └──────────────────────────► 200, MISS
```

`HEAD` follows the same path, but the final response has no body. On a `HEAD`
miss, the proxy fetches as `GET` so it can populate a reusable cache entry.

---

## 4. Cache key and what is cacheable

### Key

```ts
// src/proxy/cache.ts
export function buildKey(req: http.IncomingMessage) {
    const method = req.method === "HEAD" ? "GET" : req.method;
    return `${method} ${req.url}`;
}
```

- Key = `METHOD URL`, including the query string.
- `HEAD` is normalized to `GET`, so a `HEAD` can reuse a `GET` entry.
- There is no `Vary` in the key (deliberately deferred).
- There is no host in the key because one proxy instance proxies one origin.

### Methods

| Method | Behavior |
| --- | --- |
| `GET` | cacheable, stored |
| `HEAD` | served from `GET` entry, headers only |
| everything else | forwarded untouched, never stored, `x-cache: MISS` |

Non-cacheable methods need their request body buffered first
(`src/proxy/body.ts`), because Node's `IncomingMessage` is a stream.

### Status codes

Only `2xx` responses are stored:

```ts
if (status < 200 || status >= 300) return false;
```

Real caches have per-status rules (301, 404 can be cached; 206 needs special
handling). This project keeps it simple.

### Directives that block storage

```ts
if (
    directives.has("no-store") ||
    directives.has("no-cache") ||
    directives.has("private")
) {
    return false;
}
```

`no-cache` is stricter here than the spec: the spec allows storing but requires
revalidation before every use. We simply don't store it, so every request is a
`MISS`.

---

## 5. `Cache-Control`

`Cache-Control` is the main instruction set for caches.

| Directive | Meaning | In this proxy |
| --- | --- | --- |
| `max-age=N` | fresh for N seconds (private caches) | used, unless `s-maxage` exists |
| `s-maxage=N` | fresh for N seconds (shared caches) | used first, we are shared |
| `no-store` | never store | response not cached |
| `no-cache` | may store, must revalidate each use | response not cached (simplified) |
| `private` | only the browser may store | response not cached |
| `public` | may be stored by shared caches | not special-cased |
| `must-revalidate` | once stale, must revalidate | effectively always true here |
| `stale-while-revalidate` | serve stale while refreshing | not implemented |
| `stale-if-error` | serve stale if origin fails | not implemented |
| `immutable` | don't revalidate while fresh | not special-cased |

### Shared vs private cache

This was one of the most important lessons:

- A **browser** is a *private* cache. It uses `max-age`.
- A **proxy / CDN** is a *shared* cache. It uses `s-maxage`, falling back to
  `max-age`.

So:

```
Cache-Control: max-age=10, s-maxage=60
```

means:

- browser copy is fresh for 10s
- proxy copy is fresh for 60s

That is perfectly legal, and it is one reason a browser can send a conditional
request while the proxy still has a fresh copy.

---

## 6. Freshness lifetime

```ts
function freshnessLifetimeMs(headers: Headers): number {
    // no-store / no-cache -> 0
    // s-maxage -> max-age -> Expires -> default 60s
}
```

Precedence:

```
lifetime =
    s-maxage
    ?? max-age
    ?? (Expires - now)
    ?? 60 seconds (default TTL)
```

Storage math:

```ts
const remaining = lifetime - initialAge * 1000;
entry.expiresAt = Date.now() + Math.max(0, remaining);
```

Why subtract `initialAge`? Because the response may have already spent time in a
CDN before reaching us. We only get the *remaining* freshness, not the full
lifetime.

Example:

```
Age: 100
Cache-Control: max-age=300
```

We do **not** keep it for 300 more seconds. We keep it for `300 - 100 = 200`
seconds, because 100 seconds of its life are already gone.

Freshness check:

```ts
fresh = Date.now() < entry.expiresAt;
```

That is all `fresh` means: the proxy's copy is still inside its lifetime.

### Note on `Expires`

`Expires` is an absolute date. The spec computes lifetime as
`Expires - Date`. This implementation uses `Expires - Date.now()` and then
subtracts the initial age, which is close but not exact when the upstream `Date`
is old. It is a known simplification.

---

## 7. `Age`: why `now - Date`?

`Date` is **when the response was generated** — an absolute timestamp:

```
Date: Sat, 18 Oct 2026 12:00:00 GMT
```

`Age` is **how old it is** — a duration in seconds:

```
Age: 100
```

To turn the timestamp into a duration:

```
age = now - Date
```

If it is now `12:00:02` and `Date` is `12:00:00`, the response is 2 seconds old.
That is the whole reason for the subtraction. `Date` is the birth time; `now -
Date` is the elapsed time since birth.

Caches must preserve the origin's `Date`, so `now - Date` includes time spent in
the CDN, in transit, and in every cache along the way.

### Two estimates, take the max

```ts
const upstreamAge = parseInt(headers.get("age") ?? "0");
const apparentAge = max(0, floor((Date.now() - Date.parse(headers.get("date"))) / 1000));

initialAge = max(upstreamAge, apparentAge);
```

Why `max` and not sum?

- `Age` is what upstream caches *claim*.
- `now - Date` is what the clock *observes*.

They estimate the **same quantity** (current age), so adding them double-counts.
Taking the max is conservative: under-estimating age could make us serve stale
content; over-estimating only causes an early revalidation.

Worked example:

```
Date: 12:00:00
now:  12:00:02
Age:  100
max-age: 300
```

```
upstreamAge  = 100
apparentAge  = 2
initialAge   = max(100, 2) = 100
remaining    = 300 - 100 = 200s
```

Without `Age`, the `Date` fallback still catches old responses.

### Current age when serving

```ts
ageSeconds(entry) =
    entry.initialAge +
    floor((Date.now() - entry.storedAt) / 1000);
```

So `Age` = age when we received it + time it has been sitting in our cache.

We strip the upstream `Age` header and send our computed value instead. On a
revalidation `304`, `initialAge` is recomputed from the `304` headers and the
clock restarts.

---

## 8. Stale entries and revalidation

A stale entry is **not deleted**. It still has a body and validators, so we ask
the origin whether it is still valid.

```ts
const etag = entry.headers.get("etag");
const lastModified = entry.headers.get("last-modified");

if (etag) {
    headers.set("if-none-match", etag);
} else if (lastModified) {
    headers.set("if-modified-since", lastModified);
}
```

- `ETag` → `If-None-Match` (stronger, preferred)
- `Last-Modified` → `If-Modified-Since`
- neither → full refetch, no conditional

### What `304` means

`304 Not Modified` means:

> "The copy you already have is still valid."

It does **not** mean "the response is fresh". It is a *validation* result, not a
freshness result. After a `304` we must update the stored metadata and recompute
freshness from the new `Cache-Control` / `Age` / `Date`.

`refreshEntry()` does that, but deliberately keeps the old body and skips headers
that would be wrong to copy from a `304`:

```ts
if (lower === "age") continue;
if (lower === "content-length") continue;
if (lower === "content-encoding") continue;
if (lower === "transfer-encoding") continue;
if (lower === "set-cookie") continue;
```

Then:

```ts
entry.storedAt = new Date(now);
entry.initialAge = parseInitialAge(notModifiedHeaders);
entry.expiresAt = now + max(0, lifetime - initialAge * 1000);
```

### Outcomes

| Origin response | What we do |
| --- | --- |
| `304` | refresh metadata, reuse stored body |
| `200` cacheable | replace entry with new body |
| `200` uncacheable | delete stale entry, serve response once |
| no validators | full `200` refetch |

---

## 9. The two conditional conversations (the confusing part)

There are two independent layers:

```
client  ── If-None-Match ──►  proxy  ── If-None-Match ──►  origin
        ◄── 304 / 200  ──            ◄── 304 / 200  ──
```

1. **Client ↔ proxy** — `clientCacheMatches()`
2. **Proxy ↔ origin** — `revalidate()`

They are not the same check, and one does not imply the other.

### Four cases on a fresh proxy entry

| Proxy entry | Client sent validator? | Client validator matches? | Response |
| --- | --- | --- | --- |
| fresh | no | — | `200` + body |
| fresh | yes | yes | `304`, no body |
| fresh | yes | no | `200` + body |
| stale | any | — | revalidate origin first |

A fresh entry does **not** automatically produce a `304`. A normal browser
navigation usually has no `If-None-Match`, so a `HIT` still sends `200` + body.
`304` is always a reply to a conditional request, never an announcement that our
copy is fresh.

### Why can a client send a conditional while we are still fresh?

Because the browser and proxy have separate copies and separate clocks:

1. **`s-maxage` vs `max-age`** — proxy stays fresh longer than the browser.
2. **Forced revalidation** — reload, DevTools "disable cache", or a request with
   `Cache-Control: no-cache` / `max-age=0`.
3. **Another client refreshed the shared entry** — proxy got a `304` and extended
   its expiry; this browser's copy still has the old expiry.
4. **Age handling, heuristics, eviction, clock skew** — the copies drift.

Timeline for case 1:

```
Cache-Control: max-age=10, s-maxage=60

t=0    proxy stores until t=60
       browser stores until t=10

t=11   browser copy stale → sends If-None-Match
       proxy copy still fresh → answers 304 from memory
       origin is never contacted
```

Timeline for case 3:

```
t=0    proxy stores, expires t=60
t=61   browser B triggers revalidation → 304 → proxy expires t=121
t=62   browser A (old copy, expired at t=60) asks
       proxy is fresh → 304 from cache
```

This is why the `fresh` check must happen **before** looking at the client's
validators. If we assumed "conditional request ⇒ stale", we would hit the origin
even when our copy is perfectly usable.

---

## 10. What the client actually receives

Every cacheable-method response gets:

```ts
out.set("age", String(age));
out.set("x-cache", cacheStatus);
out.set("server", SERVER_NAME);
```

- `Age` — recomputed, not forwarded
- `x-cache` — `HIT` or `MISS`
- `server` — always `caching-proxy` (Cloudflare sends its own `server` header)

### `x-cache` values

| Situation | `x-cache` |
| --- | --- |
| fresh hit | `HIT` |
| stale entry revalidated with `304` | `HIT` |
| stale entry replaced with `200` | `MISS` |
| no entry, full fetch | `MISS` |
| non-`GET`/`HEAD` | `MISS` |

Revalidated `304` is labeled `HIT` because the body comes from our cache, even
though we contacted the origin.

### Forwarded headers

`Cache-Control` is forwarded **verbatim**. We only read it, never rewrite it.
`Age` is the one caching header we replace.

`3xx`/`4xx`/`5xx` responses are forwarded but not stored.

---

## 11. Header sanitizing

Proxies must not blindly forward every header. Two categories matter:

### Hop-by-hop headers

These describe a single connection, not the message:

```
connection, keep-alive, proxy-authenticate, proxy-authorization,
te, trailer, transfer-encoding, upgrade
```

Also, any header named in the `Connection` header must be dropped.

### Recalculated headers

| Header | Why we handle it |
| --- | --- |
| `host` | `fetch` sets it from the target URL |
| `content-length` | `fetch` recomputes it for the request body |
| `content-encoding` | `fetch` decompresses the body, so the header would lie |
| `transfer-encoding` | transfer framing is per-connection |
| `age` | we compute a fresh value |
| `set-cookie` | must be preserved individually, not merged |

`Set-Cookie` is special because multiple values must not be joined with commas.
`headers.getSetCookie()` gives the individual values, and `appendHeaders()` adds
each one separately.

`upstreamRequestHeaders()` also drops the client's validators on a cache miss, so
we fetch a full body that can be stored. It strips:

```
if-none-match, if-modified-since, if-match,
if-unmodified-since, if-range
```

---

## 12. `HEAD` and non-`GET` methods

### HEAD

- Cache key normalizes `HEAD` → `GET`.
- On a miss, fetch as `GET` so the body can be cached.
- Send only headers (`response.end(undefined)`).
- `Content-Length` is not recomputed by us; the origin's headers are used.

### Other methods

`POST`, `PUT`, `PATCH`, `DELETE`, etc.:

- request body is buffered with `prepareBody()`
- forwarded as-is
- response is forwarded
- never stored
- `x-cache: MISS`

One current detail: this branch does not set `Age`. It sets `x-cache` and the
`server` header (via `clientResponseHeaders`), but `Age` only appears on
`GET`/`HEAD` responses.

---

## 13. Clearing the cache: why an HTTP endpoint?

The cache is:

```ts
export const cache = new Map<string, CacheEntry>();
```

That `Map` lives **inside the running proxy process**.

Running `caching-proxy clear-cache` starts a **different process** with different
memory. It cannot reach into the server's `Map`. So the command has to ask the
running server to clear itself.

Options considered:

| Approach | How | Tradeoff |
| --- | --- | --- |
| HTTP endpoint | `DELETE /__cache` | simple, testable, returns a count |
| Signal + pidfile | server writes pid, CLI sends `SIGUSR2` | no reply, stale pids |
| Unix socket | CLI connects to a socket | OS-specific, more code |
| Redis / disk | shared external store | real-world, but big change |

Chosen: **HTTP endpoint**.

```ts
// src/proxy/clear-cache.ts
export const CLEAR_CACHE_PATH = "/__cache";

export function handleClearCacheRequest(request, response): boolean {
    const pathname = (request.url ?? "/").split("?")[0]?.replace(/\/+$/, "") || "/";
    if (pathname !== CLEAR_CACHE_PATH) return false;

    const cleared = cache.size;
    cache.clear();

    response.writeHead(200, {
        "content-type": "application/json",
        "cache-control": "no-store",
    });
    response.end(JSON.stringify({ cleared }));
    return true;
}
```

The server checks this **before** proxying:

```ts
if (handleClearCacheRequest(request, response)) return;
```

The CLI sends `DELETE http://127.0.0.1:<port>/__cache` and prints the count.

Tradeoff: the endpoint has no auth and no loopback restriction. Anyone who can
reach the proxy port can clear its cache. That is fine for a local learning tool,
but worth tightening if it is ever exposed.

`clear-cache` requires `-p` because the CLI cannot know which port the server is
on. `serve` defaults to `3000`, but the clear command has no safe default.

---

## 14. CLI design

Built with `yargs`.

```
caching-proxy                          # command list
caching-proxy --help                   # command list + global flags
caching-proxy serve -o <origin> [-p]   # start the server
caching-proxy clear-cache -p <port>    # clear a running server
```

### Commands, not flags

`serve` and `clear-cache` are yargs subcommands. Each lives in `commands/` and
exports a `CommandModule` with:

- `command`
- `describe`
- `builder` (options, validation, usage)
- `handler`

`index.ts` only wires them:

```ts
await yargs(rawArgs)
    .scriptName("caching-proxy")
    .usage("$0 <command> [options]")
    .command(serveCommand)
    .command(clearCacheCommand)
    .demandCommand(1, "Choose a command: serve or clear-cache")
    .strict()
    .help()
    .alias("help", "h")
    .version()
    .parse();
```

### No-args behavior

Bare `caching-proxy` prints the command list and exits `0`, like `bun` does. It
does not start a server. `-h` / `--help` shows yargs' full help with flags.

### Required options

`serve` requires `--origin`; `clear-cache` requires `--port`:

```ts
.option("origin", { demandOption: true, type: "string", ... })
```

Validation is separate (`checkOrigin`, `checkPort`), so `--origin not-a-url` and
`--port 99999` produce useful errors instead of starting a broken server.

### The usage-string gotcha

`demandOption: true` makes an option required at parse time, but yargs does **not**
add it to the generated command usage. So `caching-proxy clear-cache --help`
initially showed `caching-proxy clear-cache`, which looked like port was optional.

Fix: set an explicit usage string in the command builder:

```ts
.usage(`$0 clear-cache -p <port>\n\n${DESCRIPTION}`)
```

`$0` expands to `caching-proxy`. The description is included manually because a
custom `.usage()` replaces the default usage text.

### Global command

- `package.json`:

  ```json
  "bin": { "caching-proxy": "./index.ts" }
  ```

- shebang in `index.ts`:

  ```ts
  #!/usr/bin/env bun
  ```

- `chmod +x index.ts`
- `bun link`

Bun creates:

```
~/.cache/.bun/bin/caching-proxy -> <repo>/index.ts
```

That directory is already on `PATH` on this machine (`bun pm bin -g`). The symlink
points at the repo, so edits are live. If the repo moves, re-run `bun link`.

---

## 15. Code map

```
index.ts                       yargs wiring, no-args help
commands/
  serve.ts                     serve command + options
  clear-cache.ts               clear-cache command + HTTP client
  shared.ts                    DEFAULT_PORT, validators, no-args help
src/
  types.ts                     CacheEntry, CacheHit, RevalidationResult
  proxy/
    server.ts                  request orchestration, responses
    cache.ts                   freshness, Age, store, get, revalidate
    headers.ts                 request/response header sanitizing
    body.ts                    buffer non-GET/HEAD request bodies
    clear-cache.ts             DELETE /__cache handler
```

### `CacheEntry`

```ts
export type CacheEntry = {
    status: number;
    headers: Headers;      // sanitized client-facing headers
    body: Uint8Array;
    storedAt: Date;
    initialAge: number;    // age when we stored it
    expiresAt: number;     // Date.now() based
};
```

Stored headers are the **sanitized client-facing** headers, not the raw upstream
ones. That way a `HIT` replays exactly what a `MISS` would have sent.

---

## 16. How I tested it

I wrote a stub origin that could produce different caching behaviors:

| Endpoint | Response |
| --- | --- |
| `/store` | `max-age=1`, `ETag`, `304` on match |
| `/nostore` | `Cache-Control: no-store` |
| `/cdn` | `max-age=300`, `Age: 100` |

Then hit the proxy with `curl` and inspected headers:

```bash
curl -s -D - -o /dev/null http://localhost:3000/store
```

Verified sequence:

```
1 miss          200  x-cache=MISS  age=0
2 hit           200  x-cache=HIT   age=0
3 after 1.3s    200  x-cache=HIT   age=0   (upstream 304, body reused)
4 conditional   304  x-cache=HIT   age=0   (client If-None-Match matched)
5 no-store #1   200  x-cache=MISS  cc=no-store
6 no-store #2   200  x-cache=MISS  cc=no-store  (never stored)
7 cdn Age=100   200  x-cache=MISS  age=100
8 cdn again     200  x-cache=HIT   age=100
```

Clear-cache flow:

```
1st request     x-cache: MISS
2nd request     x-cache: HIT
clear -p 4211   Cleared 1 entry.
3rd request     x-cache: MISS
```

Also checked:

- `GET /__cache` does not get proxied
- missing `-p` errors
- `--origin not-a-url` errors
- `--port 99999` errors
- non-loopback / token tests were removed when the endpoint was simplified

`tsc --noEmit` is clean.

---

## 17. Questions I had while building

### Why `now - Date`?

`Date` is a timestamp; age is a duration. Subtracting the birth time from now
gives elapsed time.

### Why `max(Age, now - Date)` and not the sum?

They estimate the same thing. Adding double-counts. The max is the conservative
choice.

### Is `fresh` the same as `no-cache`?

No. `fresh` is time-based (`max-age` / `s-maxage` / `Expires`). `no-cache` means
"revalidate before every use". We currently don't store `no-cache` responses, so
they never reach the freshness check.

### If the browser sends a conditional, is the proxy always stale?

No. The browser and proxy are separate caches with separate clocks. The browser's
copy can expire while the proxy's is still fresh, especially with `s-maxage` >
`max-age`, forced reloads, or another client refreshing the shared entry.

### Does a fresh entry mean `304` to the client?

No. `304` is a response to the client's conditional request. A plain `GET` gets
`200` + body even on a fresh `HIT`.

### Can `clear-cache` just delete the `Map`?

No. It runs in a different process. It has to call the server's HTTP endpoint.

### Why override `server`?

Upstreams (Cloudflare, for example) send their own `server` header. The proxy
should identify itself.

### Why strip `content-encoding`?

`fetch` decompresses the body. Forwarding the original `content-encoding` would
claim the body is still compressed.

---

## 18. Known limitations / next steps

Things this project deliberately does not do yet:

1. **`Vary`** — cache key ignores `Vary`, so responses that differ by
   `Accept-Language`, `Accept-Encoding`, etc. can be served incorrectly.
2. **Eviction / size limits** — `cache` grows forever.
3. **Request coalescing** — two simultaneous misses both hit the origin
   (thundering herd).
4. **`stale-while-revalidate` / `stale-if-error`** — not implemented.
5. **Request `Cache-Control`** — a request with `no-cache` or `max-age=0` is not
   forced to revalidate; the proxy only looks at its own entry.
6. **`Authorization` / `Cookie` safety** — a shared cache should not reuse
   authenticated responses unless allowed by `public` / `s-maxage` /
   `must-revalidate`. Not handled.
7. **`Set-Cookie` caching** — responses with `Set-Cookie` can be stored and
   replayed to other clients. Should probably be treated as private.
8. **`206 Partial Content`** — `2xx` includes `206`, but range semantics are not
   handled, so partial responses should not be cached under the full key.
9. **`Expires` math** — uses `Expires - now` instead of `Expires - Date`.
10. **Naive `Cache-Control` parser** — splits on commas, so quoted values
    containing commas would break.
11. **No compression handling** — relies on `fetch` to decompress.
12. **Origin base path** — `new URL(origin).origin` drops any path
    (`https://x.com/api` becomes `https://x.com`).
13. **In-memory only** — cache is lost on restart; no persistence.
14. **No purge by URL** — `clear-cache` wipes everything.
15. **Admin endpoint has no auth** — anyone who can reach the port can clear.

Good next exercises: `Vary`, eviction, request coalescing, per-key purge,
`stale-while-revalidate`, and a proper `Cache-Control` parser.

---

## 19. Glossary

| Term | Meaning |
| --- | --- |
| **Origin** | The server that owns the data |
| **CDN / upstream** | An intermediary cache closer to the client |
| **Private cache** | A cache for one user (browser) |
| **Shared cache** | A cache for many users (proxy, CDN) |
| **Fresh** | Still inside its freshness lifetime |
| **Stale** | Past its freshness lifetime |
| **TTL** | Time to live / freshness lifetime |
| **Validator** | `ETag` or `Last-Modified`, used for revalidation |
| **Conditional request** | Request with `If-None-Match` / `If-Modified-Since` |
| **Revalidation** | Asking the origin whether a stored copy is still valid |
| **`304`** | "Your stored copy is still valid" |
| **`Age`** | How many seconds old the response is |
| **`Date`** | When the response was generated |
| **Hop-by-hop header** | Header for one connection only |
| **`Vary`** | Response depends on these request headers |
| **Coalescing** | Merging concurrent identical requests into one |
| **`x-cache`** | This project's `HIT` / `MISS` marker |

---

## 20. Cheat sheet

```bash
# install + global command
bun install
bun link

# run
caching-proxy serve -o https://dummyjson.com
caching-proxy serve -o https://dummyjson.com -p 8080

# clear
caching-proxy clear-cache -p 3000

# inspect
curl -s -D - -o /dev/null http://localhost:3000/users?limit=1
curl -s -D - -o /dev/null http://localhost:3000/users?limit=1

# clear directly
curl -X DELETE http://127.0.0.1:3000/__cache
```

Freshness in one line:

```
fresh = now < storedAt + (lifetime - initialAge)
```

Age in one line:

```
Age = max(upstream Age, now - Date) + time spent in our cache
```

Cache decision in one line:

```
fresh  → answer locally (304 only if the client asked conditionally)
stale  → revalidate origin, then answer
```
