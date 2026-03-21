# Pre-Deploy Quality Checklist
## Squad-Informed, AI-Executable

Use this before any production deploy. Items marked with `[AI]` can be automated by Claude with current MCP access. Items marked `[MANUAL]` require human verification. Items marked `[NEEDS ACCESS]` could be automated with additional MCP tooling.

---

## 1. Feature Flags (PM + FE Engineer)

- [ ] `[AI]` List all LaunchDarkly flags touched by this PR (`ldcli` via MCP)
- [ ] `[AI]` Verify each flag is enabled in staging with correct targeting rules
- [ ] `[AI]` Verify flag is OFF in production (if gradual rollout planned)
- [ ] `[MANUAL]` Confirm rollout plan: who turns it on, what % to start, what's the kill switch criteria?

**Why this matters:** "Feature works in staging but not prod" is almost always a flag that's on in one and off in the other.

## 2. Config & Environment (FE Engineer + Cannabis SME)

- [ ] `[AI]` Diff environment config files between staging and prod branches (GitHub MCP)
- [ ] `[NEEDS ACCESS]` Verify required env vars exist in target environment (Vault/secrets manager)
- [ ] `[NEEDS ACCESS]` Confirm service endpoints point to correct environment (no staging URLs in prod config)
- [ ] `[AI]` Check if any new dependencies were added — do they require env vars that might not be set?
- [ ] `[MANUAL]` If database migration: has it been run in staging? Did it succeed? Check migration log.

**Why this matters:** 80% of "it worked on my machine" incidents are a missing env var or a URL pointing at the wrong environment.

## 3. API Contracts & Dependencies (FE Engineer + Data Analyst)

- [ ] `[AI]` Check if any API request/response types changed in this PR (grep for schema/type changes)
- [ ] `[AI]` Search for other services that import from or call the changed endpoints (Zoekt code search)
- [ ] `[AI]` Check Datadog APM for current error rates on affected services — establish baseline before deploy
- [ ] `[MANUAL]` If breaking API change: have downstream consumers been updated? Is there a deprecation period?

**Why this matters:** Changing a response shape that another team depends on is the fastest path to a 2am page.

## 4. Post-Deploy Smoke Test (PM + Cannabis SME + Data Analyst)

Run within 15 minutes of deploy:

- [ ] `[AI]` Check Datadog error rate for affected services — compare to pre-deploy baseline
- [ ] `[AI]` Search Datadog logs for new ERROR/FATAL entries since deploy timestamp
- [ ] `[AI]` Verify key monitors are green (list specific monitors by name)
- [ ] `[AI]` Check APM latency — any p99 spikes on affected endpoints?
- [ ] `[MANUAL]` One manual smoke test of the happy path in production (place a test order, verify POS receives it)

**Why this matters:** Most config issues show up in the first 10 minutes as error spikes or latency jumps. Catching them early means rollback vs. incident.

## 5. Rollback Plan (PM + FE Engineer)

- [ ] `[MANUAL]` Can this be rolled back with a git revert + redeploy? Or does it require migration rollback?
- [ ] `[MANUAL]` If database migration: is it reversible? What's the rollback SQL?
- [ ] `[AI]` Identify the last known good commit hash (GitHub MCP — last successful deploy)
- [ ] `[MANUAL]` Who is on-call and aware this deploy is happening?

**Why this matters:** "How do we undo this?" asked during an incident is too late. Answer it before you deploy.

## 6. Domain-Specific (Cannabis SME)

- [ ] `[MANUAL]` If touching compliance (age verification, purchase limits, tax calculation): has legal/compliance reviewed?
- [ ] `[MANUAL]` If touching POS integration: has the change been tested against the actual POS terminal, not just the API mock?
- [ ] `[MANUAL]` If touching menu/product display: does it handle dispensaries with 50 products the same as dispensaries with 5,000?
- [ ] `[AI]` If touching pricing/tax: run Datadog query comparing tax calculation outputs before/after in staging

**Why this matters:** Cannabis retail has regulatory requirements that generic ecommerce doesn't. A tax calculation bug isn't just a bug — it's a compliance violation.

---

## How to Use This With Claude

Before deploying, paste into Claude Code:

```
Run the pre-deploy checklist for PR #<number>.
Check LaunchDarkly flags, diff configs, query Datadog baselines,
and flag anything that looks off.
```

Claude will execute all `[AI]` items and report back with a pass/fail for each, plus any warnings. `[MANUAL]` items get listed as a reminder for the human deployer.

---

## What This Catches (and What It Doesn't)

**Catches:**
- Flag not enabled in target environment
- Missing or mismatched env vars
- Error rate spikes immediately post-deploy
- Breaking API changes without consumer updates
- New dependencies with unset config

**Does NOT catch:**
- Race conditions under production load
- Data-dependent bugs (works for 99% of dispensaries, fails for one edge case)
- Third-party service outages
- Slow-burn issues that take days to manifest
- "Works in staging, breaks in prod" due to data differences

*Generated by Claude with 5-persona squad methodology. Last updated 2026-03-21.*
