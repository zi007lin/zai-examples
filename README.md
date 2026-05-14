# zai-examples

Sample specs for [ZAI](https://zai.htu.io/app), the spec validator used in the [ZiLin Methodology](https://htu.io) at HTU.

This repo demonstrates how the ZAI rubric distinguishes complete specs from incomplete ones. The pair below is a real FEAT spec at two stages: one with a missing rubric item, one with it corrected.

## The pair

| File | Score | What it shows |
|---|---|---|
| [`example-feat-broken.md`](./example-feat-broken.md) | **9 / 10 PARTIAL** | A FEAT spec missing the `### Who benefits` subsection under Game Theory |
| [`example-feat-fixed.md`](./example-feat-fixed.md) | **10 / 10 PASS** | The same spec with the missing subsection restored |

Both specs describe the same feature: **a pre-trade compliance check API for derivative trades** — Python FastAPI service deployed to AWS via Terraform, with LLM-assisted review of ambiguous cases and a cryptographic audit hash chain.

The diff between the two files is roughly 80 words inside one section. The rubric catches the absence.

## Try it yourself

1. Open [zai.htu.io/app](https://zai.htu.io/app)
2. Drop `example-feat-broken.md` into the scorer
3. See the score: **9/10 PARTIAL**, failure: `game_theory: missing required subsections: "Who benefits"`
4. Drop `example-feat-fixed.md` into the same scorer
5. See the score: **10/10 PASS**

Total time: under 60 seconds.

## Why this failure mode matters

Most engineers writing technical specs document **threats** — what could go wrong, what an attacker might do. Far fewer document **beneficiaries** — who is strictly better off after this system ships, and how. The ZAI rubric requires both subsections under Game Theory because beneficiary enumeration is a separate cognitive discipline from threat modeling.

Saying "we have a Game Theory section" is not the same as having one. The rubric makes the difference visible and unmissable.

The principle, in one line: *If you can't name the cooperators, you haven't designed for them.*

## Repository contents

```
example-feat-broken.md   # FEAT spec, scores 9/10
example-feat-fixed.md    # FEAT spec, scores 10/10 (the fix is roughly 80 words)
README.md                # This file
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
