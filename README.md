# drt-http-api-lib

The wire, for a DRT guest that serves HTTP: how a request message becomes a
request table, the one response envelope every answer goes out in, and the
three functions that say what the edge observed about the caller.

Extracted from discofetch `api/supervisor.lua` -- the "helpers" banner and the
request table at the top of `serve_forever`. The arithmetic, the branch order
and the refusal behaviour are unchanged; only the globals moved.

## The surface

Pure. No host, no queue, no clock, so no constructor:

    M.split_query(target)      -> path, query
    M.request_from(msg)        -> the normalised request table
    M.int_param(v)             -> a whole number, or nil
    M.observed_address(req)    -> the caller's address as the edge saw it
    M.observed_port(req)       -> the source port the edge terminated
    M.public_forwarded(chain)  -> the relay chain minus our own topology

Configurable values:

    M.AUTH_CHALLENGE           -> { ['www-authenticate'] = 'Bearer realm="discofetch"' }
    M.DEFAULT_CONTENT_TYPE     -> 'application/json'

Bound. These reach the response queue, so they come from `new()`:

    M.new(deps) -> instance
    instance:reply(conn, status, payload, headers, content_type)
    instance:fail(conn, status, code, message, headers)
    instance:no_body(conn, status)

## The injected deps

    M.new({
      out_queue = <the response queue handle from queue.declare>,
      push      = <function push(queue, message)>,   -- queue.push
      json      = <table with an encode function>,
    })

All three are required and each is asserted at `new()` in a sentence that
names the dep. There is no clock here and no db; if a future method wants
either, a boundary has been crossed.

`push` is injected rather than called because a module that reaches for a
global queue cannot be handed a second one -- and because the test suite is
then a table that records what it was pushed, with no host in the room.

## Usage

    local http = require('http_api')            -- see "Consumption", below
    local api  = http.new({ out_queue = http_out, push = queue.push, json = json })

    local path, query = http.split_query(m.path or '/')   -- or, all of it:
    local req = http.request_from(m)

    api:reply(m.conn, 200, { ok = true })
    api:reply(m.conn, 200, '[]')                          -- verbatim; see below
    api:reply(m.conn, 200, body, nil, 'text/plain')
    api:fail(m.conn, 401, 'unauthenticated', 'no verified identity on this request',
             http.AUTH_CHALLENGE)
    api:no_body(m.conn, 204)

## The two facts that only existed as comments

**A string payload is passed through verbatim, and that is the feature.**
`json.encode` cannot express an empty ARRAY. The encoder decides a table's
shape from its keys, an empty table has none, so it emits `{}` -- and there is
no `json.as_array` to say otherwise (msgpack has one, json does not). A client
that checks `.length` on `{}` breaks, so an empty list is spelled out as text.
Do not "improve" `reply` into an always-encode.

**An off-allowlist header name is dropped whole, and silently.** `headers` may
carry only names the deployment's `response_headers` allowlist grants. A name
that is not on it produces no error and no partial header; the response simply
goes out without it. There are TWO host files (`api/api.host.lua` and
`api/api.dev.host.lua`) and a name added to one works in dev and vanishes in
production. This library does not filter -- the host already does -- so the
obligation is written where someone adding a header will read it. The names
this library's own output depends on are `www-authenticate` (AUTH_CHALLENGE)
and, for callers, `retry-after` and `cache-control`.

Related: `content_type` is a FIELD on the message rather than a header name,
because the allowlist refuses `content-type` by name along with the other
framing headers.

## Dependency edges

**None.** This library is the floor: it depends on nothing else in the set and
must stay that way. Who depends on it:

- `discofetch-fetchpoint-lib` -- reply, fail, observed_address, observed_port,
  public_forwarded
- `discofetch-accounts-lib` -- int_param
- the supervisor composition root -- all of it, including `request_from` at the
  top of the serve loop and `AUTH_CHALLENGE` at the two 401 sites

