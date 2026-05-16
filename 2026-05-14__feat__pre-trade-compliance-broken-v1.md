# FEAT: Pre-trade compliance check API for derivative trades

## Intent

Build a pre-trade compliance check API for derivative trades. Python FastAPI service deployed to AWS via Terraform. Accepts derivative trade payload (instrument, counterparty, notional, jurisdiction), runs structural compliance checks for Mifid II (pre-trade transparency, derivatives reporting fields) and Dodd-Frank (swap data repository requirements, mandatory clearing eligibility), uses an LLM (Anthropic Claude via API) to flag ambiguous edge cases for human review, and returns a structured compliance decision (PASS / FLAG / FAIL) with reasoning. Service is the audit-trail-of-record for pre-trade decisions; every call logged to S3 with cryptographic hash chain. Goal: investment-firm engineers can drop this into an existing RFQ pipeline as a pre-trade gate without rebuilding their compliance posture from scratch.

## Decision Tree

| Decision | Options | Chosen | Why |
|---|---|---|---|
| D1: Compute platform | Lambda + API Gateway / ECS Fargate / EKS / EC2 | ECS Fargate behind ALB | Steady traffic profile, persistent LLM connections |
| D2: Web framework | FastAPI / Flask / Starlette / Django | FastAPI | Type-checked Pydantic models, async-native |
| D3: LLM provider | Anthropic Claude / OpenAI / Bedrock / self-hosted | Anthropic Claude via API | Strong structured reasoning on regulatory text |
| D4: Secret storage | env vars / Secrets Manager / Parameter Store / Vault | AWS Secrets Manager | IAM-bound rotation, ECS integration |
| D5: Audit storage | DynamoDB / RDS / S3 with hash chain / SIEM | S3 with hash chain | WORM-style append, regulatory-friendly, tamper detection |
| D6: Decision semantics | Binary / Three-state / Confidence score | Three-state PASS/FLAG/FAIL | Surfaces ambiguous cases for human review |
| D7: Frontend | Bundled UI / API-only / Admin dashboard | API-only for v1 | Compliance teams use existing case-management tools |
| D8: Multi-region deployment | Single / Active-active / Active-passive | Single region us-east-1 | Latency budget allows; multi-region adds complexity |

### Trigger for change

Bump to v2 when: (a) LLM provider API materially changes; (b) jurisdiction beyond US/EU is required; (c) FLAG state proves insufficient; (d) regulator publishes new clearing eligibility rules.

## Final Spec

### Service surface

```
POST /v1/trade-decision
GET  /v1/healthz
GET  /v1/audit/{trade_id}
```

### Request schema

```python
class TradeDecisionRequest(BaseModel):
    trade_id: UUID
    instrument: DerivativeInstrument
    counterparty: Counterparty
    notional: Money
    jurisdiction: Jurisdiction
    rfq_context: Optional[RFQContext]
```

### Response schema

```python
class TradeDecisionResponse(BaseModel):
    trade_id: UUID
    decision: Literal["PASS", "FLAG", "FAIL"]
    rubric_results: dict[str, RubricCheck]
    llm_review: Optional[LLMReview]
    reasoning: str
    audit_hash: str
    audit_url: str
    decision_at: datetime
```

### Compliance rubric

Mifid II checks (structural): RTS 2 pre-trade transparency, EMIR Article 9 reporting, RTS 27/28 best execution, RTS 22 transaction reporting.

Dodd-Frank checks (structural): Title VII swap data repository, CFTC mandatory clearing, CFTC Part 43 real-time reporting, cross-border swap rules.

LLM review fires only on AMBIGUOUS structural results. Output constrained to RubricCheck schema.

### AWS deployment

```
VPC → ALB → ECS Fargate (FastAPI, 1-10 replicas)
Secrets Manager: anthropic-api-key (90-day rotation)
S3 audit bucket (Object Lock, 7-year retention)
CloudWatch: structured logs + Prometheus metrics
```

### Terraform module

```
terraform/
  main.tf
  variables.tf
  outputs.tf
  audit_bucket.tf
  observability.tf
```

## Acceptance Criteria

