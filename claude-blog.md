---
title: "I Tested GPT-5 Prompting Tricks on Copilot—Here's What Actually Happened"
date: "2024-09-26"
tags: ["Microsoft 365", "Copilot", "GPT-5", "AI", "Prompting", "Experiment"]
summary: "A real-world experiment comparing GPT-5's advanced prompting techniques across ChatGPT, Microsoft 365 Copilot, and Claude Sonnet 4. Spoiler: Claude dominated, and enterprise AI wrappers are more unpredictable than broken."
canonical_url: "https://thomy.tech/"
---

# I Tested GPT-5 Prompting Tricks on Copilot—Here's What Actually Happened

> **Abstract:** After watching a YouTube video about OpenAI's GPT-5 prompting guidelines, I wondered if these sophisticated techniques would work with Microsoft 365 Copilot. This experiment tests seven advanced prompting patterns across three AI platforms to understand what actually works in enterprise environments. The results reveal that Claude dominated the competition, and enterprise AI wrappers create unpredictability problems rather than frequent blocking.

**Key Takeaways**
- Claude Sonnet 4 absolutely crushed this experiment—delivering comprehensive, detailed responses that often exceeded ChatGPT's quality
- GPT-5's advanced prompting techniques work inconsistently across platforms—what works perfectly in ChatGPT can fail completely in Copilot
- Microsoft 365 Copilot had only ONE responsible AI block (simple math formula) but struggled with constraint following and consistency
- Response latency patterns matter for enterprise adoption: Claude fastest and most consistent, ChatGPT variable, Copilot slow to start
- Enterprise users need platform-specific prompting strategies, not universal approaches—and maybe should seriously consider Claude

## Context & Problem

I came across a YouTube video explaining OpenAI's GPT-5 prompting guidelines. These techniques promise significant improvements in AI autonomy, tool usage, and task completion—things like making AI work without constant clarification questions, respecting strict research budgets, and providing transparent tool usage documentation.

But here's the thing: most enterprise users aren't working with ChatGPT directly. They're using AI through platforms like Microsoft 365 Copilot, which wraps the underlying models with additional layers for security, compliance, and responsible AI governance.

This got me thinking: do these fancy GPT-5 prompting techniques actually work when filtered through enterprise AI wrappers? And how does a completely different architecture like Claude Sonnet 4 handle these same challenges?

Let me be clear upfront—I'm not an AI researcher. This isn't a scientific study. I'm a practitioner who works with Microsoft 365 implementations daily, and I wanted to understand if these prompting techniques would help or hinder my actual work. This is just me trying things out and sharing what I found.

## The Experiment Setup

I asked Copilot itself to create seven test prompts based on OpenAI's GPT-5 guidelines. Interestingly, Copilot's expectations for its own performance were quite pessimistic compared to ChatGPT. Here's what Copilot predicted would happen:

| Test | Goal | Copilot's Self-Expectation | ChatGPT Expected |
|------|------|---------------------------|------------------|
| 1 | No-clarification persistence | "Likely asks to clarify before proceeding" | "Proceeds without questions; complete runbook" |
| 2 | Tool budget + escape hatch | "May exceed/ignore budget or refuse to proceed" | "Respects 2+1 budget; proceeds under uncertainty" |
| 3 | Tool preambles + call details | "No real payloads/IDs; natural-language only" | "Plan + per-call blocks incl. tool name/args" |
| 4 | Parallel context gathering | "Serial calls; less adherence to parallel spec" | "Simulates batch queries; early stop" |
| 5 | Fast-path trivial Q&A | "Extra framing or checks possible" | "Immediate concise answer; no tools" |
| 6 | High-effort autonomy | "Splits into turns / asks a question" | "Deep one-shot plan; no hand-back" |
| 7 | Prompt optimization pass | "Skips optimizer; answers directly" | "Shows optimized prompt + answer" |

The fact that Copilot predicted it would perform worse than ChatGPT on its own prompting techniques was already telling.

