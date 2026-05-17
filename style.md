# pok.sh Code Style

This is how we write Go in this repository. It is **Tiger Style** ([reference][tiger])
adapted for Go: we keep the spirit (safety, performance, developer experience —
in that order, with simplicity as the binding principle) and drop the Zig-specific
bits that do not translate (snake_case, manual memory, `comptime` assertions).

The first law is **simplicity**. A new contributor should be able to open any file
and understand what we are doing without a map. If a rule below ever fights that
goal, simplicity wins.

[tiger]: https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md

---

## 1. Naming

**Use full words. No abbreviations.** Reading code is more frequent than typing it.

```go
// bad
func handle(c net.Conn, r *server.Registry) { ... }
hdr := make([]byte, 4)
sess, err := yamux.Client(conn, nil)

// good
func handleConnection(connection net.Conn, registry *server.Registry) { ... }
header := make([]byte, 4)
session, err := yamux.Client(connection, nil)
```

**Exceptions:**

- Loop indices (`i`, `j`) are fine when the variable is purely an arithmetic index.
  Prefer `for index := range slice` or `for range count` when possible.
- Go-idiomatic receivers are short and unchanged: `func (r *Registry) Get(...)`.
  This is the one place where Go convention beats the Tiger Style rule.
- Standard-library style stays: `io.Reader`, `io.Writer`, `ctx context.Context`,
  `err error`.

**Put units and qualifiers last, in descending significance.**

```go
const MaxPayloadBytes = 16 << 20     // good — type ("max") then unit ("bytes")
const latencyMillisecondsP99 = ...   // good — what, unit, qualifier

const BytesMaxPayload = ...          // bad — units belong at the end
```

**Pick nouns over adjectives** so the name reads cleanly in prose:
`registry.tunnels` beats `registry.active`. The former composes
(`registry.tunnels_max`); the latter does not.

**Group related names with matching lengths** so columns line up:
`source`/`target` over `src`/`dst`; `request`/`response` over `req`/`resp`.

---

## 2. Functions

**Hard cap: 70 lines of function body.** No exceptions. There is a sharp
discontinuity between a function fitting on a screen and not.

**Function shape is an inverse hourglass:** small parameter list, simple return
type, meaty middle. Triple-return functions like `(*X, string, string)` are a
smell — prefer named returns or a small typed error.

**Centralize control flow.** Branching (`if`, `switch`, retry loops) lives in the
parent function. Helpers are leaf functions that compute one thing and return.
This is what "push `if`s up and `for`s down" means: callers branch, callees compute.

**Order in a file:** top-level constants first, then types, then `init` if any,
then the most important exported function, then the rest in order of use.
`main` is always at the top of a `main` package file.

---

## 3. Control flow

**State invariants positively.** Match the happy path first:

```go
// good
if length < lengthMax {
    // happy path
} else {
    return ErrTooLong
}

// bad — invariant phrased as a negation
if !(length >= lengthMax) { ... }
```

**Use modern range syntax.**

```go
for i := 0; i < 100; i++ { ... }   // bad — Go 1.22 made this obsolete
for range 100 { ... }              // good — when index is unused
for index := range 100 { ... }     // good — when index matters
for index := range slice { ... }   // good — when iterating a collection
```

**No recursion** in production paths. Loops with explicit bounds are easier to
reason about and impossible to blow the stack with.

**Put a limit on everything.** Every loop and every queue has a fixed upper
bound. Examples in this codebase: `MaxPayloadBytes`, `randomAttemptsMax`,
`subdomainLengthMax`, `randomCollisionRetries`. If a value can be unbounded in
principle, give it a bound in practice and document why.

---

## 4. Comments

Two voices, both right:

> "Don't forget to say why." — Tiger Style
>
> "Comments are not evil. They are as necessary to programming as basic
> branching or looping constructs… Inside your code should be explanations
> about what the code is supposed to be doing." — Cal Evans, *A Comment on
> Comments* ([97 Things Every Programmer Should Know][evans])

The synthesis: **write comments generously, in two flavors.**

[evans]: https://github.com/97-things/97-things-every-programmer-should-know/blob/master/en/thing_16/README.md

**Why-comments** explain a decision the code cannot. Why this constant, why
this bound, why this lock and not another. They prevent future contributors
from "fixing" something that was deliberate.

```go
// good — records a decision the code cannot
//
// MaxPayloadBytes is the largest payload a single frame may carry.
// Control-plane messages are tiny JSON blobs (hundreds of bytes), so 16 MiB
// is generous by orders of magnitude. The cap exists purely as a fail-fast
// bound against a malicious or buggy peer sending an absurd length prefix.
const MaxPayloadBytes = 16 << 20
```

**What-comments** explain dense or unidiomatic code. A single line of
`encoding/binary` or `crypto/rand` is opaque to anyone who has not used
those APIs recently. A one-line annotation costs nothing and saves a
documentation lookup.

```go
// good — translates one cryptic stdlib line into English
//
// Encode the payload length as a 4-byte big-endian prefix.
binary.BigEndian.PutUint32(header[:], uint32(payloadLength))
```

**The first audience is a Go developer who has never seen this codebase
before.** They should follow the file top-to-bottom without stopping to
puzzle out individual lines. The second audience is a future contributor
who knows the codebase but is staring at a four-month-old design choice
and wondering whether to keep it — that is where the why-comments earn
their keep.