- [ ] `POST /v1/trade-decision` returns correct decision state for 12 reference trades in the test corpus
- [ ] All Mifid II and Dodd-Frank structural checks return deterministic results (byte-identical across 100 replays)
- [ ] LLM review fires only for AMBIGUOUS structural results; never on clear PASS or FAIL paths
- [ ] Every decision call writes a hash-chained audit record to S3 within 5 seconds
- [ ] Audit hash chain verification confirms no tampering across the test corpus
- [ ] Terraform module deploys cleanly from a fresh AWS account with Region and S3 bucket name as required variables
- [ ] Service responds within 500ms P95 for deterministic-only paths
- [ ] Service responds within 3000ms P95 for FLAG paths
- [ ] Anthropic API key never appears in logs, env dumps, or audit records
- [ ] CloudWatch alarms fire on: 4xx rate >5%, 5xx rate >1%, LLM failure rate >2%, audit-write failure
- [ ] Integration tests cover all 8 compliance rules with PASS/FAIL/FLAG examples each
- [ ] README documents local and deployed test corpus execution

## Game Theory Cooperative Model review

The service sits at a multi-party intersection: trading desk, compliance team, risk officer, internal auditor, regulator. Cooperative equilibrium holds when each party's best outcome occurs through honest, complete data submission. Defection (under-reporting trades, mis-classifying instruments) is structurally bounded by the audit hash chain — every decision call leaves an immutable record that surfaces in post-trade reconciliation.

### Abuse vector

Hostile cases considered with Swiss Cheese layered defenses:

1. **Bypass the service entirely** (route trades through an internal path that skips /trade-decision). Mitigation: integration at the RFQ pipeline gate; trades reaching execution without audit_hash are rejected.
2. **Malformed payload to crash service**. Mitigation: Pydantic strict validation; 400 with structured error; no exception bubbles to worker.
3. **Prompt injection to manipulate LLM toward PASS**. Mitigation: LLM never returns the decision; it produces reasoning evaluated against deterministic rubric checks. Injection cannot bypass structural checks.
4. **Tamper with audit S3 bucket**. Mitigation: S3 Object Lock 7-year retention; IAM disallows DeleteObject; tampering produces hash-chain breakage at verification.
5. **Exfiltrate Anthropic API key**. Mitigation: Secrets Manager storage, fetched at task startup; 90-day rotation; IAM scopes secret read to ECS task role; CloudTrail logs access.
6. **Replay old PASS decisions for new non-compliant trades**. Mitigation: per-trade hash sequencing; replay produces hash mismatch.

Defense surface area exceeds expected adversary payoff under all considered cases.

### Mitigation

Composite defense layers operate independently: bypass detection, input validation, LLM containment, audit immutability, secret rotation, replay prevention. No single layer is sufficient; the combination provides defense in depth.

## Subject Migration Summary

| Subject | Before | After |
|---|---|---|
| Pre-trade compliance verification | Manual review by compliance team, ad-hoc | Automated rubric check + LLM review for edge cases |
| Audit trail format | Free-form notes in compliance case management | Hash-chained immutable S3 records, machine-readable |
| Decision latency | Hours to days for human review | <500ms deterministic, <3s FLAG |
| Regulator inquiry response time | Days to assemble evidence | Seconds to produce audit chain for any trade_id |
| Trader workflow | RFQ → execute → discover compliance issue T+1 | RFQ → pre-trade check → execute only on PASS |
| Open questions | Should FLAG block execution or surface a warning with trader override? Should the audit bucket sit in a separate compliance-team account? Should the LLM provider be selectable per-jurisdiction? | resolved on merge |

## Files created / updated

```
src/main.py
src/schemas/trade.py
src/schemas/compliance.py
src/compliance/mifid_ii.py
src/compliance/dodd_frank.py
src/llm/anthropic_client.py
src/llm/prompts.py
src/audit/hash_chain.py
src/audit/s3_writer.py
src/observability/metrics.py
src/config.py
tests/integration/test_decisions.py
tests/unit/test_hash_chain.py
tests/corpus/reference_trades.json
terraform/main.tf
terraform/variables.tf
terraform/outputs.tf
terraform/audit_bucket.tf
terraform/observability.tf
Dockerfile
requirements.txt
README.md
```

