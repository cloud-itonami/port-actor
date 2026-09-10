# Operator quickstart — port-actor

Every step below was actually executed on 2026-09-06 (macOS, `clojure`, `nbb`,
`node`/`npx`, `jq`, `curl`, `dig`) from the repo root, and the output shown is
the real output. If a step does not reproduce, the repo has drifted — do not
skip it silently.

Two steps below are expected to be **red**: step 2 fails 2 of 26 assertions,
and step 5 shows two identities that disagree. Both are the repo's current
state, not a mistake in this document. They are described in the README under
*Known drift* and *Identity*.

## Prerequisites

- `clojure` (the boundary tests and lint run on the JVM)
- `nbb` (ClojureScript-on-Node — the workspace script host)
- `node` / `npx` (vitest is fetched on demand by `npx --yes`)
- `jq`, `curl`, `dig`

No install step: this repo has no `package.json`; vitest runs standalone
against the single test file.

## 1. Inspect the manifest

```bash
jq -r '{id: .["@id"], name, nanoid, runtime,
        capabilities: (.capabilities|length),
        pipelines: (.pipelines|length),
        actors: (.actors|length),
        triggers: [.pipelines[].trigger.type] | group_by(.)
                  | map({(.[0]): length}) | add}' actor-manifest.jsonld
```

Real output:

```json
{
  "id": "did:web:port.etzhayyim.com",
  "name": "port",
  "nanoid": "p0rt7890",
  "runtime": "k8s-langserver",
  "capabilities": 5,
  "pipelines": 13,
  "actors": 7,
  "triggers": {
    "cron": 2,
    "subscribeRepos": 1,
    "xrpc": 10
  }
}
```

To see which pipeline is which:

```bash
jq -r '.pipelines[] | (.trigger.type + "  " +
       (.trigger.nsid // .trigger.cron // (.trigger.collections|join(","))))' \
  actor-manifest.jsonld | nl
```

Entries 12 and 13 (`cron 0 */6 * * *` and
`xrpc com.etzhayyim.apps.port.coverage.get`) are the coverage pair that step 2
does not yet know about.

## 2. Run the manifest invariants (vitest) — currently RED

```bash
npx --yes vitest run actor-manifest.test.ts
```

Real result:

```text
 Test Files  1 failed (1)
      Tests  2 failed | 24 passed (26)
```

The two failures are stale count literals, nothing else:

```text
FAIL  Port Actor Manifest > has 11 pipelines
AssertionError: expected [ { …(2) }, …(12) ] to have a length of 11 but got 13

FAIL  Port Actor Manifest > xrpc pipelines > has 9 xrpc pipelines
AssertionError: expected [ { steps: [ { …(3) } ], …(1) }, …(9) ] to have a length of 9 but got 10
```

`actor-manifest.test.ts:77` pins 11 total pipelines and `:131` pins 9 xrpc
pipelines. The manifest has 13 and 10. The remaining 24 assertions — canonical
`@id`, `name`/`nanoid`, `runtime`, MCP-primitives-only capabilities, the cron
and `subscribeRepos` pipeline shapes, the 3-step occupancy pipeline, and the
7-facet `actors` list — pass.

## 3. Run the boundary contract tests

```bash
clojure -M:test
```

Real output:

```text
Running tests in #{"test"}

Testing port.murakumo-test

Ran 9 tests containing 291 assertions.
0 failures, 0 errors.
```

These tests introspect `cell-specs`, so the assertion count tracks the number
of cells; they do not hardcode cell names and so do not drift the way step 2
did.

```bash
clojure -M:lint
```

Real output ends with `errors: 0, warnings: 1` — the warning is an unused
`input` binding at `src/port/murakumo.kotoba:201`. The alias uses
`--fail-level error`, so this exits 0. (The elapsed time clj-kondo prints on
the same line is machine- and load-dependent; do not treat it as an
invariant.)

## 4. Exercise the actor boundary — deny by default

The pure boundary lives in `src/port/murakumo.kotoba`. With **no attestations**,
a cell plan is `:blocked` and carries zero effects:

```bash
nbb --classpath src -e '
(require (quote [port.murakumo :as m]))
(let [blocked (m/cell-plan :health {:attestations {}})]
  (println "status:" (:status blocked))
  (println "missing gates:" (count (:missing-gates blocked)))
  (println "effects:" (count (:effects blocked))))'
```

Real output:

