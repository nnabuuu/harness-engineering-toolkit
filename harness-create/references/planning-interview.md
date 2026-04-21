# Planning Interview Questions

Detailed question sequences for Phase 1 of harness creation. The orchestrator (SKILL.md) handles routing — this file contains only the interview content.

---

## Route A: Iterative Improvement (Phases 1-4)

### Phase 1: Task Understanding

Ask these questions. Wait for answers before proceeding.

1. **What is the artifact?** What concrete thing will the agent be working on?
   - A document (article, report, spec)?
   - Code (a module, test suite, configuration)?
   - Data (dataset cleanup, classification, organization)?
   - A structured knowledge base (memory, taxonomy, index)?

2. **What is the current state?** Does the artifact already exist, or is it being created from scratch?
   - If exists: where is it, what format, how far along?
   - If from scratch: what inputs/materials are available?

3. **Who is the audience / consumer?** Who will judge the final output? What do they care about?

4. **What does "done" look like in your head?** Don't worry about precision yet — describe the ideal outcome in your own words.

---

### Phase 1.5: Prerequisites & Environment

Before defining eval criteria, establish what must be true for the harness to run at all.

1. **What environment does this run in?**
   - Local machine, specific directory, Docker container, remote server?
   - If code: what runtime? (Node.js version, Python venv, etc.)

