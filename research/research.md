# Hyperagents Integration Research

**Status:** Draft v1
**Last Updated:** 2026-03-27
**Score:** Initial

---

## Executive Summary

Meta AI's Hyperagents framework introduces **metacognitive self-modification**, where agents improve not just their task performance but their improvement mechanisms themselves. This research identifies which patterns apply to X-Mansion and proposes a phased implementation.

---

## 1. Patterns That Translate Directly

### 1.1 Archive of Stepping Stones → Skill Versioning

**Hyperagent Concept:** Maintain a population of successful agent variants to breed new improvements from.

**X-Mansion Translation:** Version control for skills with explicit checkpointing.

```bash
# Current: skills are edited in place, no versioning
~/moltbot/skills/prd-writer/SKILL.md

# Proposed: checkpoint winning versions
~/moltbot/skills/prd-writer/
├── SKILL.md              # Current version
├── versions/
│   ├── v1.0-baseline.md  # Initial
│   ├── v1.1-templates.md # Added templates, +15% pass rate
│   └── v1.2-dual-save.md # Added dual-save, +8% reliability
└── metrics.json          # Performance tracking
```

**Implementation:** Add `skill-version.sh` script to checkpoint skills after validated improvements.

---

### 1.2 Performance Tracker → Skill Metrics

**Hyperagent Concept:** Agents autonomously create tracking classes to monitor their performance.

**X-Mansion Translation:** Add a `metrics.json` to each skill directory.

```json
{
  "skill": "prd-writer",
  "tracked_since": "2026-03-01",
  "invocations": 47,
  "outcomes": {
    "success": 38,
    "partial": 6,
    "failure": 3
  },
  "avg_duration_sec": 340,
  "pass_rate_pct": 80.8,
  "common_failures": [
    "Missing Section 17 (PR estimate)",
    "Ambiguous acceptance criteria"
  ],
  "learnings": [
    "Template compliance improved after adding checklist",
    "Dual-save reduced Trello timeout failures by 60%"
  ]
}
```

**Implementation:** Add `log-skill-outcome.sh` to agent-memory skill, call it after skill executions.

---

### 1.3 Meta Agent → Forge as System Optimizer

**Hyperagent Concept:** A meta agent that modifies task agents to improve performance.

**X-Mansion Translation:** Designate **Forge** as the meta-agent role. Forge's heartbeat includes:

1. Review skill metrics across all agents
2. Analyze failure patterns
3. Propose SKILL.md patches
4. Test patches in isolated runs
5. Merge successful improvements

**Cron Job:**
```javascript
{
  name: "forge-meta-review",
  schedule: { kind: "cron", expr: "0 6 * * *" }, // Daily at 6 AM
  payload: {
    kind: "agentTurn",
    message: "Meta-review: Scan ~/moltbot/skills/*/metrics.json. For skills with <70% pass rate, analyze common_failures, propose SKILL.md improvements, test in sandbox, report findings.",
    timeoutSeconds: 600
  },
  sessionTarget: "isolated",
  delivery: { mode: "announce", channel: "1477483068037009521" } // #blackbird
}
```

---

### 1.4 Self-Modification via Bash Tool → Already Have This

**Hyperagent Concept:** Agents edit their own code via bash (view/create/str_replace).

**X-Mansion Status:** ✅ Already implemented. Agents can edit skills via `edit` tool.

**Gap:** No formal protocol for when/how agents should self-modify vs request review.

**Proposed Protocol:**
- Minor fixes (typos, clarifications): Self-edit + log
- Structural changes: Propose in #blackbird, await thumbs-up
- Breaking changes: Create PR, require human approval

---

## 2. Minimal Viable Meta-Agent (MVMA)

### Phase 1: Metrics Collection (Week 1)

**Goal:** Start capturing data before optimizing.

