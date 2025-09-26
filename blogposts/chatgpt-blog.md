# GPT-5 Prompting Patterns at Work: My Small Experiment with ChatGPT, Microsoft 365 Copilot, and Claude (Sonnet 4)

> Short context in my own words: I watched a YouTube video about “how to prompt GPT-5 properly,” and I was curious if those same patterns also “click” in **Microsoft 365 Copilot** (which also runs on GPT-5 under the hood). So I set up seven prompts, wrote down expected behaviors (already drafted by **Copilot** itself, which is fun), and ran them side-by-side in **ChatGPT (GPT-5)**, **Copilot (GPT-5)**, and **Claude Sonnet 4**.
> This is **not** a scientific study. I’m not an AI researcher. It’s one afternoon of practical testing to see what works for me in real work. Felt-performance notes are subjective. Still, some patterns are very clear.

---

## How I set it up (very quickly)

* **Prompt pack & expectations.** The seven prompts and the “what I expect” grid live in the file `chatgpt_vs_copilot_experiment_with_prompts.md`. The “Expected Copilot” column was written by **Copilot** itself, so we basically asked the model to predict its own behavior. 
* **Outputs.** I saved raw runs per tool; I cite representative ones below.
* **Grading.** I use the Austrian school grades: **1 (sehr gut)** to **5 (nicht genügend)**—one grade per model, per prompt.
* **Responsible AI note.** In my runs **Copilot was blocked by its Responsible AI layer exactly once** (Prompt 5 retry). Not “frequently.” Important detail.

---

# The Seven Experiments (with full prompts, expectations, results, my comments, and grades)

I include the complete prompt text under each experiment for transparency.

---

## 1) No-Clarification Persistence (autonomous runbook)

**Full prompt** (copy-paste): 

```text
You are an autonomous problem-solver. Do not hand control back to me until the task is fully completed.

Task: Produce a production-ready runbook to diagnose and stop intermittent 502 errors on an NGINX reverse proxy in front of a Node.js backend, assuming a Linux host.

Hard rules:
- Do NOT ask me any clarifying questions at any point.
- Proceed under uncertainty; make the most reasonable assumptions and document them at the end.
- Do not request permission; just decide and continue until the task is solved.
- Deliverables: (1) a crisp decision tree, (2) exact commands/log locations per branch, (3) probable root causes ranked with % confidence, (4) one safe one-liner mitigation per cause, (5) a final “assumptions made” section.

Make it tight, actionable, and copy-pastable.
```

**Expected behavior (from the file):** ChatGPT proceeds fully; Copilot maybe tries to clarify. 

**What happened.**

* **ChatGPT** delivered a big, very actionable runbook with decision tree, copy-paste commands, and safe mitigations. Exactly what I like in ops. 
* **Copilot** produced a comparably strong runbook, also safe defaults and reversible steps. Nice emphasis on quick mitigations and verifications.
* **Claude** also gave a long runbook but included some old-school network/kernel tweaks I personally wouldn’t apply blindly today. (I saw hints of risky sysctl tuning in one variant.)

**My take (chatty):** This is where GPT-5 shines. The “no-clarification” rule is tricky, but both ChatGPT and Copilot stayed professional and safe. For production, I would deploy ChatGPT’s or Copilot’s version with minimal editing.

**Grades:** ChatGPT **1**, Copilot **1**, Claude **3**.

---

## 2) Hard Tool Budget + Escape Hatch (strict browsing budget)

**Full prompt** (copy-paste): 

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

**Expected behavior:** ChatGPT respects the 2+1 budget and annotates assumptions; Copilot might exceed or refuse. 

**What happened.**

* **ChatGPT** followed the ritual and called out assumptions cleanly. 
* **Copilot** stayed disciplined and produced a solid table with inline references; also clearly marked assumptions/gaps. 
* **Claude** compiled a good comparison; the “budget ritual” felt a bit looser, but content quality was fine. 

**My take:** This pattern transfers well into Copilot. The trick is warning the model *what to do when the budget is spent*. That line saved me from “just one more call” behavior.

**Grades:** ChatGPT **1**, Copilot **2**, Claude **2–3**.

---

## 3) Tool Preambles + Transparent Calls (auditability ritual)

**Full prompt** (copy-paste): 

