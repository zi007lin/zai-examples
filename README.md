# zai-examples

Sample specs for [ZAI](https://zai.htu.io/app), the spec validator used in the [ZiLin Methodology](https://htu.io) at HTU.

This repo demonstrates how the ZAI rubric distinguishes complete specs from incomplete ones. The pair below is a real FEAT spec at two stages — one with a missing rubric item, one with it corrected — published twice, with two different filename conventions, to show both upload paths.

## The pairs

### Canonical-name pair (recommended for new authors)

The canonical ZiLin filename pattern is `YYYY-MM-DD__<type>__<title>(-vN)?.md`. ZAI reads the type from the filename directly — no inference needed.

| File | Score | What it shows |
|---|---|---|
| [`2026-05-14__feat__pre-trade-compliance-broken-v1.md`](./2026-05-14__feat__pre-trade-compliance-broken-v1.md) | **9 / 10 PARTIAL** | A FEAT spec missing the `### Who benefits` subsection under Game Theory |
| [`2026-05-14__feat__pre-trade-compliance-fixed-v1.md`](./2026-05-14__feat__pre-trade-compliance-fixed-v1.md) | **10 / 10 PASS** | The same spec with the missing subsection restored |

### Non-canonical-name pair (H1-fallback path)

The same content under non-canonical filenames. ZAI's detector falls back to the document's `# FEAT:` H1 to infer the type. Useful for legacy exports, brainstorm drafts, and any document whose filename predates the canonical naming convention. This path was added in [`zi007lin/zai` PR #90](https://github.com/zi007lin/zai/pull/90); without it, these files would have hard-thrown on upload.

| File | Score | What it shows |
|---|---|---|
| [`example-feat-broken.md`](./example-feat-broken.md) | **9 / 10 PARTIAL** | Same spec, non-canonical filename, type inferred from H1 |
| [`example-feat-fixed.md`](./example-feat-fixed.md) | **10 / 10 PASS** | Same spec, non-canonical filename, type inferred from H1 |

Both pairs describe the same feature: **a pre-trade compliance check API for derivative trades** — Python FastAPI service deployed to AWS via Terraform, with LLM-assisted review of ambiguous cases and a cryptographic audit hash chain.

The diff between broken and fixed is roughly 80 words inside one section. The rubric catches the absence regardless of which pair you upload.

## Try it yourself

1. Open [zai.htu.io/app](https://zai.htu.io/app)
2. Drop `2026-05-14__feat__pre-trade-compliance-broken-v1.md` into the scorer
3. See the score: **9/10 PARTIAL**, failure: `game_theory: missing required subsections: "Who benefits"`. The score panel's type badge reads `FEAT` with no provenance chip — the type came straight from the filename.
4. Drop `2026-05-14__feat__pre-trade-compliance-fixed-v1.md` into the same scorer
5. See the score: **10/10 PASS**
6. Repeat with the non-canonical pair (`example-feat-broken.md` / `example-feat-fixed.md`); same scores, but the type badge now carries an `inferred from H1` provenance chip.

Total time: under 60 seconds.

## Why this failure mode matters

Most engineers writing technical specs document **threats** — what could go wrong, what an attacker might do. Far fewer document **beneficiaries** — who is strictly better off after this system ships, and how. The ZAI rubric requires both subsections under Game Theory because beneficiary enumeration is a separate cognitive discipline from threat modeling.

Saying "we have a Game Theory section" is not the same as having one. The rubric makes the difference visible and unmissable.

The principle, in one line: *If you can't name the cooperators, you haven't designed for them.*

## Repository contents

```
2026-05-14__feat__pre-trade-compliance-broken-v1.md   # canonical-name pair, broken, 9/10
2026-05-14__feat__pre-trade-compliance-fixed-v1.md    # canonical-name pair, fixed,  10/10
example-feat-broken.md                                 # non-canonical pair, broken, 9/10 via H1 fallback
example-feat-fixed.md                                  # non-canonical pair, fixed,  10/10 via H1 fallback
README.md                                              # This file
```

## About ZAI

ZAI (Zero Ambiguity Intelligence) is the structural validation layer in the ZiLin Methodology — a spec-driven AI development pattern used to ship governed, auditable software. Specs are scored against per-type rubrics (FEAT, BUG, SPEC, CHORE, REFACTOR, UX, BRAND, EPIC); only PASS specs trigger the downstream autonomous implementation pipeline.

- Live scorer: [zai.htu.io/app](https://zai.htu.io/app)
- Methodology overview: [htu.io](https://htu.io)
- Brand: HTU (High Tech United)

## License

Apache-2.0. Sample specs in this repo are documentation, not implementation; the spec subject (pre-trade compliance check API) is a worked example, not a production system. Adapt freely.

## Maintainer

This repo is maintained alongside the [zai.htu.io](https://zai.htu.io) tool. For questions about the rubric or methodology, see [htu.io](https://htu.io).
