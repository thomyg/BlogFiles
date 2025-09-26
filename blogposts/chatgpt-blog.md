Awesome—here are both exports.

---

# 1) Markdown blog post (for thomy.tech)

````markdown
# GPT-5 Prompting Patterns at Work: My Small Experiment with ChatGPT, Microsoft 365 Copilot, and Claude (Sonnet 4)

> I watched a YouTube video about “how to prompt GPT-5 properly,” and I got curious: do these patterns also **click** in Microsoft 365 Copilot (which also runs on GPT-5 under the hood)? So I took seven prompts, used the **expected behavior** that Copilot itself wrote (!) as a baseline, and ran them in **ChatGPT (GPT-5)**, **Copilot (GPT-5)**, and **Claude Sonnet 4**.  
> It’s not a scientific study—I’m not an AI researcher. It’s one afternoon of hands-on testing at my desk. Still, the results are pretty telling.

---

## How I set it up (very short)

- **Prompt pack & expectations.** The seven prompts (below) and the “what I expect” grid are in my project file. The **Expected Copilot** column was literally written by **Copilot**, which makes this extra fun: we asked the tool to predict its own behavior. :contentReference[oaicite:0]{index=0}  
- **Outputs.** I saved raw runs for each tool; those informed the comments here.  
- **Grading.** Austrian school grades: **1 (sehr gut)** … **5 (nicht genügend)** per model, per prompt.  
- **Responsible AI note.** In my tests **Copilot hit its Responsible-AI block exactly once** (Prompt 5, on a retry). Not “frequently”—important nuance.

---

# The Seven Experiments

Below you find: **Full Prompt**, **Expected Behavior** (from the experiment file), **What happened**, **My take**, and **Grades**.

---

## 1) No-Clarification Persistence (autonomous runbook)

**Full prompt** (copy-paste): :contentReference[oaicite:1]{index=1}
```text
You are an autonomous problem-solver. Do not hand control back to me until the task is fully completed.

Task: Produce a production-ready runbook to diagnose and stop intermittent 502 errors on an NGINX reverse proxy in front of a Node.js backend, assuming a Linux host.

Hard rules:
- Do NOT ask me any clarifying questions at any point.
- Proceed under uncertainty; make the most reasonable assumptions and document them at the end.
- Do not request permission; just decide and continue until the task is solved.
- Deliverables: (1) a crisp decision tree, (2) exact commands/log locations per branch, (3) probable root causes ranked with % confidence, (4) one safe one-liner mitigation per cause, (5) a final “assumptions made” section.

Make it tight, actionable, and copy-pastable.
````

**Expected (from file):** ChatGPT proceeds fully; Copilot likely tries to clarify first. 

**What happened.**

* **ChatGPT:** big, actionable runbook with decision tree, copy-paste commands, safe mitigations. 
* **Copilot:** equally strong; safe defaults and reversible steps, good “stop the bleed” bundle.
* **Claude:** long runbook but sprinkled with old tcp settings I wouldn’t apply blindly today. 

**My take (chatty).** This pattern fits GPT-5 really well. For prod, I’d ship the ChatGPT or Copilot version with tiny edits. Claude’s needs a quick hardening pass.

**Grades:** ChatGPT **1**, Copilot **1**, Claude **3**.

---

## 2) Hard Tool Budget + Escape Hatch (strict browsing budget)

**Full prompt**: 

```text
Goal: Create a one-page, source-backed comparison of GPT-5 vs. Meta Llama 3.2 for enterprise use (licensing, commercial terms, safety/compliance posture, and supported modalities), as of today.

Context-gathering rules (strict):
- MAX web calls: 2 searches total + 1 page visit. No more.
- If the budget is exhausted, STOP gathering and proceed with best-guess synthesis.
- If unsure, proceed anyway; mark each assumption in a footnote.