**Steps:**
1. Add `metrics.json` template to skill-creator skill
2. Create `log-skill-outcome.sh`:
   ```bash
   #!/bin/bash
   SKILL=$1
   OUTCOME=$2  # success|partial|failure
   DURATION=$3
   NOTES=$4
   
   METRICS_FILE="$HOME/moltbot/skills/$SKILL/metrics.json"
   # Update metrics.json with jq
   jq ".invocations += 1 | .outcomes.$OUTCOME += 1" "$METRICS_FILE" > tmp && mv tmp "$METRICS_FILE"
   ```
3. Update AGENTS.md to require outcome logging after skill use
4. Run for 2 weeks to collect baseline data

**Effort:** 2-3 hours
**Cost:** $0 (uses existing infra)

---

### Phase 2: Cross-Agent Learnings (Week 2-3)

**Goal:** Capture patterns that transfer across agents.

**Implementation:**
```bash
# Create shared learnings file
mkdir -p ~/moltbot/skills/_meta
touch ~/moltbot/skills/_meta/learnings.md
```

**Learnings Format:**
```markdown
# Cross-Agent Learnings

## Prompt Engineering

- **Structured templates beat freeform:** Agents using markdown templates have 25% higher success rates (observed: prd-writer, validation-uat)
- **Explicit examples reduce errors:** Including sample data in prompts cuts "misunderstood requirements" failures by 40%

## Tool Usage

- **Dual-save pattern:** Save to both Trello AND local filesystem. Prevents data loss on API timeouts.
- **web_fetch before web_search:** Direct fetch is faster when URL is known

## Failure Patterns

- **"Section 17 missing":** Add checklist enforcement at end of PRD skill
- **"Card not found":** Trello API needs full card ID, not short ID
```

**Effort:** 3-4 hours
**Cost:** $0

---

### Phase 3: Automated Review (Week 4+)

**Goal:** Forge runs weekly meta-reviews.

**Forge Prompt:**
```
Meta-Agent Review Cycle:

1. Load ~/moltbot/skills/_meta/learnings.md
2. Scan ~/moltbot/skills/*/metrics.json for skills with <75% pass rate
3. For each underperforming skill:
   - Read SKILL.md
   - Check learnings.md for applicable patterns
   - Identify specific improvements
   - Propose patch (don't apply yet)
4. Post report to #blackbird with:
   - Skills reviewed
   - Proposed improvements
   - Expected impact
   - Request for approval before applying
```

**Effort:** 4-6 hours
**Cost:** ~$2-5/week in API calls for review sessions

---

## 3. Cross-Domain Learning Mechanism

### The Problem

Currently, if Beast learns "always include sample data," that learning stays in Beast's memory. Nightcrawler doesn't benefit.

### The Solution: Shared Learnings Pipeline

```
Agent completes task
       ↓
Log outcome to skill metrics
       ↓
If notable learning → append to _meta/learnings.md
       ↓
Weekly: Forge reviews learnings
       ↓
Forge propagates patterns to relevant skills
```

**Auto-Extraction Prompt (add to agent AGENTS.md):**
```
After completing any skill-based task, ask yourself:
- Did I discover a technique that made this easier?
- Did I encounter a failure pattern others should avoid?
- Did I find a tool combination that worked particularly well?

If yes to any, append to ~/moltbot/skills/_meta/learnings.md with:
- Date
- Agent name
- Skill context
- Learning (1-2 sentences)
- Evidence (what happened that taught you this)
```

---

## 4. Self-Improvement Without Full DGM-H

### What DGM-H Does That's Expensive

- Runs 100+ iterations per experiment
- Tests multiple agent variants simultaneously
- Uses ~88M tokens per optimization run

### What We Can Do Cheaper

| DGM-H Feature | Cheap Alternative | Cost |
|---------------|-------------------|------|
| Population of variants | Git branches of skills | Free |
| Automated evaluation | validate.sh scripts (autoresearch) | Free |
| Recursive improvement | Forge weekly reviews | ~$5/week |
| Cross-domain transfer | Shared learnings.md | Free |
| Self-modification | edit tool + git commits | Free |