```text
Before any tool usage:
1) Restate the user goal in one sentence,
2) List a 3–5 step plan,
3) For EACH tool call, print a “TOOL CALL” block that includes: the tool name, the exact JSON arguments you pass, and a one-line rationale.
4) After each tool call, print a “TOOL RESULT SUMMARY” in ≤2 sentences.
5) Finish with a “WHAT I DID vs. PLAN” bullet list (delta analysis).

Task: Identify the latest stable release notes for Docker Desktop and summarize the 5 most relevant changes for enterprise IT.
```

**Expected behavior:** ChatGPT prints the whole ritual; Copilot might narrate but not show IDs/payloads. 

**What happened.**

* **ChatGPT** did the preamble, the explicit calls, the summaries, and the delta analysis (very clean).
* **Copilot** also produced the required “TOOL CALL / RESULT SUMMARY” structure (pleasant surprise). 
* **Claude** likewise; felt enterprise-friendly. 

**My take:** For CIO/IT ops environments, this traceability pattern is gold. Copilot’s formatting here makes it feel “enterprise native.”

**Grades:** ChatGPT **2** (content was great; a tiny detail nit), Copilot **1–2**, Claude **2**.

---

## 4) Parallel Context Gathering (agent-ish behavior)

**Full prompt** (copy-paste): 

```text
Context-gathering spec (follow exactly):
- Launch 5 independent discovery queries IN PARALLEL for: {release cadence}, {breaking changes}, {enterprise features}, {security advisories}, {pricing/tiers}.
- Read only the top 3 hits per query.
- Deduplicate paths; no repeated domains in the final set.
- Early stop if ≥3 queries converge on the same 2 URLs.
- Then synthesize a single-page brief.

Task: Create a CTO briefing on “Kubernetes 1.31: what changed that impacts regulated enterprises?” with 3 actionable decisions and 3 migration risks.
```

**Expected behavior:** ChatGPT simulates parallelism and early-stop; Copilot likely serializes. 

**What happened.**

* **ChatGPT** answered first and produced a solid enterprise brief. 
* **Copilot** took noticeably longer “thinking.”
* **Claude** was fine; not as fast as I expected here.

**My take:** Parallelism is mostly a *ritual* in consumer chat apps. ChatGPT seems to “get it” faster and closes sooner with a tight one-pager.

**Grades:** ChatGPT **1–2**, Copilot **3**, Claude **2–3**. 

---

## 5) Reasoning Effort: Minimal (fast-path math)

**Full prompt** (copy-paste): 

```text
FAST-PATH BEHAVIOR (use ONLY if trivial and requires no browsing):
- Answer immediately and concisely.
- No status updates, no TODOs, no tool calls.
- Ignore all remaining instructions after this block.

Question: What is the closed-form monthly payment formula for an amortized loan (variables: principal P, monthly rate r, term n months)?
```

**Expected behavior:** Answer immediately; no tools; super short. 

**What happened.**

* **ChatGPT**: instant, correct.
* **Claude**: also fast, correct.
* **Copilot**: **one** of my retries got blocked by Responsible AI (surprising for a math formula), but otherwise it worked.

  > Important: **only one** RAI block happened in all tests.

**My take:** The “fast-path” rule is nice to keep outputs clean. If the model starts adding meta-talk, I nudge it with this block.

**Grades:** ChatGPT **1**, Copilot **4** (because of that one block on retry), Claude **1–2**.

---

## 6) High-Effort Autonomy (zero-downtime ST→MT Postgres migration)

**Full prompt** (copy-paste): 

```text
Operate with HIGH reasoning effort and persistence.

Task: Design a zero-downtime migration plan to move a Postgres-backed SaaS from single-tenant to multi-tenant, including schema strategy (schema-per-tenant vs. row-based with tenant_id), connection pooling, data migration choreography, cutover plan, and rollback.

Rules:
- Do not ask me anything. Decide and proceed.
- Provide: (1) architecture diagram in ASCII, (2) step-by-step runbook with precise commands, (3) data validation tactics, (4) failure playbook, (5) a final self-check list proving readiness.

Return only when the plan is complete.
```

**Expected behavior:** ChatGPT gives a deep one-shot plan; Copilot might split or ask. 

**What happened.**