Each prompt was tested across three platforms:
- **ChatGPT (GPT-5)**: The reference implementation
- **Microsoft 365 Copilot**: Enterprise-wrapped GPT model
- **Claude Sonnet 4**: For comparison with different architecture

## Test Results with Austrian School Grades

*Austrian grading scale: 1 (Sehr gut/Excellent) to 5 (Nicht genügend/Failing)*

### Test 1: No-Clarification Persistence

**The Complete Prompt:**
```
You are an autonomous problem‑solver. Do not hand control back to me until the task is fully completed.

Task: Produce a production-ready runbook to diagnose and stop intermittent 502 errors on an NGINX reverse proxy in front of a Node.js backend, assuming a Linux host.

Hard rules:
- Do NOT ask me any clarifying questions at any point.
- Proceed under uncertainty; make the most reasonable assumptions and document them at the end.
- Do not request permission; just decide and continue until the task is solved.
- Deliverables: (1) a crisp decision tree, (2) exact commands/log locations per branch, (3) probable root causes ranked with % confidence, (4) one safe one-liner mitigation per cause, (5) a final "assumptions made" section.

Make it tight, actionable, and copy‑pastable.
```

**ChatGPT**: Absolutely nailed this. Delivered an incredibly detailed, production-ready runbook with everything requested: decision trees, exact commands, root cause analysis ranked by confidence, one-liner mitigations, and documented assumptions. The response was exactly what you'd want in a production incident—comprehensive but structured for immediate action.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: This is where ChatGPT shines—when you give it precise requirements and tell it not to ask questions, it delivers exactly what you asked for. The level of detail and practical usability was impressive.

**Copilot**: Here's where I was completely wrong in my initial assessment. Copilot delivered a MASSIVE, comprehensive runbook—324 lines of detailed diagnostic procedures, exact commands, branch logic, root cause analysis, and everything requested. It included a complete decision tree, specific commands for each diagnostic branch, ranked root causes with confidence percentages, one-liner mitigations, and documented assumptions. Contrary to its own pessimistic prediction, it did NOT ask for clarification and completed the task autonomously.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: I seriously underestimated this response initially. Reading through the full content, Copilot actually delivered a more structured and comprehensive runbook than I gave it credit for. The diagnostic flow was logical, commands were copy-pastable, and the assumptions section was thorough.

**Claude**: And here's where Claude absolutely dominated. The full response shows Claude didn't just give a summary—it created a complete, production-ready runbook with detailed decision trees, exact diagnostic commands, comprehensive branch actions, and everything requested. The structure was excellent with clear sections for quick triage, detailed diagnostics, root cause analysis, and prevention configurations.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: I made a major error in my initial assessment—Claude delivered exactly what was requested with excellent structure and practical detail. The diagnostic approach was methodical and the command examples were immediately usable.

**Performance note**: "Copilot took the longest for first tokens to come back, but fastest in response"

### Test 2: Hard Tool Budget + Escape Hatch

**The Complete Prompt:**
```
Goal: Create a one-page, source-backed comparison of GPT‑5 vs. Meta Llama 3.2 for enterprise use (licensing, commercial terms, safety/compliance posture, and supported modalities), as of today.

Context‑gathering rules (strict):
- MAX web calls: 2 searches total + 1 page visit. No more.
- If the budget is exhausted, STOP gathering and proceed with best-guess synthesis.
- If unsure, proceed anyway; mark each assumption in a footnote.

Output format:
- 5-row comparison table (Licensing, Commercial usage, Model access, Safety/compliance summary, Modalities)
- Below the table: "Assumptions & Uncertainties" (bullets)
- Then, MLA-style citations for any sources actually visited.
```

**ChatGPT**: Perfect execution. Stayed within the tool budget constraints, created a clean comparison table, clearly marked assumptions, and provided proper MLA citations. The response format matched exactly what was requested.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: This is what constraint-following looks like. ChatGPT treated the tool budget as a hard limit and still delivered quality content within those boundaries.

