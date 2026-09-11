# port-actor

**World port infrastructure actor.** It carries the identity, the declarative
manifest, and the pure gate model for the `port` actor — a registry of major
world ports with their berths, terminals, and vessel port-calls, fed by
port-call events from the `vessel` actor.

This repo is **not** the runtime. It holds no server, no database connection,
and no scheduler. What lives here is the *plan*: which cells exist, which
attestations each cell demands, and what effects a fully attested cell is
allowed to emit. Executing those effects is the runtime's job.

## What is in this repo

| path | what it is |
|---|---|
| [`actor-manifest.jsonld`](actor-manifest.jsonld) | Declarative actor manifest: 5 capabilities, 13 pipelines (2 cron, 1 `subscribeRepos`, 10 xrpc), 7 actor facets (regions and cargo types) |
| [`actor-manifest.test.ts`](actor-manifest.test.ts) | vitest invariants over the manifest. **Currently 2 of 26 fail** — see *Known drift* below |
| [`src/port/murakumo.kotoba`](src/port/murakumo.kotoba) | Pure actor boundary. `cell-plan` checks the 7 required gates and returns `:blocked` (zero effects) or `:ready` with one `:mst/put-record` effect per collection |
| [`test/port/murakumo_test.kotoba`](test/port/murakumo_test.kotoba) | Contract tests for the boundary. They introspect `cell-specs` rather than hardcoding cell names, so they hold as the manifest changes |
| [`.well-known/did.json`](.well-known/did.json) | A DID document. **It is not the one currently served** — see *Identity* below |
| [`storage-profile.edn`](storage-profile.edn) | Repository storage profile declaration (`:kotoba/local-agent-kagi-chunks-v1`) |
| [`docs/operator-quickstart.md`](docs/operator-quickstart.md) | Runnable quickstart — every step in it was executed as written, and the output shown is the real output |

## Gate model (deny by default)

All 21 cells in `cell-specs` require the same seven gates:

```
:council-charter-attestation      :no-platform-held-key-baseline
:no-probing-baseline              :murakumo-only-inference-baseline
:did-primary-baseline             :append-only-gate-baseline
:kotoba-only-substrate-baseline
```

A plan with any missing attestation is `:blocked` and carries **zero**
effects. With no attestations at all, every one of the 21 cells is blocked and
the actor plans nothing:

```text
statuses: {:blocked 21}
total effects: 0
```

Only a fully attested plan is `:ready`, and then each cell yields exactly one
`:mst/put-record` effect. The quickstart demonstrates both directions.

## Identity — the two DIDs in this repo do not agree

Two different DIDs are asserted by two different files, and they are not
interchangeable. Measured 2026-09-06:

| DID | asserted by | resolves? |
|---|---|---|
| `did:web:port.etzhayyim.com` | `actor-manifest.jsonld` `@id`, and `port.murakumo/actor-did` — so this is the DID stamped into **every planned record** | **No.** `port.etzhayyim.com` has no DNS record; the fetch fails to connect |
| `did:web:etzhayyim.com:actor:port` | `.well-known/did.json` `id` | **Yes**, `https://etzhayyim.com/actor/port/did.json` → 200 |

Worse, the document served at that resolving URL is **not** the one committed
here. The served copy uses the `jws-2020` context suite (committed:
`ed25519-2020`), has an empty `alsoKnownAs`, points its PDS at
`https://pds.aozora.app` (committed: `https://pds.etzhayyim.com`), and carries
an `#xrpc-libp2p` service where the committed file has `#aozora`. Neither
GitHub Pages candidate serves the committed file (both 404).

Nothing here says which one is right — that is a decision, not a measurement.
What is measured is that a reader cannot infer the actor's identity from this
repo alone. The quickstart's step 5 reproduces all of it.

One nuance worth keeping: the *served* document's `_meta.primaryLexicon` is
`com.etzhayyim.port`, which agrees with the collection namespace the cljc
scaffold emits (`com.etzhayyim.port.health`, …) and **not** with the manifest,
whose pipelines all sit under `com.etzhayyim.apps.port.*`. The string
`com.etzhayyim.port.` appears zero times in `actor-manifest.jsonld`. So the
scaffold and the manifest name their collections differently, and the served
DID document sides with the scaffold.

## Known drift

`actor-manifest.test.ts` pins two pipeline counts as literals, and the
manifest has since grown past them:

| assertion | expects | actual |
|---|---|---|
| `actor-manifest.test.ts:77` — total pipelines | 11 | 13 |
| `actor-manifest.test.ts:131` — xrpc pipelines | 9 | 10 |

The two extra pipelines are the coverage pair appended after the invariants
were written: the `coverageSnapshot` cron and the
`com.etzhayyim.apps.port.coverage.get` xrpc. The other 24 assertions pass,
including the 7-facet `actors` count and the MCP-primitives-only capability
whitelist. This is drift in the invariants, not in the manifest; it is
recorded here rather than silently fixed, because updating the numbers is a
change to what the suite *checks*, and belongs to whoever decides that the
coverage pair is intended.

## Running it

```bash
kbb -M:test    # boundary contract tests — 9 tests, 291 assertions, 0 failures
kbb -M:lint    # clj-kondo — 0 errors, 1 warning
```

The full walk-through, including the manifest suite and the identity probes,
is [`docs/operator-quickstart.md`](docs/operator-quickstart.md).

## Provenance

Migrated from `etzhayyimcojp/20-actors` on 2026-05-21 (see `NOTICE`). The
`murakumo.cljc` boundary is a manifest-migration scaffold: each cell's
`:ceiling` says so, and every record it plans carries `:scaffold true` and
`:constitutionalStatus "attested-plan"`.

## License

Apache License 2.0 with the etzhayyim Charter Compliance Rider v3.1 — see
`NOTICE`.
