# Usage Examples

## /harness-self-check

```
You:    /harness-self-check
Agent:  Tell me briefly: what AI agent are you using, and what's it doing for you?
You:    Claude Code, building a B2B platform...
Agent:  You probably have some kind of goal doc — a PRD, a spec, something.
        When was the last time you opened it and checked whether it still
        matches what you're building today?
You:    Hmm... it's been a while.
Agent:  So your agent has been faithfully working toward a goal you wrote
        three months ago, while your actual product has moved on. Every
        hour it spends is optimizing the wrong thing.
        ...
Agent:  ## Harness Diagnostic Report
        Self-deceptions found: 4/5
        The biggest gap: The Stale Goal...
        The fix: Open your PRD, rewrite the first paragraph to match
        what you're actually building. 30 minutes.
```

## /harness-audit

```
You:    /harness-audit
Agent:  [scans files, CI config, git history]

        # Harness Audit Report

        ## Findings

        ### Warning: The Stale Goal — DETECTED
        docs/PRD.md last modified 94 days ago. CLAUDE.md does not
        reference it. Your agent doesn't know this file exists.

        ### Pass: The Monolith — NOT DETECTED
        CLAUDE.md is 72 lines, links to 4 sub-documents. Healthy.

        ### Warning: The Paper Rule — DETECTED
        5 "don't do X" rules in CLAUDE.md. Only 2 have automated checks.
        3 are paper-only.

        Want me to create a check script for the 3 paper rules?
```