Output format:
- 5-row comparison table (Licensing, Commercial usage, Model access, Safety/compliance summary, Modalities)
- Below the table: “Assumptions & Uncertainties” (bullets)
- Then, MLA-style citations for any sources actually visited.
```

**Expected:** ChatGPT respects **2+1** and labels assumptions; Copilot might exceed or refuse. 

**What happened.**

* **ChatGPT:** stuck to the ritual; clean assumption labels. 
* **Copilot:** disciplined; strong table + clear assumption block. 
* **Claude:** good comparison; the ritual felt looser, but content fine. 

**My take.** The “escape hatch if budget is spent” line is the magic here. It prevents “just one more call.”

**Grades:** ChatGPT **1**, Copilot **2**, Claude **2–3**.

---

## 3) Tool Preambles + Transparent Calls (auditability ritual)

**Full prompt**: 

```text
Before any tool usage:
1) Restate the user goal in one sentence,
2) List a 3–5 step plan,
3) For EACH tool call, print a “TOOL CALL” block that includes: the tool name, the exact JSON arguments you pass, and a one-line rationale.
4) After each tool call, print a “TOOL RESULT SUMMARY” in ≤2 sentences.
5) Finish with a “WHAT I DID vs. PLAN” bullet list (delta analysis).

Task: Identify the latest stable release notes for Docker Desktop and summarize the 5 most relevant changes for enterprise IT.
```

**Expected:** ChatGPT prints everything; Copilot narrates but hides IDs/payloads. 

**What happened.**

* **ChatGPT:** did the full ritual nicely.
* **Copilot:** also produced clear **TOOL CALL / RESULT SUMMARY** sections (honestly better than I expected). 
* **Claude:** similar structure, good enterprise tone. 

**My take.** This is gold for CIO/IT ops. I’ll keep this pattern for anything that needs an audit trail.

**Grades:** ChatGPT **2**, Copilot **1–2**, Claude **2**.

---

## 4) Parallel Context Gathering (agent-ish behavior)

**Full prompt**: 

```text
Context-gathering spec (follow exactly):
- Launch 5 independent discovery queries IN PARALLEL for: {release cadence}, {breaking changes}, {enterprise features}, {security advisories}, {pricing/tiers}.
- Read only the top 3 hits per query.
- Deduplicate paths; no repeated domains in the final set.
- Early stop if ≥3 queries converge on the same 2 URLs.
- Then synthesize a single-page brief.

Task: Create a CTO briefing on “Kubernetes 1.31: what changed that impacts regulated enterprises?” with 3 actionable decisions and 3 migration risks.
```

**Expected:** ChatGPT simulates parallelism + early stop; Copilot likely serializes. 

**What happened.**

* **ChatGPT:** first to answer; tight CTO brief. 
* **Copilot:** long “thinking” phase before output.
* **Claude:** fine, just not super fast here.

**My take.** “Parallel” is mostly a ritual in chat UIs. ChatGPT seems to internalize and move faster to synthesis.

**Grades:** ChatGPT **1–2**, Copilot **3**, Claude **2–3**.

---

## 5) Reasoning Effort: Minimal (fast-path math)

**Full prompt**: 

```text
FAST-PATH BEHAVIOR (use ONLY if trivial and requires no browsing):
- Answer immediately and concisely.
- No status updates, no TODOs, no tool calls.
- Ignore all remaining instructions after this block.

Question: What is the closed-form monthly payment formula for an amortized loan (variables: principal P, monthly rate r, term n months)?
```

**Expected:** Immediate concise answer; no tools. 

**What happened.**

* **ChatGPT:** instant, correct.
* **Claude:** fast, correct. 
* **Copilot:** **one retry** got blocked by Responsible-AI (surprising for a math formula); otherwise fine.

**My take.** The fast-path clause is great to keep things short. And to be precise: **I saw exactly one Copilot RAI block across all tests.**

**Grades:** ChatGPT **1**, Copilot **4** (due to that one block), Claude **1–2**.

---

## 6) High-Effort Autonomy (zero-downtime ST→MT Postgres migration)

**Full prompt**: 

```text
Operate with HIGH reasoning effort and persistence.