Things a reader may expect here that live elsewhere, by ruling, and must not be
copied back in:

- `valid_ip` -> `discofetch-model-lib`. Nothing in this library calls it;
  `public_forwarded` does its own inline classification, which is what keeps
  this library edge-free.
- `rate_headers`, the `rate_limited` code and the retry-after convention ->
  `token-rate-limit-lib`. They are functions of the limiter's output, not of
  the envelope.

## What was deliberately left out

- **No header-name filtering.** See above: the host drops off-list names, and a
  second copy of the allowlist in here is a copy that drifts from the two that
  matter.
- **No key derivation for rate limiters.** `observed_address` falls back to the
  first `x-forwarded-for` element, which is right where the answer is
  informational and WRONG where it is enforcement: a limiter keyed on a header
  the caller can seed is a limiter the caller configures. The waitlist and
  redeem limiters read `req.real_ip` only, at their own call sites, on purpose.
  Do not tidy those into a helper here.
- **No routing.** The route table, the method-not-allowed list and the public
  -route predicate stay in the composition root, which is the only place that
  knows what this program serves.
- **No `now`.** Nothing here is time-dependent.

## Known, carried over

Behaviour that looks like a bug, was left exactly as it was, and has a test
pinning the current answer:

- `int_param` is `tonumber` plus an integrality check, so it accepts forms no
  query-string documentation mentions: `0x10` parses as 16, `1e3` as 1000, and
  surrounding whitespace is tolerated. Callers range-check but do not
  re-validate the spelling.
- `observed_port` has the same `tonumber` reach: `0x1bb` is accepted as 443.
- `public_forwarded` does not classify a v4-mapped v6 address, so
  `::ffff:10.0.0.1` is echoed as public. The v4 test needs a leading
  `<digits>.<digits>.` and the v6 test looks only at `::1`, `fc`/`fd` and
  `fe8`-`feb`.
- `public_forwarded`'s `b ~= nil` guards in the 172/12 and CGNAT branches are
  unreachable-as-written: the v4 match binds both captures or neither.
  Harmless, and removing them is a rewrite of a branch nobody asked to change.
- `split_query` percent-decodes the VALUE only; a key spelled `a%20b` stays
  encoded. It also takes last-wins on a repeated key, so `?a=1&a=2` yields `2`
  with no array form and no error.
- `split_query`'s decoder ignores malformed escapes (`%zz` is left alone)
  rather than refusing the request.

## Consumption

Nothing requires this yet. Guests in DRT have no `require` and no `dofile`
today; the load-time modules slice is designed but unshipped. The module is
real -- one file, last statement `return M` -- and the `require` line in
"Usage" above is what it becomes the day that ships. Until then the test
harness wraps the file in an IIFE and concatenates the cases, which is exactly
what `test/run.sh` does.

## Tests

    sh test/run.sh          # DRT=/path/to/drt to point at another binary

Green means the last line is exactly `PASS`. 110 cases: the query split and its
decoding, the integer parser's refusals, both observation functions including
the absent-is-nil rules, every branch of the private/CGNAT/loopback/link-local
classification, the request table's absent-vs-nil and host-normalisation
behaviour, each named `new()` refusal, and the envelope -- string passthrough
with the encoder proven uncalled, the default and overridden content type,
header passthrough, the error envelope, the 204 shape, and two instances
writing only their own queues.

Two of those cases exist because the obvious spelling of them proves nothing,
and both are the kind that rot silently:

- **The decoder's ORDER is pinned, not just its output.** `+` becomes a space
  BEFORE `%xx` is expanded, so `%2B` survives as a literal `+`. Cases built
  only from `a+b` and `a%20b` pass under either order; the case that discriminates
  is `q=a%2Bb -> a+b`.
- **The "no port fallback" case uses a chain that would parse as a number.**
  `observed_port({ forwarded = '198.51.100.1' })` is nil whether or not a
  fallback exists, so it cannot fail; `forwarded = '8080'` can.