**Lean toward more rather than less.** A reviewer will tell you when a
comment has crossed into stating-the-obvious territory; the opposite
direction — code dense enough that the next person has to reverse-engineer
it — is much harder to repair.

**Tight, not big.** Every comment pays for itself or it gets cut. A
one-line comment that adds real information beats a ten-line paragraph
that restates the same point three ways. If you find yourself writing
"because" twice in the same comment, the comment is padded.

```go
// bad — three sentences saying one thing
//
// We retry on collision rather than mutating the candidate, because a
// silent mutation could push the name outside the guaranteed-good range.
// randomAttemptsMax is generous: a collision rate high enough to exhaust
// 100 tries would indicate the wordlists are misconfigured at build time,
// not a runtime condition.
const randomAttemptsMax = 100

// good — same insight, one sentence
//
// Random-name retry cap. Exhausting 100 tries means the wordlists collide
// with the blocklists on every draw — a build-time bug, not a runtime one.
const randomAttemptsMax = 100
```

**The one case where a better name beats a comment.** If a block needs
`// loop n times`, the variable `n` is the bug — rename it `attemptsMax`
and the comment evaporates. But this is the exception, not the rule. When
in doubt, comment.

**Doc comments are full sentences** — capital letter, period at the end.
Start with the identifier name (`Validate enforces the user-facing
subdomain rules.`). This is Go convention; `go doc` and IDEs render them as
the public-facing description. The top doc-comment of a public function
should give a reader enough to call it correctly without scrolling into the
body.

**End-of-line comments** are phrases, no punctuation, kept tight:
`count := 0 // wraparound is fine`.

**The career-limiting move (Evans, ibid.):** do not paste angry emails or
attribute design decisions to specific colleagues in comments. Code outlives
the mood that produced it.

---

## 5. Errors

**Wrap with `%w` at every boundary** so callers can use `errors.Is` /
`errors.As`. Exported sentinel errors live next to the function that returns
them.

**Return errors; do not panic.** `panic` is for "this represents a bug in our
own code" (e.g. `subdomain.Random` exhausting attempts means the wordlists
are misconfigured at build time) or for `crypto/rand` failing (the OS is broken).

**Match positively on errors at the boundary:**

```go
if errors.Is(err, ErrSubdomainTooShort) { ... }
```

---

## 6. Formatting

- `gofmt` and `go vet` must be clean. CI will reject anything else.
- Group logically related lines, separate phases with a single blank line.
  A function should read as a sequence of paragraphs, each doing one thing.
- 100-column soft limit. If you are wrapping past that, the function is
  probably doing too much.

A canonical paragraph layout inside a function:

```go
func handleConnection(connection net.Conn, registry *Registry) {
    // Phase 1: setup
    defer connection.Close()
    address := connection.RemoteAddr().String()

    // Phase 2: yamux session
    session, err := yamux.Server(connection, nil)
    if err != nil {
        log.Printf("yamux server (%s): %v", address, err)
        return
    }
    defer session.Close()

    // Phase 3: control stream
    stream, err := session.AcceptStream()
    ...
}
```

---

## 7. Safety / assertions

Go has no `assert`. The equivalent is **boundary checking with explicit errors**
plus **panics for "cannot happen" cases**. The Tiger Style rules still apply:

- Validate every argument at the public boundary of a package.
- Pair checks: validate again at the next layer where the property matters
  (e.g. `Validate` rejects bad input; the registry's `Insert` still rejects
  duplicates).
- Assert the **negative space** in tests: feed every error path invalid input
  and confirm it produces the expected `errors.Is` match. Don't only test the
  happy path.

---

## 8. Dependencies

Match Tiger Style's "zero dependencies" attitude as much as Go lets us. The
standard library does more than you think. Before adding a module:

1. Could a few dozen lines of stdlib code do this?
2. Is the maintenance burden worth it for the API improvement?
3. Does the dep itself have a long transitive chain?

Current allowed direct dependencies are pinned in `go.mod`. New ones need a
note in the PR explaining why.

---

## 9. Performance: think before you measure

Tiger Style: "the best time to solve performance is in the design phase".
Back-of-the-envelope before code:

- Memory: how big does this allocation get under N requests/second?
- Network: how many bytes per request? Can we batch?
- CPU: is this in the hot path? If yes, is it allocation-free?
- Disk: do we touch it at all? (Currently no — we are an in-memory edge proxy.)

A 10x design improvement beats a 2x microoptimization every time. The hot path
in pok.sh is **public listener → tunnel session → client local server**.
Everything on that path stays allocation-light and lock-light. The control path
(rare: once per tunnel lifetime) can be liberal.

---

## 10. Tests

- Table-driven tests with `t.Run(name, ...)` subtests.
- Test the happy path AND every error path. Tiger Style's "negative space" rule.
- Use `errors.Is` to match expected errors, not string comparisons.
- Concurrency tests use `atomic.Int32` for counters; never call `t.Fatalf` from
  a goroutine (it calls `Goexit` on the wrong goroutine) — use `t.Errorf` and
  `return`.
- Bench what is in the hot path. Don't bench what is in the cold path.

---

## 11. The first law

If any of the above ever produces code that is harder to understand than the
"wrong" alternative, write the simpler version and update this file. Style
exists to help readers; when it stops helping, it stops applying.