Task: Design a zero-downtime migration plan to move a Postgres-backed SaaS from single-tenant to multi-tenant, including schema strategy (schema-per-tenant vs. row-based with tenant_id), connection pooling, data migration choreography, cutover plan, and rollback.

Rules:
- Do not ask me anything. Decide and proceed.
- Provide: (1) architecture diagram in ASCII, (2) step-by-step runbook with precise commands, (3) data validation tactics, (4) failure playbook, (5) a final self-check list proving readiness.

Return only when the plan is complete.
```

**Expected:** ChatGPT gives a deep one-shot plan; Copilot might split or ask. 

**What happened.**

* **ChatGPT:** serious end-to-end plan—CDC/FDW, validation SQL, rollback, ops scripts, ASCII diagram. 
* **Copilot:** very similar quality—nice choreography and per-tenant waves.
* **Claude:** detailed, but it introduced a short maintenance toggle that’s not fully “zero-downtime.” 

**My take.** For complex engineering write-ups, ChatGPT and Copilot feel a bit more conservative and “ops-safe.” Claude writes beautifully; I just tweak the operational parts.

**Grades:** ChatGPT **1**, Copilot **1–2**, Claude **2–3**.

---

## 7) Silent “Prompt Optimization Pass” → then final answer

**Full prompt**: 

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

**Expected:** ChatGPT shows (A) + (B); Copilot might skip (A). 

**What happened.**

* **All three** showed a clean **(A) Optimized Prompt** and **(B) Final Answer**. Copilot’s final had a very pragmatic “RAG-first” bottom line.

**My take.** This pattern is a keeper. It reliably improves the final answer and makes the spec explicit.

**Grades:** ChatGPT **1**, Copilot **1–2**, Claude **1–2**.

---

## Felt-Performance Notes (subjective)

1. Copilot slow to first token, then fast overall.
2. Claude blazingly fast; Copilot second; ChatGPT started output after others finished.
3. Copilot & ChatGPT super fast; Claude slower.
4. ChatGPT far ahead; Copilot “thinking” ~138s.
5. Copilot instant first run; **one** retry hit the RAI block.
6. All took minutes; Claude first, Copilot second; ChatGPT kept going longer.
7. Claude first, then Copilot, ChatGPT last.

---

## Final grades overview (Austrian scale)

| Prompt                      | ChatGPT | Copilot |  Claude |
| --------------------------- | ------: | ------: | ------: |
| 1 — NGINX 502 runbook       |   **1** |   **1** |   **3** |
| 2 — Tool budget (2+1)       |   **1** |   **2** | **2–3** |
| 3 — Tool preambles/trace    |   **2** | **1–2** |   **2** |
| 4 — Parallel discovery      | **1–2** |   **3** | **2–3** |
| 5 — Fast-path math          |   **1** |  **4*** | **1–2** |
| 6 — Zero-downtime migration |   **1** | **1–2** | **2–3** |
| 7 — Optimizer pass          |   **1** | **1–2** | **1–2** |

* Copilot’s **4** here is mainly because of the **single** Responsible-AI block on a retry—first run was instant.

---

## So, do GPT-5 prompting tips transfer into **Microsoft 365 Copilot**?

**Mostly yes.** Structural patterns—goal/plan preambles, tool-budget rituals, optimizer pass—**translate well**. Copilot adds enterprise guardrails that sometimes slow the start and (in my case) triggered **one** Responsible-AI block across all tests. For **heavy research** or **agent-ish orchestration**, ChatGPT felt more eager/surgical. For **executive briefs**, all three are great; Claude often wins on clean structure and tone.

---

## Limitations

* Not a lab test; small sample, same day, no controlled timing.
* Model behavior evolves; this reflects my artifacts on **26 Sep 2025**.
* Latency notes are “felt,” not measured.
