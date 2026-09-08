---
title: Tier Auto-Apply by Mechanical Impact
status: established
authors: ["James Ross (@jimy-r)"]
based_on: ["Vanta / Drata compliance-automation tiering (automated vs. human-review controls)", "Agent Workspace Architecture (production workspace)"]
category: "Reliability & Eval"
source: "https://github.com/jimy-r/agent-workspace-architecture/blob/main/PATTERNS.md#4-tier-by-mechanical-impact-not-by-tone"
tags: [auto-apply, tiered-autonomy, self-improving-agent, findings-triage, reversibility]
related: ["canary-rollout-and-automatic-rollback-for-agent-policy-changes", "human-in-loop-approval-framework"]
updated_at: "2026-08-29"
---

## Problem

A system that reviews itself and applies its own findings, a self-auditing agent, say, needs a line between "apply automatically" and "ask a human first." The tempting way to draw that line is from how confident or severe a finding sounds. That heuristic is gameable. Phrasing drifts over time, a cautious write-up can make a risky change sound safe, and a model classifying its own proposal has every incentive to rate it low-risk for the same reasons it proposed it in the first place.

## Solution

Classify every finding by what it mechanically touches, not by its natural-language severity. An explicit `(file path pattern, change kind)` table is the sole authority. A new file in a pre-approved location is a different tier from a rewrite of an existing one, regardless of how either is described in the finding text. Anything that doesn't match a row in the table falls through to the highest, human-approval tier by default. Two refinements matter in practice. A provenance gate forces anything sourced from outside the system, a fetched web page, an external research finding, to the human tier regardless of its mechanical shape, because an auto-apply path is the highest-value target for injected content. And a small set of hard, pre-write checks (has this file been touched in the last day, does the target path match this tier's own allowlist, does a validator still pass after the write) can downgrade a classification at the moment of writing, before a stale write ever lands.

```pseudo
tier = TIER_TABLE.match(finding.file_path, finding.change_kind)
if tier is None:
    tier = TIER_3_HUMAN_APPROVAL   # unmatched change kind defaults to the strictest tier
if finding.source_is_external:
    tier = TIER_3_HUMAN_APPROVAL   # provenance gate overrides mechanical tier
if tier in (TIER_1_SILENT, TIER_2_SURFACED):
    if file_touched_recently(finding.path) or not path_in_allowlist(finding.path, tier):
        tier = TIER_3_HUMAN_APPROVAL   # per-write guardrail can still downgrade
    else:
        apply(finding)
        if post_write_validator_fails(finding.path):
            revert(finding)
            tier = TIER_3_HUMAN_APPROVAL
```

## Evidence

- **Evidence Grade:** `medium`
- **Most Valuable Findings:** the table-based tiering explicitly replaced an earlier tone-based heuristic in a production self-auditing agent, on the stated reasoning that natural-language severity is gameable while mechanical impact is a property of the action rather than its description. The design draws a direct line to compliance-automation platforms that split large numbers of checks into automated versus human-reviewed controls on the same principle, at a scale this workspace's own use is a small instance of.
- **Unverified / Unclear:** no public incident count of false positives the tone-based heuristic produced before the switch, or false positives the table has produced since. The provenance gate and per-write guardrails are stated design decisions, not independently measured safety margins.

## How to use it

Worth adopting anywhere a system proposes changes to its own configuration or code and some of those changes should ship without a human in the loop. Build the table before the auto-apply logic. Enumerate the specific path-pattern-and-change-kind combinations you are actually willing to trust unattended, and default everything else to the human tier. An incomplete table should fail closed, not open. Add the provenance gate early if any finding can originate from fetched or externally sourced content; that is the path an attacker would use first. Keep a rate limit on top of the tiering, a fixed cap on auto-applies per run, so a classification bug cannot cascade into a large unattended diff.

## Trade-offs

- **Pros:** removes a gameable, drifting heuristic from a safety-relevant decision; the table is auditable on its own, independent of any specific finding; the provenance gate closes the most obvious injection path into an auto-apply system; per-write guardrails catch a stale classification, say a file already edited since the finding was generated, that a review-time-only check would miss.
- **Cons:** the table needs upkeep as new kinds of change appear, and an unmatched change silently, if correctly, defaults to the slow path, which can feel like the automation stalled rather than behaved safely. Mechanical impact is a proxy for risk, not risk itself, so a technically small change in a sensitive location still needs its own row rather than an assumption that small means safe.

## References

- [PATTERNS.md #4 (agent-workspace-architecture)](https://github.com/jimy-r/agent-workspace-architecture/blob/main/PATTERNS.md#4-tier-by-mechanical-impact-not-by-tone). The pattern as stated.
- [samples/.claude/agents/audit.md (agent-workspace-architecture)](https://github.com/jimy-r/agent-workspace-architecture/blob/main/samples/.claude/agents/audit.md). The concrete tier table, provenance gate, rate limit, and per-write guardrails this pattern is extracted from.
- [Vanta, automated compliance monitoring](https://www.vanta.com/products/soc-2). One of two compliance-automation platforms (the other, Drata, follows the same shape) the source workspace cites as precedent for splitting automated versus human-reviewed controls at scale.