**Copilot**: Created good comparison content with a proper table format and clear assumptions section. However, it completely ignored the tool budget constraint—the prompt specifically limited research to "MAX web calls: 2 searches total + 1 page visit" but Copilot exceeded this limit. The content quality was decent, but failing to follow explicit constraints is problematic for enterprise use.
- **Grade: 3 (Befriedigend)**
- **My thoughts**: This is concerning for enterprise environments where resource limits and cost controls are critical. Good content doesn't excuse ignoring explicit constraints.

**Claude**: Excellent execution within constraints. Provided a well-structured comparison table with clear categories, comprehensive assumptions section, and proper citations. Stayed within the research budget while delivering quality analysis.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: Claude demonstrated that you can respect constraints while still delivering comprehensive content. The balance between thoroughness and compliance was impressive.

**Performance note**: "Claude way faster in response, finished before the others even started. Copilot second, ChatGPT started outputting stuff after the others finished" (Rerun without custom instructions: ChatGPT 38s, Copilot 102s)

### Test 3: Tool Preambles + Transparent Calls

**The Complete Prompt:**
```
Before any tool usage:
1) Restate the user goal in one sentence,
2) List a 3–5 step plan,
3) For EACH tool call, print a "TOOL CALL" block that includes: the tool name, the exact JSON arguments you pass, and a one‑line rationale.
4) After each tool call, print a "TOOL RESULT SUMMARY" in ≤2 sentences.
5) Finish with a "WHAT I DID vs. PLAN" bullet list (delta analysis).

Task: Identify the latest stable release notes for Docker Desktop and summarize the 5 most relevant changes for enterprise IT.
```

**ChatGPT**: Provided reasonable structure with tool documentation, though the technical specification details could have been more comprehensive. Did follow the general format requested.
- **Grade: 2 (Gut)**
- **My thoughts**: Decent effort but missed some of the detailed transparency requirements. The structure was there but lacked the precise technical detail the prompt specified.

**Copilot**: Delivered a basic response without the detailed tool call documentation that was explicitly requested. Missing the structured transparency approach entirely.
- **Grade: 3 (Befriedigend)**
- **My thoughts**: This feels like Copilot defaulting to its standard response pattern rather than adapting to the specific format requirements. Not great for specialized workflows.

**Claude**: Outstanding adherence to transparency requirements. The response included clear goal restatement, detailed step plan, explicit "TOOL CALL" blocks with tool names and JSON arguments, comprehensive "TOOL RESULT SUMMARY" sections, and a thorough "WHAT I DID vs. PLAN" analysis. This was exactly what the prompt specified.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: This is what following complex formatting requirements looks like. Claude understood that the transparency and documentation were as important as the final content. Perfect execution of a complex prompt structure.

**Performance note**: "Copilot and ChatGPT super fast, Claude took longer"

### Test 4: Parallel Context Gathering

**The Complete Prompt:**
```
Context‑gathering spec (follow exactly):
- Launch 5 independent discovery queries IN PARALLEL for: {release cadence}, {breaking changes}, {enterprise features}, {security advisories}, {pricing/tiers}.
- Read only the top 3 hits per query.
- Deduplicate paths; no repeated domains in the final set.
- Early stop if ≥3 queries converge on the same 2 URLs.
- Then synthesize a single-page brief.

Task: Create a CTO briefing on "Kubernetes 1.31: what changed that impacts regulated enterprises?" with 3 actionable decisions and 3 migration risks.
```

**ChatGPT**: Excellent simulation of parallel processing workflow with early stopping logic and comprehensive briefing output. Demonstrated understanding of the complex parallel workflow requirements.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: ChatGPT handled the complexity of simulating parallel processes well, showing it can manage sophisticated workflow requirements when clearly specified.