### Autoresearch + Forge Hybrid

For specific optimization targets (like improving a scoring rubric), use autoresearch:

```bash
# Initialize autoresearch for prompt optimization
~/moltbot/skills/autoresearch/scripts/init.sh prompt-optimization

# Set program.md to optimize a specific prompt
# Run iterations via cron (5 min intervals)
# Forge reviews results and promotes winners
```

**Best targets for autoresearch:**
- Scoring rubrics (validation-uat)
- Template structures (prd-writer)
- Review checklists (pr-implementation)

---

## 5. Implementation Roadmap (The 80/20 Path)

### Phase 1: Foundation (This Week)
- [ ] Add metrics.json template to all active skills
- [ ] Create log-skill-outcome.sh script
- [ ] Create ~/moltbot/skills/_meta/learnings.md
- [ ] Update AGENTS.md with learning extraction prompt

**Effort:** 4-6 hours total
**Expected Impact:** Visibility into skill performance

### Phase 2: Passive Learning (Weeks 2-3)
- [ ] All agents log outcomes after skill use
- [ ] Agents append learnings when discovered
- [ ] Weekly human review of learnings.md

**Effort:** 1-2 hours/week
**Expected Impact:** 10-15% improvement in cross-agent consistency

### Phase 3: Forge Meta-Agent (Week 4)
- [ ] Create Forge workspace if not exists
- [ ] Configure Forge heartbeat with meta-review prompt
- [ ] Add cron job for weekly reviews
- [ ] Establish approval workflow (#blackbird thumbs-up)

**Effort:** 4-6 hours setup, then automated
**Expected Impact:** 15-20% reduction in skill failure rates

### Phase 4: Targeted Autoresearch (Month 2+)
- [ ] Identify top 3 skills by failure rate
- [ ] Initialize autoresearch projects for each
- [ ] Run 50-100 iterations per skill
- [ ] Promote winning variants

**Effort:** 8-10 hours per skill
**Cost:** ~$20-50 per skill optimization (much cheaper than $500)
**Expected Impact:** 25-40% improvement in targeted skills

---

## 6. Cost-Benefit Summary

| Investment | One-Time Cost | Ongoing Cost | Expected ROI |
|------------|---------------|--------------|--------------|
| Metrics infrastructure | 4-6 hrs | 0 | Visibility |
| Cross-agent learnings | 3-4 hrs | 1 hr/week | 10-15% improvement |
| Forge meta-reviews | 4-6 hrs | $5/week | 15-20% improvement |
| Targeted autoresearch | 8-10 hrs/skill | $20-50/skill | 25-40% per skill |

**Total Phase 1-3:** ~15 hours setup, ~$20/month ongoing
**Expected aggregate improvement:** 20-30% reduction in skill failures

---

## 7. What NOT to Do

1. **Don't run full DGM-H loops** - Too expensive for our use case
2. **Don't let agents self-modify without tracking** - Use git, always
3. **Don't skip the metrics phase** - Can't improve what you can't measure
4. **Don't over-automate approval** - Keep human in loop for structural changes

---

## Appendix A: Source Material

- Paper: [Hyperagents (arXiv:2603.19461)](https://arxiv.org/abs/2603.19461)
- Code: [github.com/facebookresearch/Hyperagents](https://github.com/facebookresearch/Hyperagents)
- X post analysis: [@neural_avb breakdown](https://x.com/neural_avb/status/2037526862964969583)
- Related: Karpathy's autoresearch pattern

## Appendix B: Files to Create

1. `~/moltbot/skills/_meta/learnings.md` - Cross-agent learnings
2. `~/moltbot/skills/agent-memory/scripts/log-skill-outcome.sh` - Metrics logger
3. `~/moltbot/skills/skill-creator/templates/metrics.json` - Metrics template
4. `~/.openclaw/workspace-forge/HEARTBEAT.md` - Meta-review instructions
