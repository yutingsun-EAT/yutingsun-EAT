# Tax Agent AI — One-Pager

**Problem statement (one line):** In B2B SaaS tax & finance ops, revenue reported by the internal billing system does not match revenue reported by the third-party tax platform (e.g., Avalara / Vertex / TaxJar), and finance must determine the *correct* sales tax liability — but the root cause is often subtle and the mismatch doesn't self-resolve.

---

## 1. Assumptions (what we're ruling out / framing the real problem)

We scope intentionally. Assumptions that shape the design:

- **The discrepancy is real, not a config bug.** Both systems are pointed at the same entity, currency, and time period. We assume the baseline is correct and the gap comes from *content* (classification), not plumbing.
- **The user is a finance/tax analyst or controller** — financially literate, but not a tax attorney or engineer. They need to *trust* the answer and see the evidence.
- **The third-party tax platform is the authoritative source of *rules*.** It knows the jurisdiction and rate logic. The internal billing system owns the *product/SKU → tax category* meaning.
- **The taxonomy is manually curated and error-prone.** It encodes *business-specific meaning* that no external system knows, maintained across a large, changing catalog by humans with no continuous validation.
- **Rule changes are continuous and silent.** States update what's taxable (digital services, software, fees), and systems drift because someone must keep the mapping in sync.
- **Timing differences auto-reconcile; classification differences do not.** A pure timing mismatch (booked in the wrong period) closes at year-end. A taxability-classification mismatch persists until someone manually decides which side is correct and true-ups or refunds.
- **We scope to one hero root cause** — **taxability classification** — because it's the mismatch that *won't* self-resolve and where an agent's judgment adds the most value.

---

## 2. Problem

The revenue/tax numbers between two systems don't match, and finance can't tell why without deep investigation.

**Two root causes drive persistent (non-auto-reconciling) gaps:**

1. **Taxonomy drift** — the internal SKU → tax-category mapping goes stale. New SKUs are created without a category (or with the wrong one), and one system defaults to *exempt* while the other defaults to *taxable*. Since it's manually curated across a large, changing catalog with no continuous validation, gaps build silently.

2. **Regulation change** — a state changes what's taxable. The rule changed, but the internal classification didn't catch up. This error can appear with *zero human action* — purely from an external rule update.

Because the mismatch is a **classification disagreement**, it does **not** auto-resolve at year-end (unlike a timing difference). Someone has to investigate, determine which side is correct, and drive a true-up or refund.

---

## 3. Why this problem is important

- **Financial accuracy & compliance.** Sales tax liability must be correct; errors mean overpaying or underpaying — the latter is an audit/compliance risk.
- **Time cost.** A single discrepancy investigation is manual and slow. Finance analysts burn hours digging through line items, rules, and system configs.
- **Silent accumulation.** Because gaps build silently (new SKUs, stale mappings, rule changes), the problem compounds over time and surfaces as a surprise at tax time.
- **Trust & decision quality.** The user must be able to *explain* the correct liability with evidence, not just trust an opaque number. Auditability matters in finance.
- **Scalability.** Manual curation doesn't scale as the catalog and jurisdictions grow. This is an ongoing, evergreen problem — not a one-time fix.

---

## 4. Involved Users

| Role | Who | What they need |
|------|-----|----------------|
| **Finance / Tax Analyst** (primary) | Financially literate, not a tax attorney | Understand *why* numbers differ, see evidence, confirm/correct, get to the right liability fast |
| **Tax / Revenue Controller** (approver) | Owns the liability decision | Confidence + audit trail before approving a true-up or filing |
| **Catalog / RevOps** (secondary) | Maintains SKU taxonomy | Clear signal when a category is wrong and what to fix it to |
| **Third-party tax platform** | Avalara / Vertex / TaxJar | External source of rules + computed liability (a system, not an interactive user) |
| **AI Agents** (the solution) | The system we design | Continuously prevent, detect, and resolve classification gaps |

---

## 5. Solutions

### Option A — Single Agent (reactive debugging)

**One** agent that a Finance analyst invokes when a discrepancy is already flagged.