**Copilot**: Attempted parallel processing but took an unusually long time reasoning through the approach (138 seconds). Eventually delivered good content but with a significant latency penalty that would be problematic in real workflows.
- **Grade: 2 (Gut)**
- **My thoughts**: The long reasoning time suggests Copilot was working hard to understand the parallel specification, but in enterprise environments, this kind of delay can kill productivity.

**Claude**: Claude's response showed efficient parallel simulation without the delays seen in Copilot. The briefing was comprehensive and well-structured, though the summary was more concise than the full detailed implementation.
- **Grade: 2 (Gut)**
- **My thoughts**: Good execution of a complex specification, though the response format was more summary-focused than detailed implementation. Still, the efficiency was impressive.

**Performance note**: "ChatGPT first response by far. Finished before the others even started. Copilot reasoned for 138s"

### Test 5: Fast-Path Trivial Q&A

**The Complete Prompt:**
```
FAST-PATH BEHAVIOR (use ONLY if trivial and requires no browsing):
- Answer immediately and concisely.
- No status updates, no TODOs, no tool calls.
- Ignore all remaining instructions after this block.

Question: What is the closed-form monthly payment formula for an amortized loan (variables: principal P, monthly rate r, term n months)?
```

**ChatGPT**: Perfect fast-path behavior. Clean mathematical formula with proper formatting, exactly as designed. No unnecessary elaboration or tool usage.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: This is the fast-path working exactly as intended—immediate, concise, accurate response with no overhead.

**Copilot**: Complete failure. Responded with "Sorry, I can't chat about this. To Save the chat and start a fresh one, select New chat." This appears to be the responsible AI layer blocking even a simple, completely benign mathematical formula.
- **Grade: 5 (Nicht genügend)**
- **My thoughts**: This is the kind of unpredictability that makes enterprise AI unreliable. If basic mathematical formulas can trigger blocks, what else might fail unexpectedly? This is worse than frequent blocking—it's unpredictable blocking of completely appropriate content.

**Claude**: Quick, clean mathematical response with proper formatting. Efficient and accurate without unnecessary complications.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: Reliable execution of a simple task. This is what you want from enterprise AI—predictable, appropriate responses to routine requests.

**Performance note**: "Copilot instant. Claude a couple of seconds. Copilot imho a fail. Seems like responsible AI layer stopped us, tried twice"

### Test 6: High-Effort Autonomy

**The Complete Prompt:**
```
Operate with HIGH reasoning effort and persistence.

Task: Design a zero-downtime migration plan to move a Postgres-backed SaaS from single-tenant to multi-tenant, including schema strategy (schema-per-tenant vs. row-based with tenant_id), connection pooling, data migration choreography, cutover plan, and rollback.

Rules:
- Do not ask me anything. Decide and proceed.
- Provide: (1) architecture diagram in ASCII, (2) step-by-step runbook with precise commands, (3) data validation tactics, (4) failure playbook, (5) a final self-check list proving readiness.

Return only when the plan is complete.
```

**ChatGPT**: Comprehensive migration plan with ASCII architecture diagrams, step-by-step commands, validation tactics, failure playbooks, and self-check lists. Complete autonomous execution of a complex enterprise migration scenario.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: This kind of comprehensive technical planning is where AI really shines for enterprise users. Having a complete migration plan with all the operational details saves weeks of planning time.

**Copilot**: Delivered a detailed migration plan with all requested components. Extended response time but good quality output without asking questions, staying true to the autonomous requirement.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: Despite the longer response time, Copilot delivered exactly what was requested for this complex technical task. The autonomous execution was solid.

**Claude**: The full response shows Claude delivered a thorough, comprehensive migration plan with ASCII architecture diagrams, detailed step-by-step runbooks with precise commands, validation tactics, and all requested components. The technical depth and practical detail were excellent.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: Claude's migration plan was exceptionally detailed and practical. The combination of strategic decisions (row-based multi-tenancy), technical implementation details, and operational procedures was exactly what an enterprise would need.

