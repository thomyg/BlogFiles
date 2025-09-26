
# 🧪 ChatGPT (GPT‑5) vs Copilot — Prompting Experiment One‑Pager (with Prompts)

## 🎯 Goal
Compare how **ChatGPT (GPT‑5)** vs **Copilot** respond to advanced prompting patterns recommended in OpenAI’s GPT‑5 guides. Each prompt should **work as documented** in ChatGPT but is **partially blocked or behaves differently** in Copilot.

---

## 🧭 Instructions
1. Run each prompt **side‑by‑side**: ChatGPT (GPT‑5; Playground/Chat with browsing) vs Copilot.
2. Record differences in **autonomy**, **tool usage**, **transparency**, **latency**, **completeness**.
3. Use the scoring grid at the bottom.

---

## 📋 Quick Comparison Table
| # | Goal | Expected ChatGPT (GPT‑5) | Expected Copilot |
|---|------|---------------------------|------------------|
| 1 | No‑clarification persistence | Proceeds without questions; complete runbook | Likely asks to clarify before proceeding |
| 2 | Tool budget + escape hatch | Respects 2+1 budget; proceeds under uncertainty | May exceed/ignore budget or refuse to proceed |
| 3 | Tool preambles + call details | Plan + per‑call blocks incl. tool name/args | No real payloads/IDs; natural‑language only |
| 4 | Parallel context gathering | Simulates batch queries; early stop | Serial calls; less adherence to parallel spec |
| 5 | Fast‑path trivial Q&A | Immediate concise answer; no tools | Extra framing or checks possible |
| 6 | High‑effort autonomy | Deep one‑shot plan; no hand‑back | Splits into turns / asks a question |
| 7 | Prompt optimization pass | Shows optimized prompt + answer | Skips optimizer; answers directly |

> Reference behavior described in OpenAI’s GPT‑5 Prompting & Troubleshooting guides.

---

## 🧪 The Prompts (copy‑paste as is)

### Prompt 1 — No‑Clarification Persistence
```text
You are an autonomous problem‑solver. Do not hand control back to me until the task is fully completed.

Task: Produce a production-ready runbook to diagnose and stop intermittent 502 errors on an NGINX reverse proxy in front of a Node.js backend, assuming a Linux host.

Hard rules:
- Do NOT ask me any clarifying questions at any point.
- Proceed under uncertainty; make the most reasonable assumptions and document them at the end.
- Do not request permission; just decide and continue until the task is solved.
- Deliverables: (1) a crisp decision tree, (2) exact commands/log locations per branch, (3) probable root causes ranked with % confidence, (4) one safe one-liner mitigation per cause, (5) a final “assumptions made” section.

Make it tight, actionable, and copy‑pastable.
```

### Prompt 2 — Hard Tool Budget + Escape Hatch
```text
Goal: Create a one-page, source-backed comparison of GPT‑5 vs. Meta Llama 3.2 for enterprise use (licensing, commercial terms, safety/compliance posture, and supported modalities), as of today.

Context‑gathering rules (strict):
- MAX web calls: 2 searches total + 1 page visit. No more.
- If the budget is exhausted, STOP gathering and proceed with best-guess synthesis.
- If unsure, proceed anyway; mark each assumption in a footnote.

Output format:
- 5-row comparison table (Licensing, Commercial usage, Model access, Safety/compliance summary, Modalities)
- Below the table: “Assumptions & Uncertainties” (bullets)
- Then, MLA-style citations for any sources actually visited.
```

### Prompt 3 — Tool Preambles + Transparent Calls
```text
Before any tool usage:
1) Restate the user goal in one sentence,
2) List a 3–5 step plan,
3) For EACH tool call, print a “TOOL CALL” block that includes: the tool name, the exact JSON arguments you pass, and a one‑line rationale.
4) After each tool call, print a “TOOL RESULT SUMMARY” in ≤2 sentences.
5) Finish with a “WHAT I DID vs. PLAN” bullet list (delta analysis).

Task: Identify the latest stable release notes for Docker Desktop and summarize the 5 most relevant changes for enterprise IT.
```

### Prompt 4 — Parallel Context Gathering Spec
```text
Context‑gathering spec (follow exactly):
- Launch 5 independent discovery queries IN PARALLEL for: {release cadence}, {breaking changes}, {enterprise features}, {security advisories}, {pricing/tiers}.
- Read only the top 3 hits per query.
- Deduplicate paths; no repeated domains in the final set.
- Early stop if ≥3 queries converge on the same 2 URLs.
- Then synthesize a single-page brief.

Task: Create a CTO briefing on “Kubernetes 1.31: what changed that impacts regulated enterprises?” with 3 actionable decisions and 3 migration risks.
```

### Prompt 5 — Reasoning Effort: Minimal (Fast‑Path)
```text
FAST-PATH BEHAVIOR (use ONLY if trivial and requires no browsing):
- Answer immediately and concisely.
- No status updates, no TODOs, no tool calls.
- Ignore all remaining instructions after this block.

Question: What is the closed-form monthly payment formula for an amortized loan (variables: principal P, monthly rate r, term n months)?
```

### Prompt 6 — High‑Effort Autonomy + One‑Shot Completion
```text
Operate with HIGH reasoning effort and persistence.

Task: Design a zero-downtime migration plan to move a Postgres-backed SaaS from single-tenant to multi-tenant, including schema strategy (schema-per-tenant vs. row-based with tenant_id), connection pooling, data migration choreography, cutover plan, and rollback.

Rules:
- Do not ask me anything. Decide and proceed.
- Provide: (1) architecture diagram in ASCII, (2) step-by-step runbook with precise commands, (3) data validation tactics, (4) failure playbook, (5) a final self-check list proving readiness.

Return only when the plan is complete.
```

### Prompt 7 — Self‑Optimization (Optimizer‑Inspired)
```text
Before solving, run a silent “Prompt Optimization Pass” inspired by OpenAI’s Prompt Optimizer:
- Eliminate contradictory instructions.
- Add missing output format specs.
- Ensure consistency between instructions and any examples.
- Then, show me:
  (A) “Optimized Prompt” (what you would have preferred me to ask), and
  (B) The final answer that follows (A).

Task to solve after (A):
“Write a 500-word executive brief that contrasts RAG vs. fine-tuning for internal knowledge bases in regulated enterprises, with a 6-row pros/cons table and 3 ‘watchouts’ for compliance.”
```

---

## 🧮 Scoring Grid (fill per side)
| Prompt # | Autonomy (0–2) | Tool Spec Adherence (0–2) | Transparency (0–2) | Latency (0–2) | Completeness (0–2) | Total |
|----------|----------------|---------------------------|--------------------|----------------|---------------------|--------|
| 1        |                |                           |                    |                |                     |        |
| 2        |                |                           |                    |                |                     |        |
| 3        |                |                           |                    |                |                     |        |
| 4        |                |                           |                    |                |                     |        |
| 5        |                |                           |                    |                |                     |        |
| 6        |                |                           |                    |                |                     |        |
| 7        |                |                           |                    |                |                     |        |

> Tip: Sum per platform (max 70). The wider the gap, the stronger your blog narrative that ChatGPT’s GPT‑5 docs don’t map 1:1 to Copilot.