```text
status: :blocked
missing gates: 7
effects: 0
```

With all seven common gates attested, the same cell becomes `:ready` and plans
one `:mst/put-record` effect per collection:

```bash
nbb --classpath src -e '
(require (quote [port.murakumo :as m]))
(let [atts (into {} (map (fn [g] [g true]) m/common-gates))
      ready (m/cell-plan :health {:attestations atts
                                  :computed-at "2026-09-06T00:00:00Z"
                                  :request-id "qs-demo-1"})]
  (println "status:" (:status ready))
  (println "effects:" (mapv :op (:effects ready)))
  (println "collection:" (:collection (first (:records ready))))
  (println "rkey:" (:rkey (first (:records ready)))))'
```

Real output:

```text
status: :ready
effects: [:mst/put-record]
collection: com.etzhayyim.port.health
rkey: qs-demo-1
```

Note the collection namespace: `com.etzhayyim.port.health`, **not**
`com.etzhayyim.apps.port.health`. See the README's *Identity* section.

The same in aggregate over all 21 cells — this is the whole-actor deny-by-default
demonstration:

```bash
nbb --classpath src -e '
(require (quote [port.murakumo :as m]))
(let [plans (m/all-cell-plans {})]
  (println "cells:" (count plans))
  (println "statuses:" (frequencies (map :status (vals plans))))
  (println "total effects:" (reduce + (map (fn [p] (count (:effects p))) (vals plans)))))'
```

Real output:

```text
cells: 21
statuses: {:blocked 21}
total effects: 0
```

Attesting all seven gates flips every cell at once:

```text
statuses: {:ready 21}
total effects: 21
```

`safe-rkey` is what keeps a record key addressable — it strips the `did:web:`
prefix, replaces anything outside `[A-Za-z0-9._~-]` with `-`, and never
returns a blank:

```bash
nbb --classpath src -e '
(require (quote [port.murakumo :as m]))
(println (m/safe-rkey "did:web:port.etzhayyim.com"))
(println (m/safe-rkey ""))
(println (m/safe-rkey "a/b c"))'
```

Real output:

```text
port.etzhayyim.com
unknown
a-b-c
```

## 5. Resolve the identities — they disagree

The DID stamped into every planned record is the manifest's `@id`. Ask DNS
whether it can resolve at all:

```bash
dig +short port.etzhayyim.com
curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 \
  https://port.etzhayyim.com/.well-known/did.json
```

Real output: `dig` prints **nothing** (no A/CNAME record), and curl reports
`000` — it never connected. `did:web:port.etzhayyim.com` does not resolve.

The DID in `.well-known/did.json` is a different one, and it does resolve:

```bash
curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 \
  https://etzhayyim.com/actor/port/did.json
```

Real output: `200`.

But the served document is not the committed one:

```bash
curl -s --max-time 12 https://etzhayyim.com/actor/port/did.json -o /tmp/served-did.json
diff <(jq -S . /tmp/served-did.json) <(jq -S . .well-known/did.json)
```

The diff is real and non-empty: the served copy uses the `jws-2020` context
suite instead of `ed25519-2020`, has `alsoKnownAs: []`, points its PDS at
`https://pds.aozora.app` instead of `https://pds.etzhayyim.com`, replaces the
`#aozora` AozoraAppView service with an `#xrpc-libp2p` AtprotoXrpc service, and
adds a `_meta` block (`status: r0`, `kind: tier-b`,
`primaryLexicon: com.etzhayyim.port`).

Neither GitHub Pages candidate serves the committed file:

```bash
for u in https://etzhayyim.github.io/com-etzhayyim-port/.well-known/did.json \
         https://cloud-itonami.github.io/port-actor/.well-known/did.json; do
  curl -s -o /dev/null -w "%{http_code} $u\n" --max-time 12 "$u"
done
```

Real output: `404` for both.

## 6. What you can NOT do from this repo

- **Not run the actor.** There is no runtime, no k8s-langserver, and no graph
  connection here. The `:mst/put-record` effects in step 4 are inert values.
- **Not query any port data.** The manifest's SQL lives in pipeline
  definitions; no port, berth, terminal, or port-call row exists in this repo.
- **Not settle the identity question.** Step 5 measures the disagreement; it
  does not resolve it. Changing either DID is a governance decision, and the
  records planned in step 4 carry whichever DID `port.murakumo/actor-did`
  holds.