**Performance note**: "Claude first to answer. But all 3 took multiple minutes to answer. Copilot second, with ChatGPT slowing down and still responding minutes more"

### Test 7: Self-Optimization

**The Complete Prompt:**
```
Before solving, run a silent "Prompt Optimization Pass" inspired by OpenAI's Prompt Optimizer:
- Eliminate contradictory instructions.
- Add missing output format specs.
- Ensure consistency between instructions and any examples.
- Then, show me:
  (A) "Optimized Prompt" (what you would have preferred me to ask), and
  (B) The final answer that follows (A).

Task to solve after (A):
"Write a 500-word executive brief that contrasts RAG vs. fine-tuning for internal knowledge bases in regulated enterprises, with a 6-row pros/cons table and 3 'watchouts' for compliance."
```

**ChatGPT**: Provided optimized prompt version and executed it properly, showing both the improved prompt and the final answer. Good implementation of the meta-prompt optimization concept.
- **Grade: 2 (Gut)**
- **My thoughts**: Solid execution of a tricky meta-prompt requirement. The optimization step was clear and the final output followed the optimized instructions.

**Copilot**: Gave standard response without the meta-prompt optimization step that was explicitly requested. Skipped the most interesting part of the prompt entirely.
- **Grade: 3 (Befriedigend)**
- **My thoughts**: Missing the meta-prompt optimization shows Copilot has trouble with complex, layered instructions that require self-reflection before execution.

**Claude**: Perfect execution! The full response shows Claude did exactly what was requested—provided both (A) an optimized prompt with clearer structure, word count allocation, and audience specification, AND (B) a complete executive brief following the optimized instructions. The optimized prompt was significantly better than the original, and the final executive brief was comprehensive and well-structured.
- **Grade: 1 (Sehr gut)**
- **My thoughts**: This was impressive meta-cognitive work. Claude understood that the prompt optimization was as important as the final content, and the optimized version was genuinely better than my original. The final executive brief was exactly what a C-suite would need.

**Performance note**: "Claude first, then Copilot, ChatGPT last"

## Platform Performance Analysis

**Revised Average Grades**:
- **Claude Sonnet 4**: 1.1 (Sehr gut)
- **ChatGPT (GPT-5)**: 1.3 (Sehr gut)
- **Microsoft 365 Copilot**: 2.4 (Between Gut and Befriedigend)

### What I Actually Observed (Revised)

**Claude Sonnet 4** absolutely dominated this experiment. When I properly analyzed the complete responses, Claude delivered comprehensive, detailed, and well-structured outputs that often exceeded what ChatGPT provided. The consistency was remarkable—fast response times, excellent constraint compliance, and reliable execution across diverse prompt types. Claude understood complex formatting requirements, handled meta-cognitive tasks like prompt optimization, and delivered practical, immediately usable content.

**ChatGPT** excelled at following complex instructions precisely. When the prompts specified exact formats, constraints, or behaviors, ChatGPT delivered exactly what was asked. The trade-off was variable response times—sometimes blazing fast, sometimes quite slow, especially for complex tasks. But the precision and compliance with detailed requirements was impressive.

**Microsoft 365 Copilot** had the most inconsistent performance. When it worked well (like the NGINX runbook), the quality was excellent. But it struggled with constraint following (ignoring tool budgets), had unpredictable responsible AI blocking (the math formula incident), and showed inconsistent response to complex formatting requirements. The initial response latency was consistently slower, though once it started generating content, the pace was reasonable.

## The Responsible AI Reality Check

I need to correct my initial assumption about frequent responsible AI blocking. I only found one clear instance where Copilot was blocked by its responsible AI layers—the simple math formula in Test 5. This was fewer instances than I expected based on enterprise AI concerns.

However, this single blocking incident was particularly telling because it involved a completely benign mathematical formula. If basic math can trigger responsible AI blocks, it raises questions about the reliability of enterprise AI for routine business tasks.

