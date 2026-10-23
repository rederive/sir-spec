# SIR Schema — v0.1 FROZEN (2026-06-10) · v0.2 (2026-06-26) · v0.3 + v0.4 (2026-07-01) · v0.5 (2026-10-23)

> **Canonical home: [github.com/rederive/sir-spec](https://github.com/rederive/sir-spec).** This copy ships with
> the plugin and is kept in sync.
>
> v0.2 (§§11–14) adds `KIND STATE`, fidelity variants, decision-table fidelity, and a `specVersion`'d bundle.
> v0.3 (§§15–18) codifies four features already implemented in `sir-factory` + `rdv`: `ORACLE-CLASS`/`TRACE-SEAM`,
> export-shape `SEAM`s, the `ENVELOPE`, and carried-data authority attestation.
> v0.4 (§§19–22) is the **concurrency layer** — validated `PAR` semantics, the scheduler control effect + virtual
> time, the resource-lifecycle oracle channel, and the token `RING` — consolidated from the verified-normalization
> lineage in the **2026-07-01 unification** (see the note at §19; those sections were previously published in the
> site spec under the colliding numbers "v0.2/v0.3 (2026-06-22)" and are renumbered here — no shipped bundle pins
> that numbering). Earlier sections are unchanged. **Generation now targets v0.4.**
> v0.5 (§23 + §10/§12 extensions) splits a unit into a language-neutral **primary** and one **language file**
> per target language, freezes the practiced directive vocabulary, and adds the language-neutral vector encoding
> `sir-canonical-v1`. Executable form: `rederive/cli/sir/`.

---

## v0.1 — FROZEN (2026-06-10)

This resolves the six open grammar decisions of `SIR_HARNESS_ROADMAP.md` Part VI, plus three
additions forced by this week's evidence. **Generation targets this document.** Changes after
this point are versioned (v0.2, …), never silent. Each decision cites the specimen that
forced it — nothing here is resolved from taste alone.

Evidence base: dotenv-core (12 units, both round-trip tiers proven, coverage loop closed
IV.8) and editorconfig-core-js (held-out: firewall 19/19 cold, parse_string round-trip
25/25 + 18/18 ×2). Commits `397768c…95b4307`.

---

## 0. The invariants (unchanged, restated as frozen)

- One recursive tree of `KIND`-discriminated nodes; the discriminator **leads** — it is never
  inferred from shape (`feedback_discriminators_over_validators`).
- **The firewall:** a `FUNCTIONAL` subtree may not contain an `EFFECT`.
- **The oracle is the substrate.** A SIR artifact is self-contained = spec + frozen
  differential vectors. Prose (`INVARIANT`/`NOTES`) is an unverified prior — useful,
  optional, never load-bearing (proven: structural-only + coverage = 18/18 held-out, twice,
  two repos; and prose can be *wrong* — the `expand_variables.sir` author error).
- Expecteds are differential (run the original), never hand-authored.
- A unit's verification status is earned by the round-trip gate, original deleted.

## 1. Common header (every unit)

```
SIR     <name>                          # snake_case; a "name worth saying"
KIND    FUNCTIONAL | EFFECT | STATE     # STATE earns kindhood in v0.2 §11
SRC     <file>:<Class.>?<symbol>        # provenance; review diffs anchor here
SIG     (<params>) -> <result>
TRIGGER call | http(<route>) | cli | cron | <custom>     # OPTIONAL; default call. See §8.
REACH   entry | internal | off-path                       # See §6.
ORACLE  value | trace  @vectors: <path>  [VERIFIED <evidence>] | [not captured]
```

## 2. Decision 1 — LOOP accumulator: unit-level STATE only

`LOOP` gets **no first-class accumulator binding**. A unit that reads or writes an
accumulator declares it once, at unit level:

```
STATE  <name> (in) | (out) | (in/out)     # e.g. STATE map (in/out)   STATE cache (out)
```

*Evidence:* dotenv's `parse` fold (reads the in-progress map) and editorconfig's
`options.cache`/`options.files` accumulators both round-tripped from skeleton + vectors with
nothing more than a unit-level STATE marker — the **oracle carries fold semantics**
(18/18 ×2, both repos). A richer LOOP-accumulator slot would be prose-tier metadata; the
skeleton stays minimal. Caller-provided accumulators (a passed-in Map/array) are STATE, not
world effects — `processFileContents`'s `cache.set` stays FUNCTIONAL.

## 3. Decision 2 — emit/inject duality: EMIT may bind a result

```
EMIT    <boundary.op>(<args>) -> <name>    # outbound WITH meaningful reply: one node
EMIT    <boundary.op>(<args>)              # outbound, fire-and-forget
REQUEST <boundary.op>(<args>) -> <name>    # purely inbound (fs.read, env.read, clock, rand)
```

An outbound call with a reply (dotenv's `exec` → `spawn.sync`) is **one protocol step**: the
trace records the emission and the injected reply together. Splitting it into an EMIT+REQUEST
pair created artificial ordering ambiguity (which §5's canonical form would then have to
paper over). REQUEST survives for effects with no outbound payload.

## 4. Decision 3 — ON-ERROR is a BRANCH, not a kind

```
BRANCH on-error(<op>) { <arm> -> … }       # sugar; desugars to BRANCH with an error predicate
```

A kind must earn a **distinct verification obligation**; error arms verify exactly like
branch arms (arm coverage by captured inputs). *Evidence:* editorconfig's `getConfigSync`
try/catch (error → undefined contents → negative-cache path) is just an arm selected by an
injected outcome. Pure-tier error behavior (parse_string's error-TRUNCATION quirk) doesn't
even reach the skeleton — the vectors carry it (proven, round 1→2).

## 5. Decision 4 — traces are ordered; PAR is an explicit unordered window

- A trace is an **ordered sequence** of EMIT/REQUEST records. Canonicalization may normalize
  *representation* (key formatting, serialization) — **never ordering**.
- `PAR { … }` declares the only ordering exception: within a PAR block, the canonical form is
  the **sorted multiset** of the block's sub-traces. Verification is order-sensitive outside
  PAR windows, order-insensitive within.

*Evidence:* dotenv multi-file `load` order is load-bearing (later files override). **PAR is
frozen as design but UNVALIDATED — no specimen yet.** First concurrency specimen must test
it before PAR claims verification semantics (flagged, roadmap "harder effect cases").

## 6. Decision 5 — off-path units are kept, marked REACH

```
REACH entry      # on a public path from the module surface
REACH internal   # called only by other units
REACH off-path   # exported but uncalled on any module path
```

*Evidence:* dotenv's `is_ignored_line`, editorconfig's deprecated-but-exported
`parseFromFiles*` and "limited utility" `matcher`. Off-path units are part of the public
behavior surface — someone imports them; dropping them is **silent API loss in a migration**,
which is exactly the "unknowns" failure mode the harness exists to prevent. Cheap metadata,
no new kind, decompiler computes it from the export table + call graph.

## 7. Decision 6 — boundary descriptors are `<domain>.<verb>`, two levels

```
<domain> ∈ { fs, env, net, proc, db, time, rand, log, ipc, ui }      # closed set, versioned
<verb>   = operation class: read | write | stat | spawn | query | exec | fetch | set | …
```

- **Not per-API-function.** `fs.readFileSync` and `fs.readFile` are both `fs.read` — sync/async
  is calling-convention metadata, *proven irrelevant to the trace* (editorconfig's `getConfig`
  and `getConfigSync` produce identical trace shapes).
- Args stay on the node (`REQUEST fs.read(filepath)`), so finer-grained audit remains possible.
- The MODULE `SURFACE` line and the PR-review auditor diff operate at `domain.verb` level —
  a new `net.fetch` edge in a diff is exactly the red line a reviewer must see.
- Cap resolution maps a descriptor → a semcom cap when the catalog has one; otherwise it stays
  a typed boundary descriptor (the honest IV.4 state).

## 8. Forced addition 1 — TRIGGER is metadata, never structure

The spine rule ("an effect is an effect; the trigger is metadata") gets a frozen slot. A Next.js
route handler, a Nest controller method, a CLI subcommand, and a cron job with the same
choreography have the **same SIR body** and different `TRIGGER` lines. **This is the
migration invariant:** Next→Nest = re-emit with a rewritten TRIGGER, verified against the
same frozen trace oracle.

## 9. Forced addition 2 — opaque dependencies must be NAMED (the parse_to_uint32array lesson)

```
DEPENDS-ON <unit>, <unit>                      # in-module units
DEPENDS-ON OPAQUE(<module>.<symbol>), …        # external calls not on the known-pure allowlist
```

*Evidence:* editorconfig's `parse_buffer` semantics live in `@one-ini/wasm` — the v0 skeleton
never mentioned it, and the oracle silently carried the entire tokenizer (it worked, but a
reviewer couldn't see that the oracle was load-bearing there). Naming opaque deps makes the
firewall's blind spot visible.

**Tri-state firewall (resolves IV.2 as policy, not per-fixture tuning):**

| bucket | meaning |
|---|---|
| `EFFECT` | direct or transitive world effect detected |
| `FUNCTIONAL` | pure, all external calls on the known-pure allowlist (or none) |
| `FUNCTIONAL?` (suspect) | pure-shaped, but has OPAQUE deps not on the allowlist |

Unknown externals never silently pass as pure (the soundness fix) and never falsely claim
world effects (the noise fix). A suspect unit **with a passing value-oracle round-trip** is
behaviorally pinned even though purity is assumed — the badge upgrades trust, not kind.
The known-pure allowlist is a versioned artifact (starts: language builtins, `path.*` minus
cwd-dependence noted in GROUND_TRUTH strictness levels, `semver`, `minimatch`).

## 10. Forced addition 3 — frozen oracle-bundle contract

A **self-contained bundle** is:

```
bundle/
  sir/<unit>.sir            # specs per this schema
  oracles/<cluster>.json    # { unit, captured_from, vectors: [{name, (round,)? input|args, expected}] }
```

- Vectors are frozen at capture; coverage rounds append `cov_*`-prefixed vectors (fresh
  inputs; a disjointness guard MUST refuse any held-out string).
- The re-emit gate consumes ONLY the bundle (original deleted).
- **Closure rule (the editorconfig lessons, frozen as protocol):** a coverage round is closed
  only when (a) the full held-out set is re-run (new vectors can regress passing seams), and
  (b) **N≥2 independent emissions agree** (a pass on an uncovered class is luck). Priors are
  outvoted by evidence density, not overridden by single contradictions.

---

## Deferred (named, not frozen)

- `PAR` verification semantics (no specimen — §5 flag).
- Transactions / compensation, multi-process effects.
- An ambient-read tier (`env.cwd`, locale, TZ) — currently a GROUND_TRUTH strictness note.
- The `ui` domain verbs (frontend T3 is a non-goal at this phase).

---

# SIR Schema v0.2 — additions (2026-06-26)

Builds on v0.1 (frozen, above; unchanged). **New evidence:** the STATE-mutation re-grade of the shipped
catalog. `lodash.pullat` (runner STOPPED rather than ship — surfaced a *vacuous* STATE differential),
`aws4` (an `extraHeadersToInclude` signed-vs-canonical desync), and `lodash.defaultsdeep` (an `assignMergeValue`
wrong-leaf-rule). The `grade.mjs` STATE-differential fix (`8c33f80`) and the 20-unit re-grade under it (18 sound,
**2 real defects caught that the old gate had passed**) force three additions plus a bundle-versioning slot.
Nothing here is resolved from taste — each cites the specimen that forced it.

## 11. Forced addition 4 — `KIND STATE` earns kindhood: the `{result, post}` observable

```
KIND  STATE                                  # mutates a caller-provided argument in place
```

A kind must earn a **distinct verification obligation** (§4's rule). `STATE` does: a unit that mutates a
caller-provided argument in place is observed as **`{result, post}`** — the return value AND the post-call
state of the mutated argument — and its differential MUST **isolate arguments per call** (a fresh structural
clone per invocation).

*Evidence:* `lodash.pullat`. The v1 differential handed **one** decoded arg object to both the original and the
re-emission in sequence: the original mutated it and returned a reference, the re-emission then operated on the
**already-mutated** arg and returned the same reference → `r === w` **vacuously**, so a do-nothing mutator scored
200/200. (`decode()` passes live values *by reference*, so re-decoding does not un-alias — `structuredClone` does.)
Two follow-on defects had shipped VERIFIED under that vacuum (`aws4`, `defaultsdeep`); the sound gate caught both
on first run.

- **Mutate-and-return units** (`result === post` — the returned ref *is* the mutated arg): the held-out already
  pins the mutation through the return value; the fix is the per-call isolation in the differential.
- **Return≠post units** (`pullat` returns *removed elements*, a different object): **both** `result` and `post`
  are load-bearing and must be observed; value-mode that watches only the return is unsound for these → such a
  unit is quarantined unless the oracle observes `{result, post}`.

This **supersedes** v0.1 §2's "in-place mutation of a caller accumulator stays `FUNCTIONAL` + a `STATE` marker"
**for units whose *distinguishing* behavior is the mutation** — those are `KIND STATE`. The §2 unit-level `STATE
<name> (in/out)` marker remains for `FUNCTIONAL` units that merely thread a caller accumulator without their
identity depending on the mutation (the `processFileContents` case).

## 12. Forced addition 5 — fidelity variants: package-anchored vs spec-anchored

A unit may carry **≥1 fidelity variant**, each an independently verified artifact anchored to its own
**immovable** source — never a hand-written notion of "correct":

```
VARIANT package                              # default; anchored to the source package's actual bytes
VARIANT spec   ANCHOR <authority>            # anchored to an external authority (RFC vectors / reference impl / invariant)
```

- `package` (the implicit default for every unit to date): byte-faithful drop-in; the oracle is the source
  package run as today.
- `spec`: oracle anchored to an external authority — published RFC/test vectors, a reference implementation, or a
  spec **invariant**. Verified against THAT, not the package.
- The **gap** between two variants is the executed differential = the auto-generated **fidelity report**.

*Evidence:* `aws4`. Real aws4 forces a header into the `SignedHeaders` *list* via `extraHeadersToInclude` but omits
it from the canonical-headers *block* (the two builders use different filters) → a desynced signature. The
`package` variant reproduces that bug-for-bug (the byte-compatible drop-in); a `spec` variant anchored to the
**SigV4 consistency invariant** (`SignedHeaders names ≡ canonical-block names`) is the "correct" build. Both are
*earned*, neither hand-declared.

**Scope gate (when the feature switches on):** a `spec` variant exists ONLY where a second authority exists —
crypto, TLS, SigV4, encodings, HTTP semantics. For ad-hoc utilities (`lodash.pull`, `set-value`, …) the package
*is* the only truth; there is only ever a `package` variant and the feature stays silent. The on/off switch is
purely: does a spec oracle exist?

## 13. Forced addition 6 — decision-table fidelity: verify every cell against the original

The leaf / decision rule MUST be verified **cell-by-cell against the original package**, never modeled from the
prose hint or a plausible mental model. The decomposer runs the full Cartesian table (e.g. `dest` value-class ×
`src` value-class, including nested and array-index positions) against the original and encodes the **true**
observable per cell.

*Evidence:* `lodash.defaultsdeep`. The v0 SIR asserted *"a srcValue of `undefined` performs NO fill — no-op."*
Real lodash uses `assignMergeValue` — assign iff `(value !== undefined && !eq(dest[key], value)) || (value ===
undefined && !(key in dest))` — so `defaultsDeep({}, {a: undefined}) → {a: undefined}` **creates** the key. The
artifact faithfully implemented a *wrong SIR*, and the vacuous STATE differential masked it. Prose models
(`INVARIANT`/`NOTES`) remain unverified priors (§0); the decision table is **differential**, not prose.

## 14. Bundle contract v0.2 — `specVersion` + `variants`

Extends §10. The manifest records the spec a bundle was built against, and units may carry variants:

```
manifest.specVersion = "0.2"                 # the SIR Schema version this bundle targets; rdv asserts compatibility
units[].variants = {                          # OPTIONAL; absence ⇒ implicit single { package } variant (back-compat)
  package: { src, anchor: {kind:"package", ref}, verified },
  spec:    { src, anchor: {kind:"spec",    ref}, verified },
}
```

- **Back-compat:** a manifest with no `specVersion` is pre-0.2 (v0.1); a unit with no `variants` is an implicit
  single `package` variant. All 218 shipped units remain valid unchanged.
- `rdv check` verifies **each** variant against its own anchor and asserts `specVersion` is one it understands
  (the missing mechanical link between a shipped package and the spec it was built against).

---

## Deferred (v0.2)

- `spec`-variant *anchoring mechanism* for cross-language reference oracles (boto3 for `aws4`) — design only.
- `rdv add --fidelity <variant>` install-time selection — gated on the `rdv` installer existing.
- Promote `STATE` substructure (which arg is mutated) from oracle-carried to skeleton, IF a specimen forces it.

---

# SIR Schema v0.3 — additions (2026-07-01)

Builds on v0.1 + v0.2 (frozen, above; unchanged). **New evidence:** the three cold-agent UATs (`del` 2026-06-29,
`boxen` 2026-06-29, `query-string` 2026-06-30) and `@rederive/request`'s trace unit shipping in the OSS `rdv`.
Every addition here **codifies machinery already implemented** in `sir-factory` and `rdv` and exercised by shipped
bundles — v0.3 closes the gap between the practiced schema and the written one. Each cites the specimen that
forced it; nothing is resolved from taste.

## 15. Forced addition 7 — `ORACLE-CLASS` + `TRACE-SEAM`: an effect unit declares its injectable boundary

```
ORACLE-CLASS deterministic | trace | non-deterministic (<seam>)
TRACE-SEAM   <domain>                     # ONLY for trace-mode units: the injected boundary's domain (§7 set)
```

- `deterministic` → value-mode oracle (the default; pure function of its arguments).
- `trace` → the unit performs an EFFECT across an **injectable** boundary. It is stamped under a scripted fake
  boundary and its oracle records `{emitted, result}` — the ordered EMIT/REQUEST trace (§5 semantics) plus the
  final value. `TRACE-SEAM` names the domain so the harness knows which fake to build; the injected-boundary
  adapters live **in the verifier** (`rdv`), never in the package — a trust-nothing consumer must not run
  publisher-shipped harness code to check publisher code.
- `non-deterministic` with **no** injectable seam → **QUARANTINE**. The factory refuses to stamp a value-mode
  oracle for it; declaring `TRACE-SEAM` on a boundary that cannot actually be injected is a decomposer error.

*Evidence:* `@rederive/request`'s `httpRequest` (a `net` seam — ordered EMIT + injected response, record/replay
verified by the OSS CLI); chalk's `supports-color` ambient read (`env`/`tty`), injected as a collaborator in the
boxen run so the styling logic above the seam is value-verifiable.

## 16. Forced addition 8 — `SEAM`: export-shape adapters for non-callable exports

```
SEAM index | builder                      # OPTIONAL; absent ⇒ the export is invoked directly as a function
```

Value-mode grading drives the real export as a function. Real packages also ship two non-callable shapes:

- `index` — the export is a **pure data object** whose observable contract is property access (a consumer does
  `obj[key]`). The harness drives both sides as `key => obj[key]`.
- `builder` — the export is a **chainable constructor**: construct, walk a property/call chain, invoke. The
  harness drives both sides as `(opts, chain, input)`, where a string step is a property access and an
  `[name, ...args]` step is an access-then-call.

A seam adapter contains **zero domain knowledge** — it only navigates and invokes the *real* package, so it can
never rescue a wrong reconstruction. A non-callable export with no declared `SEAM` **quarantines** (the default
stays refuse-don't-guess). The reconstruction must export a plain function of the seam's calling convention; the
`SIG` line pins it.

*Evidence:* the boxen UAT — `cli-boxes` (a border-glyph table; property access *is* the contract) and `chalk`
(a default instance plus named `Chalk`; `new Chalk({level}).red.bold(x)`, `.hex('#f0f')(x)`). The gap was serious
enough that a cold agent, blocked by it, patched the toolkit to pass its own work — codifying the seam removes
the trigger; the immutability guardrail removes the capability.

## 17. Forced addition 9 — `ENVELOPE`: the verified input scope is part of the contract

```
ENVELOPE <one-line input scope>           # the input domain this unit is verified FOR
```

Verification is **evidence within an input distribution**, not a universal proof. When a unit is verified for a
subset of its potential domain, the envelope says so, and a consumer reusing it outside that scope knows they are
off the verified map. `pack` carries the envelope into the manifest and README; `rdv check` displays it. Absent
⇒ the unit claims its full documented signature domain. Together with §12's `FIDELITY`/`VARIANT` line, the
envelope makes a bundle state exactly *what was proven, against what anchor, over which inputs*.

*Evidence:* the del UAT (finding F5) — a glob engine verified against one project's pattern classes was correct
but **over-claiming** as a general drop-in; the envelope is the honest boundary. Same mechanism labels every
`spec`-fidelity divergence scope (the color-string percent correction).

## 18. Forced addition 10 — carried data: byte-exact extraction + an independent authority

```
CARRIED   <name> FROM <source-ref>        # extracted byte-exact by the TOOLCHAIN, never transcribed by a model
AUTHORITY { expr, equals }…               # re-runnable assertions anchoring the data to a PUBLISHED standard
```

Version-dependent tables (Unicode ranges, glyph charts, name→value maps) are **copied mechanically** at build
time (comment-safe const extraction; JSON sources parsed, not eval'd), content-hashed, and shipped as a data
module the reconstruction imports — a model never reproduces a 1,024-entry table from memory. Trust in carried
data comes from three independent legs:

1. the **oracle** exercises it (wrong data fails behavior);
2. the **content hash** pins it (any byte change is caught);
3. **authority assertions** check it against an *independent published standard* — UAX #11 for east-asian
   widths, the Unicode box-drawing chart for `cli-boxes` — not the source package's own bytes.

No assertions ⇒ **unattested** ⇒ the gate refuses by default. `rdv check` re-runs the authority assertions at
verify time, so a consumer re-verifies the attestation itself, not merely the hash.

*Evidence:* the boxen UAT — `get-east-asian-width`'s ranges and `cli-boxes`' glyph table (the terminal-stack
profile is carried-data-heavy); plus the extraction hardening the run forced (comments inside source literals,
pure-JSON sources).

## Bundle contract v0.3

`manifest.specVersion = "0.3"`. **Back-compat:** v0.2 bundles remain valid unchanged — every v0.3 field is
optional, and absence means the safe default (`ORACLE-CLASS deterministic`, direct-call export, full-signature
envelope, no carried data). `rdv` asserts `specVersion` compatibility per §14.

---

## Deferred (v0.3)

- `rdv normalize` (Mode 2, behavior-lock against a retained original) as a shipped one-command workflow — the
  method is validated on real modules; the command is not yet a product surface.
- A carried-data authority *registry* (named standards → canonical assertion sets), so common tables (UAX #11,
  box-drawing) attach attestations by reference.

---

# SIR Schema v0.4 — the concurrency layer (2026-07-01, the unification)

**Provenance note.** The four sections below were developed and validated on the **verified-normalization**
lineage (Mode 2: behavior-lock against a retained original) and previously published in the site spec under
version numbers ("v0.2" and "v0.3", 2026-06-22) that collided with §§11–18's numbering — the two documents
evolved on parallel workstreams. This unification renumbers them as **v0.4**; the artifact lineage (§14's
`specVersion`, which shipped bundles actually pin) is unchanged. Evidence base: `p-map`, `q`'s timer/microtask
core, a two-account transfer, `request`'s redirect state machine (256/256 generated + held-out scenarios), and
`ringbuf.js`. Nothing here is resolved from taste.

## 19. `PAR` verification semantics — VALIDATED (closes the §5 flag)

v0.1 §5 froze `PAR { … }` as design with an explicit UNVALIDATED flag. The flag closes:

- **Commuting sub-traces** (touch no shared state — order observationally irrelevant): the canonical form is the
  **sorted multiset** of sub-traces; verification collapses to one representative.
- **Dependent sub-traces** (a shared resource — order changes the observable): sorting is **unsound**. The unit is
  verified **over the space of interleavings** against the retained reference's reachable-**outcome set** plus
  invariants (safety, ordering, conservation), with the message-delivery order injected as a boundary the harness
  owns.
- **Coverage ladder**, reported honestly: `exhaustive` (small inputs — proven over all interleavings) →
  `sampled` (coverage-counted, when the space explodes) → `partial-order reduction` (one representative per
  equivalence class of independent reorderings — cheap exactly when the unit is correctly order-independent,
  thorough exactly when a refactor is not).

*Evidence:* `p-map`'s concurrency contract (interleaving-invariant output + a max-concurrency bound) verified
exhaustively for small inputs and POR-scaled past 10¹² interleavings; a two-account transfer whose outcome IS
interleaving-dependent; `request`'s redirect state machine.

## 20. The scheduler is a control effect — timer races verify on virtual time

A clock **read** (`REQUEST time.now`) is a point read, injected like any inbound value. **Scheduling** is a
different thing: timers (`time.timer` — setTimer/clearTimer) and the microtask scheduler are a **control
effect** that decides *when deferred work runs relative to everything else*. A timer race (does the timeout
fire before the awaited reply settles?) is nondeterministic on the wall clock, so it is verified on **virtual
time**: the event loop is injected as a scheduler the harness drives (`schedule` for microtasks;
`setTimer`/`clearTimer`/`advance` for timers), making resolution order a deterministic, replayable function of
the timeline.

*Evidence:* `q`'s `delay`/`timeout` and its microtask core.

## 21. The resource-lifecycle oracle channel

Cancellation is **invisible to the value/trace channels**: a leaked timer still fires, but its effect is a no-op
on already-settled state, so the emit/reply sequence is byte-identical while a real resource leak goes uncaught.
A unit that arms cancellable resources therefore carries a third oracle channel asserting the **lifecycle**:
`armed / fired / cancelled` for timers, `subscribed / torn-down` for handles — timers cleared, handles closed.

*Evidence:* `request`'s redirect timeout — a dropped `clearTimer` passes value + order and is caught **only** on
the lifecycle channel.

## 22. `RING` — token-coordinated concurrency reduces to a deterministic schedule

When a unit coordinates through a **single circulating permission** — only the token-holder acts — the
interleaving space collapses to a deterministic schedule (one outcome per token order; fixed-rotation is one
schedule, verified in O(1)). RING is the **design-time dual of partial-order reduction**: POR *detects*
independence to shrink the space; a ring *imposes* coordination to shrink it. A ring is a behavior
**refinement** — `R_ring ⊆ R_free` always (every ring run is a valid free run: it can only drop outcomes, never
invent one). The decomposer may **recognize** a ring (and verify `R_ring = R_free`, i.e. behavior-preserving) or
**impose** one as a deliberate, documented refinement during normalization.

*Evidence:* `ringbuf.js`.

## Bundle contract v0.4

`manifest.specVersion = "0.4"`. **Back-compat:** v0.2/v0.3 bundles remain valid unchanged — the v0.4 sections
only add semantics for concurrent/scheduled/cancellable units; absence means none apply. `rdv` asserts
`specVersion` compatibility per §14.

---

## Deferred (v0.4)

- Transactions / compensation and multi-process effects (carried forward from v0.1).
- The `ui` domain verbs (carried forward).
- Cross-language `spec`-variant anchors (carried forward from v0.2's deferred list). **Leaves the list in v0.5** (§12 extension).

---

# SIR Schema v0.5: primary + per-language files (2026-10-23)

v0.5 lets one unit be shipped in more than one language. A unit's behavior is stated once, in a language-neutral
**primary** file. What each target language needs to realize that behavior is stated separately, in one
**language file** per language. Every language is graded against the same oracle, which is encoded once in a
language-neutral form (§10 extension). v0.5 also freezes the vocabulary the factory actually writes (§23a),
which until now was practiced but never defined, and it un-defers cross-language anchors (§12 extension).

**Evidence base:** the v0.5 lab (2026-09-22 → 09-28). The re-cut `get_rgb` primary
(53 lines, rules R1–R9) plus one language file each converged cold emitters in JS, TypeScript and Rust,
cross-verified 3000/3000. In the Tier A and Tier B pressure tests, `bytes` and `slugify` were verified in all
three languages and `minimist` in JS and Rust. An independent Python implementation of the §10 encoding agreed
byte-for-byte with the JS one on both oracle sets it was run against (`get_rgb`, `is_number`).

**Back-compat:** every v0.2–v0.4 bundle remains valid unchanged. A lone `<unit>.sir` with no language file is
an implicit JS unit, exactly as before.

**Executable form.** The grammar, the vocabulary and the encoding below are implemented once, in
`rederive/cli/sir/`: `sir-parse.mjs` (one canonical parse and the canonical printer), `canon.mjs`
(`sir-canonical-v1`), `v05.mjs` (families, binding, the lints) and `validate.mjs`. Every check a validator makes
is a function over that one parse. Where this text and that code disagree, it is a bug in one of them, and
`validate.test.mjs` is where it gets a fixture.

## 23a. The practiced vocabulary, frozen

Practice has emitted directives this schema never defined. A census of the 226 shipped primaries found `UNIT`
228 times against the schema's own `SIR` 9, along with `FIDELITY`, `BEHAVIOR`, `DESCRIPTION`, `INVARIANT`,
`PROVENANCE`, `SCOPE`, `UNGUARDED-PRECONDITION` and `OUT-OF-DOMAIN`. Mechanical detection needs a defined
vocabulary, so v0.5 freezes it. The table is generated from the directive table in `sir-parse.mjs`
(`rederive/cli/sir/directive-table.mjs`), and a test in rederive fails if the two disagree. **File** says where a directive may appear (`E_REGISTER` otherwise). **Kind** says whether it is a
claim the oracle or a tool checks (*load-bearing*) or an unverified prior (*prose*, §0). A name not in the table
is an error: `E_UNKNOWN_DIRECTIVE`, or `E_PSEUDOCODE` for `LET` / `FOR` / `RETURN` / `STEP` / `NODE` and their
kin, or `E_EMITTER_META` for instructions to an emitter (`EMISSION`, `RE-EMITTER`, …), which never belong in
the artifact.

| directive | file | kind | meaning |
|---|---|---|---|
| `UNIT` | both | load-bearing | The unit name; the first line of every file (primary and language). Supersedes `SIR`. |
| `SIR` | primary | legacy (read only) | The v0.1 header. Read, never written. |
| `KIND` | primary | load-bearing | `FUNCTIONAL \| STATE \| EFFECT` (§1, §11). |
| `FIDELITY` | primary | load-bearing | `package \| spec` (§12). Supersedes the `VARIANT` spelling; the manifest `variants` map is unchanged. |
| `VARIANT` | primary | legacy (read only) | The §12 spelling of `FIDELITY`. Read, never written. |
| `ANCHOR` | primary | load-bearing | What the contract is anchored to (§12). Also an attribute (`… ANCHOR <ref>`). `ref(<lang>:…)` is the §12 extension. |
| `SRC` | primary | load-bearing | Source location of the original (§1). |
| `TRIGGER` | primary | load-bearing | Metadata, never structure (§8). |
| `REACH` | primary | load-bearing | An off-path unit, kept and marked (§6). |
| `ORACLE-CLASS` | primary | load-bearing | `deterministic \| trace` (§15). |
| `TRACE-SEAM` | primary | load-bearing | The injectable effect boundary (§15). |
| `SEAM` | primary | load-bearing | An export-shape adapter (§16). |
| `SIG` | both | load-bearing | Primary: the signature over the abstract types (§23b). Language file: the native signature. |
| `ENVELOPE` | both | load-bearing | The verified input scope (§17). In a language file: a narrowing (any language) or, for `js` only, a widening. |
| `DEPENDS-ON` | both | load-bearing | Named dependencies (§9), or `none`. Never carried data (`E_CARRIED_AS_DEPENDENCY`). |
| `DESCRIPTION` | both | prose (unverified prior) | Contract-level prose: what the unit is for. |
| `BEHAVIOR` | primary | load-bearing | Header for the numbered rules `R1 … Rn` (§23a rule 1). |
| `INVARIANT` | primary | prose (unverified prior) | A property the oracle should bear out. Prose until a vector pins it. |
| `TABLE` | primary | load-bearing | A decision table (§13 as a directive). Every row is verified by the oracle. |
| `PINNED` | both | load-bearing | `PINNED <kind> <name> = <literal>`: a hard requirement. Neutral in the primary, dialect in a language file. Cited (§23a rule 3). |
| `EARNED` | both | load-bearing | A rule or pin added after an observed divergence: `EARNED round-<n> heldout <vector>`. |
| `NOTE` | both | prose (unverified prior) | Non-load-bearing prose. Metered (`W_NOTE_DENSITY`). |
| `RATIONALE` | both | prose (unverified prior) | Why a rule is the way it is. Non-load-bearing. |
| `SCOPE` | primary | prose (unverified prior) | `IN:` / `OUT:` classification of behavior. |
| `UNGUARDED-PRECONDITION` | primary | load-bearing | An input condition the unit does not check; behavior outside it is unspecified. |
| `OUT-OF-DOMAIN` | primary | load-bearing | An input the contract explicitly does not cover, and what (if anything) is promised for it. |
| `ORACLE` | primary | load-bearing | `ORACLE value -> oracles/<unit>.json CANONICAL-SHA256 <hex>`: the oracle file and its declared hash (§10 extension). |
| `PROVENANCE` | both | prose (unverified prior) | Where the contract came from: package, version, function, history. |
| `CARRIED` | both | load-bearing | A carried constant table (§18). Primary: `CARRIED <name> FROM <source>` + `SHAPE` / `CANONICAL-SHA256` / `AUTHORITY`. Language file: where the module is. |
| `SHAPE` | primary | load-bearing | Sub-field of `CARRIED`: the table's abstract type and entry count. |
| `CANONICAL-SHA256` | primary | load-bearing | Sub-field of `CARRIED`: the table's canonical hash. (The oracle's hash rides on the `ORACLE` line.) |
| `AUTHORITY` | primary | load-bearing | Sub-field of `CARRIED`: one re-runnable assertion against the independent authority (§18). |
| `CARRIED-DATA` | primary | legacy (read only) | The v0.3 spelling of `CARRIED`. Read, never written. |
| `STATE` | primary | load-bearing | `STATE mutates <arg>`: the unit mutates an argument; the oracle observes `{result, post}` (§2, §11). |
| `EMIT` | primary | load-bearing | An effect emission that may bind a result (§3). |
| `REQUEST` | primary | load-bearing | An effect with no outbound payload (§3). |
| `BRANCH` | primary | load-bearing | A branch over a predicate (§4). |
| `ON-ERROR` | primary | load-bearing | Error handling as a branch (§4). |
| `PAR` | primary | load-bearing | An unordered window over sub-traces (§5, §19). |
| `RING` | primary | load-bearing | Token-coordinated concurrency (§22). |
| `LIFECYCLE` | primary | load-bearing | A cancellable resource (§21). |
| `EFFECT` | primary | load-bearing | An effect across a `<domain>.<verb>` boundary (§7). |
| `SURFACE` | primary | load-bearing | The module surface at `domain.verb` level (§7). |
| `NOTES` | primary | legacy (read only) | An ad-hoc legacy prose block. Read, never written; becomes `NOTE`. |
| `GUARD-PINNED` | primary | legacy (read only) | A legacy pinned guard. Read, never written; becomes `PINNED`. |
| `LANG` | language | load-bearing | Language file only: the language id, which must equal the filename's (§23b). |
| `PRIMARY-SHA256` | language | load-bearing | Language file only: the canonical-print hash of the primary it was written against (§23b). |
| `ABI` | language | load-bearing | Language file only: how abstract types map to native ones (text encoding, numeric semantics, crate or module shape). |
| `MECHANISM` | language | load-bearing | Language file only: which built-ins realize which rules, and anything an emitter in that language would otherwise get wrong. |
| `R<n>` | primary | load-bearing | One numbered rule under `BEHAVIOR`. |

**The grammar (strict mode, the v0.5 register):**

1. **Lines.** A line starting at column 0 is a directive: `NAME args [ATTR value]…`. Any indented line
   continues the directive above it, whatever the indentation (tabs, 2 or 4 spaces). The canonical printer
   emits one layout: the directive at column 0, continuations at two spaces, attributes trailing, no comments.
   **Every hash in a SIR binds to the canonical print**, so reformatting can never stale a binding.
2. **No `#` in the grammar.** A `#` outside a literal is a parse error (`E_COMMENT`). Prose has named, parsed,
   metered homes: `DESCRIPTION`, `NOTE`, `RATIONALE`. Citations are attributes: `… CITED <why>`,
   `… EARNED round-<n> heldout <vector>`, `… ANCHOR <ref>`. The census that forced this: 89% of all lines in
   the shipped corpus were `#` comments, including load-bearing pins with no slot of their own. **Legacy mode**
   still reads the shipped corpus and reports every `#` as a warning. Generation targets strict mode.
3. **`BEHAVIOR` is numbered rules.** `R1 … Rn`, each a statement about the abstract signature that a blind
   engineer could implement in more than one way only where the oracle leaves room. Pipeline or
   `STEP`/`BRANCH` narration of the original's control flow is rejected; the oracle pins the edges. Accidents
   of the original that the oracle pins are rules too, stated as behavior and cited (`CITED FIDELITY package`).
   A deterministic transcription lint measures code shapes in the primary (`typeof`, `=>`, `.match(`,
   `if (`, `return`, dialect regex literals, …). Its density is a warning, and an error under
   `--enforce-density`, the pack-time policy (gate 10%).
4. **Hard requirements get a typed, cited slot.** `TABLE` is a decision table (§13 promoted to a directive),
   and every row is verified by the oracle. `PINNED <kind> <name> = <literal>` holds a literal the behavior
   genuinely rides on. In the **primary** a pin must be language-neutral: a character set as code points, a
   byte string, a grammar. A dialect regex is a **language-file** pin: its accept-set is the contract, and the
   pattern is one rendering of it. **No pin without a citation** (`E_UNCITED_PIN`): every `PINNED` and
   `EARNED` line carries `CITED`, `EARNED` or `ANCHOR`. Literals are transcribed only on proven need.
5. **Legacy spellings** (`SIR`, `VARIANT`, `CARRIED-DATA`, `NOTES`, `GUARD-PINNED`, `UNIT: name` with a
   colon) are read, reported, and never written.

## 23b. PRIMARY + per-language files (forced addition 11)

**Files.** A unit's contract is a family:

```
sir/<unit>.sir            the PRIMARY: behavior, once, over abstract types
sir/<unit>.<lang>.sir     one LANGUAGE FILE per target language
sir/<unit>.carried.json   carried data, as §18 (unchanged)
```

`<lang>` comes from a **closed registry**: `js`, `ts`, `rust`, `php`, `csharp`, `python`. The registry is
versioned like §7's domain set and lives in the capability model (`rederive/cli/capabilities.json`,
`toolchain.languageRegistry`). A suffix outside it is a malformed bundle (`E_LANG_REGISTRY`), not a new
language. A lone `<unit>.sir` is an implicit `js` unit.

**The firewall, both ways.** The primary may not contain language mechanism. The language-leak lint rejects
dialect regexes, built-in names and host value names in a primary (`E_LANGUAGE_LEAK`). A language file may not
contain behavior: no `BEHAVIOR`, no rules, no `INVARIANT` (`E_REGISTER`). Before v0.5 there was nowhere to put
mechanism, so the shipped "neutral" exemplars carried JS regex flavor, JS `\S`, the JS line-terminator set and
`Math.min` in their bodies. The split gives that material a home and makes its absence from the primary
checkable.

**A language file contains only:**

| directive | content |
|---|---|
| `UNIT` | the unit name (must match the primary) |
| `LANG <id>` | the language; must equal the filename's |
| `PRIMARY-SHA256 <hex>` | the canonical-print hash of the primary it was written against |
| `SIG` | the native signature (e.g. `pub fn get_rgb(input: &str) -> Option<(f64, f64, f64, f64)>`) |
| `ABI` | how abstract types map to native ones: text encoding, numeric semantics, the artifact's module or crate shape |
| `MECHANISM` | which built-ins realize which rules; the regex dialect; anything an emitter in this language would otherwise get wrong |
| `ENVELOPE` | optional. A **narrowing**: what this language cannot represent (Rust: a lone surrogate). For `js` only, a **widening**: values beyond the abstract types the JS variant also handles, graded from a js-native extension oracle (§10 ext.) |
| `PINNED … CITED …` | dialect pins |
| `CARRIED <name> = <module>` | where this language's carried table lives |
| `DEPENDS-ON` | per-language named dependencies (§9), e.g. `OPAQUE(rederive.php.mul_u64)`, or `none` |
| `NOTE`, `PROVENANCE` | prose |

**Binding and staleness.** A language file's `PRIMARY-SHA256` is the SHA-256 of the primary's canonical print.
When the primary changes, every language file bound to the old print is **stale**: declared, not a claim
(`E_STALE_BINDING`; `E_UNBOUND` when the line is missing). A stale file is never graded. The toolchain stamps
the binding (`bind`). The author never computes it.

**The abstract type vocabulary** (closed and versioned, `canon.mjs` `TYPES`):

| type | meaning |
|---|---|
| `text` | a sequence of Unicode scalar values |
| `bytes` | a byte string |
| `bool` | true or false |
| `none` | the absent value; equals only itself |
| `i64` / `u64` | 64-bit integers, compared exactly. NaN, infinities and fractions can never occupy an integer slot |
| `f64` | an IEEE-754 double, compared on its bits (§10 ext.) |
| `json` | `none \| bool \| f64 \| text \| list[json] \| map[text, json]`. No `undefined`, functions, dates, boxed wrappers or regexes: those are envelope facts |
| `tuple[T, …]` `list[T]` `map[K, V]` `record[a: T, …]` | composites. A record's fields are compared in declared order |
| `T \| U` | a union. `T \| none` is the optional form |
| `...name: T` | a rest parameter (last only). The vector stays call-shaped; each language maps it natively |

A host value with no meaning in this vocabulary is not representable in the primary's `SIG`. It belongs in the
`js` file's envelope widening.

**Verification rungs** (a variant records the highest it earned; none is implied by another language's):
`type-checked` (the artifact satisfies its language file's `SIG` under the language's own checker: `tsc --strict`
with explicit `any` rejected, `cargo check`; JS cannot earn it) → `harness-verified` (one language, frozen +
held-out against the primary oracle, through a **verifier-owned** harness; adapters live in the verifier, never in
a publisher's bundle, as §15) → `cross-verified` (two or more languages agree on held-out *and* on a differential
against each other) → `oracle-anchored` (the primary oracle is `ANCHOR ref(<lang>:…)`-anchored, so agreement is
not self-consistency).

## 10. Extension (v0.5): the language-neutral vector encoding `sir-canonical-v1`

Mandatory for every oracle a `lang-variant` bundle carries; optional before that. Every value is tagged with its
abstract type, keyed by the primary's `SIG`, never by a native quirk:

| tag | carries | rule |
|---|---|---|
| `{t:'none'}` | — | equals only `none` |
| `{t:'bool', v}` | `true`/`false` | |
| `{t:'text', v, cu}` | the string | `cu:'utf8'` = Unicode scalar values. A lone surrogate has no utf8 form: the neutral encoder rejects it (an envelope fact). Only a js-native extension may hold it, tagged `cu:'utf16'` |
| `{t:'bytes', v}` | `base64:<…>` | |
| `{t:'i64'\|'u64', v}` | a decimal string | exact. Range-checked at encode time |
| `{t:'f64', v, bits}` | display decimal + 16 hex digits of IEEE-754 bits | **the bits are normative, `v` is a display hint** (shortest-roundtrip renderings differ across languages: Python `255.0`, JS `255`). **NaN rule:** every NaN, of any payload or sign, is canonicalized at encode time to `7ff8000000000000`, so NaN equals NaN. `-0` ≠ `+0`; `±Infinity` are distinct; no tolerance |
| `{t:'tuple'\|'list', v:[…]}` | elements | |
| `{t:'map', v:[[k, x], …]}` | entries | sorted by the canonical string of the key |
| `{t:'record', v:[[name, x], …]}` | fields | in declared order |
| `{t:'throw', msg}` | — | a throw is an **observation**, not a value. That it threw is the claim; `msg` is a display hint |

**The canonical string** is the compact JSON of the tagged value with keys sorted, with each `f64` reduced to
`{bits, t}` and each `throw` to `{t}`. Equality, everywhere, is equality of canonical strings. The oracle
file declares `encoding: "sir-canonical-v1"` and a `canonicalSha256` = SHA-256 of the canonical string of
`{frozen: [{args, expected}…], heldout: […]}`. The primary's `ORACLE` line carries the same hash
(`ORACLE value -> oracles/<unit>.json CANONICAL-SHA256 <hex>`). A grader verifies both the file's hash
(`E_ORACLE_FILE_SHA`) and the primary's declaration (`E_ORACLE_DECLARED_SHA`) before grading anything. A
reader fetches each hash **by directive**: a primary that carries a table has two canonical hashes, and "the
first `CANONICAL-SHA256` in the file" is the wrong one.

**Envelope extension.** Vectors whose values have no neutral meaning (a boxed `Number`, a `RegExp`, `undefined`,
a sparse array) are refused by the neutral encoder. A legacy oracle's such vectors move to
`oracles/<unit>.js.json` (`encoding: js-native-v0.4`, `envelope: js`), graded only by the `js` variant and
declared by the `.js.sir`'s `ENVELOPE` widening. That is how a JS-shaped unit coexists with the neutral form
without the form absorbing JS quirks. Legacy oracles are **re-encoded** losslessly, never regenerated
(`encode-oracle`): the same vectors, with the original kept as `<unit>.legacy.json`.

## 12. Extension (v0.5): cross-language anchors

`ANCHOR ref(<lang>:<reference>)`, either as a directive or as an attribute, anchors a fidelity variant to a
reference implementation in another language, e.g. `ANCHOR ref(python:botocore.auth.SigV4Auth)` for an `aws4`
unit. The reference is executed through that language's verifier-owned harness, and its outputs, encoded in
`sir-canonical-v1`, are the oracle. Agreement across languages is then anchored to an independent authority
rather than to each other. Detected mechanically as the `cross-lang-anchor` feature.

## Bundle contract v0.5

`manifest.specVersion = "0.5"`. A unit carries one variant per language in the existing `variants` map (§14),
keyed `lang:<id>`:

```
"variants": {
  "package":   { …the v0.4 fields… },
  "lang:rust": { "lang": "rust", "sir": "sir/<unit>.rust.sir", "langSha256": "…",
                 "primarySha256": "…", "oracle": "oracles/<unit>.json", "oracleSha256": "…",
                 "src": "src/<unit>.rs", "srcSha256": "…", "verified": { "rung": "cross-verified", … } }
}
```

`src` is optional: a variant without it is **spec-only**, shipped for the consumer to re-derive locally.
`verified {rung}` appears only where an artifact was actually verified. The features are `lang-variant`
(detected from a `<unit>.<lang>.sir` file or a `lang:<id>` key) and `cross-lang-anchor`. An rdv declares the
**languages it can harness** beside the features it implements, and halts naming the missing language. It reads
the bundle's shape itself (variant keys, per-language SIR paths), never only the bundle's declared `features`,
so an under-stamped bundle halts too. A factory never stamps a feature the verifier cannot check: it refuses to
pack instead.

**Status at this version:** the format, the vocabulary, the encoding and the validator are implemented. The
language capability model is in place, with `lang-variant` and `cross-lang-anchor` declared and **not yet
implemented** by a released rdv, which today harnesses `js` only. They flip to implemented per language as each
verifier-owned driver ships (TypeScript first, then Rust).

## Deferred (v0.5)

- Transactions / compensation and multi-process effects (carried forward from v0.1).
- The `ui` domain verbs (carried forward).
- Language-file inheritance (`EXTENDS`, e.g. `ts` from `js`). Language files are self-contained in v0.5.
- `CARRIED` sub-fields (`SHAPE`, `CANONICAL-SHA256`, `AUTHORITY`) as scoped AST children. Today they parse as
  continuations of `CARRIED`.
