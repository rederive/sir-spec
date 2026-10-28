# SIR — the Semantic Intermediate Representation

The specification for **SIR verified-recompose**: decompile a code unit into a language-neutral behavioral
contract (a SIR + a frozen, held-out oracle), reconstruct it blind, and accept only what independent rebuilds
prove on inputs they never saw.

**The normative document is [`SIR_SCHEMA.md`](SIR_SCHEMA.md)** — currently **v0.5** (v0.1 frozen 2026-06-10;
every change since is a versioned, evidence-cited addition, never a silent edit). A narrative introduction with
worked examples lives at **[rederive.ai/spec.html](https://rederive.ai/spec.html)**.

## The invariants (the short version)

- The oracle is the substrate: a SIR artifact is self-contained — spec + frozen differential vectors. Prose is
  an unverified prior, never load-bearing.
- Expecteds are differential — stamped by executing the original — never hand-authored.
- Verification is earned by the round-trip gate, original deleted: N independent blind re-emissions must agree
  on the full held-out set.
- The verifier must be immovable from the seat of the party being verified.
- v0.5: a unit is one language-neutral **primary** plus **one language file per target language** (`js`, `ts`,
  `rust` today), and every language is graded against the same oracle in the `sir-canonical-v1` encoding.

## Implementations

| artifact | what | where |
|---|---|---|
| `rdv` | the trust-nothing verifier (`check` / `resynth`) | [npm `rederive`](https://www.npmjs.com/package/rederive) · [source](https://github.com/rederive/rederive) |
| `sir-factory` | the build orchestrator (install→decompose→stamp→re-emit→grade→pack) | [npm](https://www.npmjs.com/package/sir-factory) · [source](https://github.com/rederive/sir-factory) |
| `sir` plugin | the Claude Code skills & agents (`sir-verify`, `preflight`, role agents) | [source](https://github.com/rederive/sir-toolkit) |

## Installing the verifier

```sh
npm i -g rederive@^0.5        # provides the `rdv` command
npx -p rederive rdv check <bundle-dir>   # one-off, without a global install
```

v0.5 bundles (`specVersion: "0.5"`) need **rederive 0.5.0 or later**; an older `rdv` halts on them. Note that the
npm package named `rdv` is an unrelated third party: always install or run `rdv` from the `rederive` package.

## Versioning & stewardship

Changes are additive and versioned (`specVersion` in every bundle manifest; `rdv` asserts compatibility).
Each addition must cite the specimen that forced it — nothing is resolved from taste. The file format and
specification will never be put behind a license, fee, or patent assertion against conforming implementations
(see the [rederive Promise](https://rederive.ai/promise.html)).

Licensed Apache-2.0.