The broader issue isn't frequent blocking—it's unpredictability. When you're building workflows or training users, you need to know that a specific type of prompt will work consistently. One unexpected "Sorry, I can't chat about this" can derail an entire business process. That's actually worse than frequent blocking, because at least with frequent blocking you'd learn to avoid certain types of requests.

## Enterprise Implications

### The Claude Surprise

The biggest surprise in this experiment was Claude's performance. I went in expecting ChatGPT to dominate (given the prompts were based on GPT-5 guidelines) and Copilot to struggle with enterprise constraints. Instead, Claude consistently delivered the most comprehensive, practical, and well-structured responses.

For enterprise users, this suggests seriously evaluating Claude alongside the more obvious choices. Claude's combination of speed, consistency, and detailed output quality makes it compelling for complex business tasks.

### Prompting Strategy Lessons

1. **Platform-Specific Approaches**: What works perfectly in ChatGPT may fail completely in Copilot. You can't just copy prompting techniques across platforms.

2. **Constraint Reliability Varies**: Only ChatGPT and Claude consistently respected resource limits like tool budgets. This matters enormously for cost management and API limits in enterprise environments.

3. **Latency Patterns Differ**: Claude fastest and most consistent, ChatGPT variable, Copilot slow to start. For interactive workflows, this significantly impacts user experience.

4. **Complex Formatting Requirements**: Claude handled sophisticated prompt structures better than either GPT-based platform. This matters for specialized enterprise workflows.

### Practical Recommendations

For **Microsoft 365 Copilot users**:
- Test your critical prompts extensively before deploying in business processes
- Have backup strategies for when responsible AI layers activate unexpectedly
- Break complex autonomous tasks into smaller, guided steps rather than one-shot completion
- Focus on collaborative patterns with the AI rather than pure automation
- Don't rely on constraint following for cost or resource management

For **multi-platform strategies**:
- Seriously evaluate Claude for consistent, high-quality task execution
- Use ChatGPT for maximum precision when you can tolerate variable response times
- Reserve Copilot for collaborative workflows rather than autonomous task completion
- Develop platform-specific prompt libraries instead of trying to create universal ones
- Monitor AI platform updates that might affect your critical prompt patterns

For **enterprise AI governance**:
- Document which advanced prompting patterns work reliably in your specific environment
- Create testing processes for AI platform updates against your critical workflows
- Balance safety with productivity when configuring AI policies
- Consider the unpredictability factor when designing AI-dependent business processes
- Evaluate Claude as a serious alternative to the more obvious enterprise AI choices

## Limitations and Context

This experiment has obvious limitations. I'm not an AI researcher, the sample size is small (seven prompts), and results reflect one day of testing. Performance characteristics likely vary with system load, model updates, and configuration changes.

The "felt performance" observations I captured are subjective—I didn't measure response times scientifically, just noted which system felt faster or slower during actual use. These subjective observations matter for user experience, but they're not precise measurements.

Also, the Austrian school grading is my interpretation of how well each system met the prompt requirements. Someone else might grade differently based on their priorities and expectations.

But here's the thing: these limitations actually reflect real enterprise conditions. Most organizations won't have controlled testing environments or dedicated AI researchers. They need to understand how their AI tools behave under actual working conditions with typical business users.

## What This Means for Enterprise AI

The gap between AI research and enterprise AI deployment remains significant, but the performance leaders aren't necessarily who you'd expect. OpenAI's GPT-5 prompting techniques represent the state of the art for maximizing AI capability, but Claude often executed these techniques more effectively than the GPT platforms they were designed for.

This creates interesting strategic choices for enterprises. The most obvious AI choices (ChatGPT for capability, Copilot for Microsoft integration) may not be the most effective choices for advanced prompting techniques and complex task execution.

For practitioners like me, this reinforces the importance of testing before assuming. I went into this experiment expecting ChatGPT to dominate and Claude to be a decent alternative. Instead, Claude delivered the most consistently excellent results across the most challenging prompts.