2. **What must be installed or available before starting?**
   - Tools (compilers, linters, CLIs)
   - Services (database, dev server, external APIs)
   - Credentials (API keys, auth tokens — note: harness won't store these, just checks they exist)

3. **Is there a dev/test environment that needs to be running?**
   - Dev server? Database? Docker containers?
   - How to verify it's ready? (URL to hit, port to check, command to run)

4. **What could break during the run?**
   - Token expiration, service downtime, disk space, rate limits?
   - For each: how would we detect it? How should the harness react — pause or fail?

---

### Phase 2: Defining the Optimization Target

Help the user decompose vague goals into scorable dimensions.

**Technique: The "Complaints" Method**

Instead of asking "what's good?", ask:
- "What would you complain about if you saw the current version?"
- "What are the top 3 things that would make you say 'this isn't good enough'?"
- "If someone else did this task, what mistakes would annoy you most?"

Complaints are natural eval criteria inverted. Each complaint maps to a scoring dimension.

**Technique: The "Rubric Draft" Method**

Based on the complaints, propose 4-7 scoring dimensions. For each:

```
[Dimension Name] (weight: X/100)
- 5/5: [describe what excellence looks like]
- 3/5: [describe what acceptable looks like]
- 1/5: [describe what failure looks like]
- Detection method: [how an AI evaluator would assess this]
```

Present this draft rubric to the user. Iterate until they agree. The weights matter — they encode what the user actually cares about most.

**Common pitfalls to catch:**
- All dimensions weighted equally → push user to prioritize
- Dimensions that overlap significantly → merge them
- Dimensions that require human taste with no proxy → flag as "human review gate" items
- Missing negative criteria (things to penalize) → ask "what should the agent definitely NOT do?"

---

### Phase 3: Agent Architecture

Based on the task type and eval criteria, determine the agent roles needed.

**Default architecture: Generator + Evaluator**

This covers 80% of overnight tasks. The Generator improves the artifact; the Evaluator scores it against the rubric from Phase 2.

**When to add more agents:**

| Signal | Additional Agent | Role |
|--------|-----------------|------|
| Multiple distinct skill domains | **Specialist** | Handles one specific dimension (e.g., a "fact-checker" separate from a "style editor") |
| Risk of scope drift | **Planner** | Reviews progress.md at each iteration and decides what to focus on next |
| High cost of errors | **Guardrail** | Pre-screens Generator output before it becomes the new version |
| Output requires specific tools | **Tool Agent** | Runs linters, tests, scrapers, or other deterministic checks |

For each agent, define:
- **Role description**: One sentence on what it does
- **Perspective instruction**: How it should think (e.g., "You are a skeptical reviewer who has NOT seen the writing process")
- **Input**: What files/context it receives
- **Output**: What it produces and where it saves it
- **Isolation requirement**: Must it run in a separate context? (Almost always yes for Evaluator)

---

### Phase 4: Guardrails and Exit Conditions

Define when the loop should stop and what it should not do.

**Exit conditions** (at least one required):
- Score threshold: "Stop when total score >= X"
- Max iterations: "Run at most N iterations"
- Diminishing returns: "Stop if score improves by < Y points for 2 consecutive iterations"
- Time budget: "Run for at most T hours"

**Guardrails** (task-specific):
- What the agent must NOT change (e.g., "do not alter the core thesis")
- Format constraints (e.g., "output must remain valid Markdown")
- Size constraints (e.g., "article must stay between 3000-4000 characters")
- Content constraints (e.g., "no new technical jargon")
- Rollback condition: when should a version be discarded rather than iterated on?

---

## Route B: Investigation/Diagnostic (Phases 1D-3D)

### Phase 1D: Symptom Description

Precisely describe the observable bug behavior. Ask:

1. **What is the symptom?** What does the user see/experience that's wrong?
   - Get the exact user-visible behavior, not an interpretation
   - "Widget shows 3 times" is good, "Widget has a rendering bug" is too vague

2. **What is the expected behavior?** What *should* happen instead?

3. **Is it reproducible?** What exact steps trigger it? Every time, or intermittent?

4. **What is the blast radius?** Does this affect one feature or many? One user or all?

5. **What has already been tried?** Any debugging already done? What was ruled out?

**Output:** A precise symptom statement that any developer could verify independently.

**Prerequisites (lightweight):**
- Can the symptom be reproduced in the current environment?
- What services/tools must be running to reproduce it?

---

### Phase 2D: Hypothesis Generation

Based on the symptom and the code architecture, generate 3-5 ranked hypotheses.

**Technique: The "Call Chain" Method**

Trace the data/control flow from user action to bug manifestation:
1. Map the complete call chain (user → frontend → backend → service → response → render)
2. At each boundary crossing, ask: "Could the bug originate here?"
3. Each potential origin point becomes a hypothesis

**For each hypothesis, define:**

```
### H{N}: [Hypothesis Name] (Likelihood: high/medium/low)
- **Claim**: [What would need to be true for this to be the cause]
- **Verification method**: [Concrete steps to confirm or eliminate]
- **Expected evidence if TRUE**: [What you'd observe]
- **Expected evidence if FALSE**: [What you'd observe]
- **Files to inspect**: [Specific file paths]
```

**Ranking criteria:**
- How many observed symptoms does this hypothesis explain?
- Is this the simplest explanation (Occam's razor)?
- Is this at a boundary crossing (where bugs are most common)?

**Common pitfalls to catch:**
- All hypotheses at the same layer → push to consider different layers
- No hypothesis explains ALL symptoms → may be a combination
- Hypothesis requires "magic" (unexplained mechanism) → too vague, refine it

---

### Phase 3D: Evidence Collection Plan

For each hypothesis, define 1-2 executable verification steps.

**Good verification steps are:**
- **Deterministic**: The result either confirms or eliminates the hypothesis
- **Non-destructive**: Reading code, adding logs, inspecting events — not modifying behavior
- **Ordered by information value**: The step that eliminates the most hypotheses goes first

**For each step:**
```
### Verification Step V{N}.{M}
- **Target hypothesis**: H{N}
- **Action**: [Read file X lines Y-Z / Run command / Check SSE event stream / ...]
- **Look for**: [Specific pattern, value, or absence]
- **If found**: H{N} is CONFIRMED → stop investigating, document root cause
- **If not found**: H{N} is ELIMINATED → proceed to next hypothesis
- **If ambiguous**: [What additional step to take]
```

**Step ordering strategy:**
1. Start with the cheapest steps (file reads) before expensive ones (running services, capturing events)
2. Start with steps that can eliminate multiple hypotheses at once
3. If H1 has a quick file-read verification, do that before H2's runtime verification