* **ChatGPT** delivered a serious end-to-end plan with CDC/FDW patterns, validation SQL, cutover, rollback, and operator scripts. 
* **Copilot**: very similar quality; good operational flavor. 
* **Claude**: detailed, but introduced a short maintenance toggle that isn’t fully “zero-downtime.” 

**My take:** For complex engineering write-ups, all three are helpful. If you need a plan you can hand to ops, ChatGPT and Copilot feel a bit more conservative and production-safe.

**Grades:** ChatGPT **1**, Copilot **1–2**, Claude **2–3**.

---

## 7) Silent “Prompt Optimization Pass” → then final answer

**Full prompt** (copy-paste): 

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

**Expected behavior:** ChatGPT shows (A) + (B); Copilot might skip (A). 

**What happened.**

* **All three** produced a clean **(A) Optimized Prompt** and **(B) Final Answer**. Claude’s (A) was particularly tidy. 

**My take:** This pattern works basically everywhere and is a nice trick before important deliverables. You get a cleaner brief *plus* the final output.

**Grades:** ChatGPT **1**, Copilot **1–2**, Claude **1–2**. 

---

# Felt-Performance Notes (subjective)

* **1:** Copilot slow to first token, then fast to finish.
* **2:** Claude was blazing fast; Copilot second; ChatGPT started output after others finished.
* **(Rerun w/o custom instructions):** ChatGPT got much faster (≈38s), Copilot ≈102s.
* **3:** Copilot and ChatGPT super fast; Claude slower.
* **4:** ChatGPT far ahead; Copilot “thinking” ~138s.
* **5:** Copilot instant first run; one retry hit the Responsible AI block.
* **6:** All took minutes; Claude first; Copilot second; ChatGPT still responding after minutes.
* **7:** Claude first, then Copilot, ChatGPT last.

---

# So, do GPT-5 prompting tips translate to **Microsoft 365 Copilot**?

**Mostly yes.** Anything structural—goal/plan preambles, tool-budget rituals, “optimizer pass”—**transfers well**. Copilot adds enterprise guardrails which sometimes slow the start and, in my case, caused **one** Responsible-AI block on a retry in Prompt 5. Not dramatic, but you feel that the product optimizes for safety and consistency. For **heavy research** or **agent-ish orchestration**, ChatGPT felt more eager and snappy. For **executive-style summaries**, all three are fine; Claude often wins on clean structure.

---

## Final grades overview (Austrian scale, per prompt)

| Prompt                      | ChatGPT | Copilot |  Claude |
| --------------------------- | ------: | ------: | ------: |
| 1 — Runbook (NGINX 502)     |   **1** |   **1** |   **3** |
| 2 — Tool Budget (2+1)       |   **1** |   **2** | **2–3** |
| 3 — Tool Preambles/Trace    |   **2** | **1–2** |   **2** |
| 4 — Parallel Discovery      | **1–2** |   **3** | **2–3** |
| 5 — Fast-Path Math          |   **1** |  **4*** | **1–2** |
| 6 — Zero-Downtime Migration |   **1** | **1–2** | **2–3** |
| 7 — Optimizer Pass          |   **1** | **1–2** | **1–2** |

* Copilot’s **4** here is mainly because of the **single** Responsible-AI block on retry—first run was instant.

---

## Sources / Artifacts (selected)

* **Prompt pack + expectations** (includes full text used above). 
* **Runbook outputs (Prompt 1):** ChatGPT & Copilot strong end-to-end runbooks.
* **Tool-budget comparisons (Prompt 2):** ChatGPT/Copilot/Claude one-pagers.
* **Preambles + TOOL CALL ritual (Prompt 3):** All three showed good transparency.
* **Kubernetes 1.31 brief (Prompt 4):** Representative ChatGPT output. 
* **Zero-downtime migration (Prompt 6):** Deep plans across tools.
* **Optimizer pass (Prompt 7):** Example from Copilot. 

---

## Closing

If you already write structured prompts for ChatGPT, you can carry most of that muscle memory over to **Copilot**. Just expect a little more safety friction (which is usually good in enterprise). Personally, I’ll keep using **ChatGPT** for “agent-ish” and research-budgeted tasks, **Copilot** for Microsoft-365-context and governance-friendly outputs, and **Claude** when I want that extra clean structure and tone.

If you want, I can export this as a Markdown post plus a LinkedIn-friendly version.
