# The 4-Layer Harness Model

Shared reference for all skills in this toolkit.

## Layers

| Layer | Name | Core Question | Scope |
|-------|------|---------------|-------|
| **L1** | Goal Anchoring | Is the agent solving the right problem? | Single task |
| **L2** | Context Engineering | Can it access the right knowledge at the right time? | Single task + cross-session |
| **L3** | Execution Constraints | Is there an automated gate between "done" and "delivered"? | Single task |
| **L4** | Evaluation Loop | Can the system discover unknown problems and turn them into permanent rules? | Cross-session + long-term |

## Priority Rule

Always fix L1 → L2 → L3 → L4. A stale goal makes everything downstream pointless.

## Five Self-Deceptions

| # | Name | You think | Actually | Layer |
|---|------|-----------|----------|-------|
| 1 | The Stale Goal | "I have a goal" | Goal doc is months old, doesn't match current direction | L1 |
| 2 | The Monolith | "I have instructions" | Everything in one file, no priority structure | L2 |
| 3 | The Paper Rule | "I have checks" | Checks are generic defaults, project-specific rules are unenforced | L3 |
| 4 | The Gut Review | "I review everything" | No systematic criteria, quality varies by how busy you are | L4 |
| 5 | The Frozen Harness | "I set up my harness" | Nothing changed in 30+ days, flywheel stopped | L4 |

## Two Scopes

The 4-layer model operates at two scopes:

### Single-Session Harness
Making one agent do one task right: goal is clear, context is loaded, checks run before delivery, output is evaluated.

### Long-Term Harness
Making the system stay right over weeks and months: goals stay current, context doesn't rot, checks evolve with new error patterns, the flywheel keeps turning.

Self-deceptions 1-3 are primarily single-session problems.
Self-deception 5 is a long-term problem.
Self-deception 4 spans both.