## Models Applied

- **#2 Decision Tree** — The Decision Tree section above records architectural decisions D1–D8. Each row enumerates options, the chosen option, and the rationale. Trigger for change documents revisit conditions.
- **#1 Game Theory Cooperative Model** — Applied in the Game Theory section. Five cooperators enumerated with their gains. Six abuse vectors with per-vector mitigation. Honest play shown to dominate.
- **#8 Swiss Cheese Defense** — Applied in Mitigation. Six independent defense layers operating independently. No single layer is sufficient.
- **#15 Inversion / Premortem** — Applied in Abuse vector. Six adversary cases enumerated with failure-mode-first framing.
- **#16 Mechanism Design** — Audit hash chain creates a mechanism where defection produces detectable artifacts. Three-state decision taxonomy is incentive-compatible.

## Legal triggers

Trigger #2 — regulatory compliance scope. The service implements Mifid II and Dodd-Frank rule sets. Implementation correctness is the firm's responsibility; the encoded rubric is a structural starting point requiring validation by the firm's compliance team and external counsel against current regulatory text before production deployment. Encoded rule references reflect the regulatory landscape as of the spec authoring date.

Trigger #4 — open source license declaration. If published as open source, license declaration is required at publication. Recommend Apache-2.0 for permissive reuse with patent grant. Final license choice deferred to publication time.

Trigger #5 — third-party API dependency (Anthropic). Service depends on a third-party LLM API. Vendor contract terms (data handling, training opt-out, SLA) are the firm's responsibility to validate before production deployment.

No PHI, no PCI, no PII beyond counterparty LEI (public identifier data per ISO 17442). No royalty obligations or liability exposure beyond standard "as-is" disclaimers.

## Work Estimate

### Active operator time

| Phase | Wait dependency | Estimate |
|---|---|---|
| Schema definition (request/response, rubric checks) | None | 4 hours |
| Compliance rule implementations (8 rules) | Schemas complete | 12 hours |
| LLM integration (client + prompts + structured output) | Schemas complete | 6 hours |
| Audit hash chain (construction + S3 writer + verification) | Schemas complete | 4 hours |
| Terraform module (ECS, ALB, IAM, Secrets Manager, S3) | None | 8 hours |
| Integration test suite (12 reference trades) | All implementation complete | 6 hours |
| Documentation (README, module usage) | Implementation complete | 2 hours |
| **Total** | — | **42 hours** |

### Wall-clock time

| Phase | Wait dependency | Estimate |
|---|---|---|
| Schema definition | None | 1 day |
| Compliance rule implementations | Schemas complete | 3 days |
| LLM integration | Schemas complete | 2 days |
| Audit hash chain | Schemas complete | 1 day |
| Terraform module | None | 2 days (parallel) |
| Integration test suite | All implementation complete | 2 days |
| AWS account setup + first deploy | Terraform complete | 1 day (waits on AWS access) |
| Documentation | Implementation complete | 1 day |
| **Total** | — | **2-3 weeks** |

### Assumptions

- Operator has AWS account access with Terraform IAM role pre-provisioned
- Anthropic API key available; no procurement delay
- Compliance team available for rule-set validation within 1 week of first deploy
- No regulatory amendments published mid-build that materially change Mifid II or Dodd-Frank scope
- Test corpus of 12 reference trades is provided by the firm's compliance team

### Actuals (filled post-execution)

| Phase | Wait dependency | Estimate | Actual | Delta |
|---|---|---|---|---|
| Schema definition | None | 4 hours | TBD | TBD |
| Compliance rule implementations | Schemas complete | 12 hours | TBD | TBD |
| LLM integration | Schemas complete | 6 hours | TBD | TBD |
| Audit hash chain | Schemas complete | 4 hours | TBD | TBD |
| Terraform module | None | 8 hours | TBD | TBD |
| Integration test suite | All implementation complete | 6 hours | TBD | TBD |
| Documentation | Implementation complete | 2 hours | TBD | TBD |
| **Total** | — | **42 hours** | TBD | TBD |