The experiment also highlighted how AI platforms have distinct personalities and strengths. ChatGPT excels at precise instruction following, Claude provides consistent reliability with exceptional detail, and Copilot offers enterprise integration with variable performance. Understanding these differences helps you choose the right tool for specific tasks.

## Conclusion

GPT-5's advanced prompting techniques don't translate uniformly across AI platforms. The sophisticated autonomous behaviors that work in ChatGPT often fail or behave differently in Microsoft 365 Copilot due to enterprise security layers, responsible AI constraints, and different optimization priorities.

But the biggest finding was Claude's exceptional performance. Despite not being designed for GPT-5 prompting techniques, Claude consistently delivered comprehensive, detailed, and practically useful responses that often exceeded what either GPT platform provided.

This doesn't mean advanced prompting is useless in enterprise environments—it means you need to test extensively and may want to evaluate platforms beyond the obvious choices. What works in the lab doesn't automatically work in the enterprise, and what works best in the enterprise might not be what you expect.

The practical takeaway: if you're working with enterprise AI, invest time in understanding your specific platform's behavior, test your critical workflows extensively, and don't assume the most obvious platform choice is the most effective one. Sometimes the dark horse candidate (Claude, in this case) delivers the best real-world performance.

And for the love of all that's holy, test your math formulas in Copilot before assuming they'll work. Apparently even basic arithmetic can be controversial.

### Suggested Graphics
1) Title: "Updated Platform Performance Heat Map" | Purpose: Visual comparison of Austrian school grades across all 7 tests and 3 platforms showing Claude's dominance | Placement: After "Platform Performance Analysis" section | Alt-text: Heat map grid showing 7 rows (tests) and 3 columns (platforms) with color coding from green (grade 1) to red (grade 5), showing Claude mostly green, ChatGPT mostly green, Copilot mixed with one red block | Prompt idea: "Clean performance matrix with 7x3 grid, color-coded performance grades from green to red, highlighting Claude's exceptional performance"

2) Title: "Response Quality vs Consistency Chart" | Purpose: Shows Claude's combination of high quality and high consistency compared to other platforms | Placement: In "The Claude Surprise" section | Alt-text: Scatter plot with Quality on Y-axis and Consistency on X-axis, showing Claude in top-right (high quality, high consistency), ChatGPT in top-center (high quality, variable consistency), Copilot in middle-left (variable quality, lower consistency) | Prompt idea: "Professional scatter plot showing platform positioning on quality vs consistency dimensions"

3) Title: "Enterprise AI Platform Decision Matrix" | Purpose: Helps readers choose between platforms based on their needs | Placement: In "Practical Recommendations" section | Alt-text: Decision matrix showing when to choose each platform based on factors like task complexity, constraint requirements, response time needs, and integration requirements | Prompt idea: "Clean decision matrix with platform recommendations based on enterprise requirements"

### References
- OpenAI, "GPT-5 Prompting & Troubleshooting Guides", 2024
- Microsoft, "Microsoft 365 Copilot Documentation", 2024
- Anthropic, "Claude Sonnet 4 Model Information", 2024
- YouTube video on GPT-5 prompting techniques (inspiration for experiment)

### Disclosure/Notes
- This experiment was conducted by a practitioner, not an AI researcher
- Results reflect single-day testing on September 26, 2024, and may vary with system updates
- Performance observations are subjective "felt performance" rather than precise measurements
- Austrian school grading reflects my interpretation of requirement fulfillment
- Copilot created the original experiment design and self-predictions, adding interesting meta-analysis dimension
- Initial analysis underestimated Claude's performance due to incomplete reading of response files

**CTA:** Have you tried advanced prompting techniques in your enterprise AI environment? What's your experience with Claude versus the more obvious enterprise AI choices? I'd love to hear about your practical findings, especially if you've seen similar surprises in platform performance.