- It investigates the mismatch, pulls both sides' classifications, determines which is correct, explains it with evidence, and recommends a true-up/refund.
- **Pros:** Simple to build, low latency, one clear interface, easy to explain.
- **Cons:** **Reactive only.** It fixes the case you bring it, but it does nothing to *prevent* the next one. The same taxonomy drift and rule-change gaps keep happening. The Finance team has to keep *noticing* issues and bringing them to the agent.

### Option B — Multi-Agent (prevent + detect + resolve, self-correcting)  ⭐ preferred

A **small set of specialized agents** that work together so the system prevents problems, catches residual ones, and self-corrects over time.

| Agent | Role | Creates value by |
|-------|------|------------------|
| **Agent 1 — Taxonomy Guardian** | Maintains SKU → tax category | Continuously reviews existing SKUs for correctness and *applies* a category to new SKUs (instead of leaving them blank/default) |
| **Agent 2 — Regulatory Watcher** | Tracks external rule changes | Watches state/jurisdiction rule changes; notifies Agent 1 (fix taxonomy) **and** Agent 3 (proactive heads-up to Finance) |
| **Agent 3 — Finance-facing Tax Ops PM** | The human-facing interface | Runs the reconciliation loop (internal vs. third-party), checks third-party platform config when debugging, investigates & explains *which side is correct*, alerts Finance on rule changes/conflicts, escalates edge cases to a human, and tracks cases to closure |

**How they work together:**

```
Agent 2 (rules watcher) ──> Agent 1 (fix taxonomy)
                        └─> Agent 3 (heads-up Finance)

[cheap background reconciliation] ──flags mismatch──> Agent 3 (investigate, explain, escalate)
```

- **Prevention:** Agents 1 & 2 keep things right before they break.
- **Detection:** a *cheap* background reconciliation (not a separate heavy agent — keeps latency low) flags residual mismatches and feeds Agent 3.
- **Resolution + self-correction:** Agent 3 investigates, explains, gets human confirmation, and the corrected classification feeds back into Agent 1's taxonomy — so the system **learns and prevents recurrence.**

**Why Option B is preferred:**

1. **Prevents recurrence, not just fixes it.** Option A patches today's problem; Option B keeps the taxonomy correct and the rules fresh, so the *same* issue doesn't come back.
2. **Self-corrects.** The multi-agent loop feeds resolutions back into the taxonomy, so the system improves over time and reduces future manual work.
3. **Proactive, not reactive.** Finance gets a *heads-up* on rule changes and emerging conflicts *before* they become a surprise tax bill or audit flag.
4. **Right-sized.** Only 3 agents — each has a distinct, explainable job — plus a cheap background check for detection. This keeps latency low and avoids over-engineering (every extra agent adds a round-trip and coordination overhead).
5. **Human-in-the-loop.** Agent 3 escalates genuinely ambiguous cases (bundles, edge cases) to a human and tracks to closure. The model is "AI does the continuous grunt work; humans do the judgment on edge cases" — the honest, defensible framing.

**Trade-off / note on self-correction:** Self-correction still requires **human prompting/approval** on the judgment calls (e.g., confirming the correct classification on an ambiguous item). The agents *suggest* and *automate the grunt work*, but the human confirms the edge cases. That keeps the system safe and auditable.

---

## 6. What we deliberately cut

- **Timing/recognition** as the hero root cause — it auto-reconciles at year-end, so it's less compelling than classification.
- **A 4th dedicated "detection" agent** — detection is a cheap mechanical comparison, so it's a background check feeding Agent 3, not a separate agent (keeps latency low).
- **A full tax-filing workflow** — that's downstream/outsourced to a service/CPA; out of scope.
- **Automated unilateral decisions on ambiguous items** — edge cases always escalate to a human for confirmation.

## 7. Metrics / how we'd measure success

- **Time-to-resolve** a discrepancy (goal: cut materially vs. manual baseline).
- **% of investigations resolved without escalation** (the agent gets it right on its own more often).
- **% of discrepancies prevented** (taxonomy stay-correct rate; rule-change catch rate before impact).
- **User confidence** rating on the agent's explanation (auditability = trust).
- **Surprise-tax-bill/audit-flag rate** (should trend down as prevention kicks in).

---

*Prepared for the PM design take-home / interview walkthrough. 1 page; reviews the same problem → multi-agent vs. single-agent → recommendation.*
