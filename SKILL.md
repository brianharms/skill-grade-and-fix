---
name: grade-and-fix
description: "Visual verification + automatic fix loop. Same as /grade but when a feature FAILS, it uses /hypothesize to diagnose the cause, implements the top fix, and re-screenshots to verify. Use when user says /grade-and-fix, 'grade and fix', 'verify and fix', or wants automated verification with repair."
user-invocable: true
arguments: "Optional: 'all' for session-wide, or a specific scope. No argument = grade and fix the most recent plan."
---

# /grade-and-fix — Visual Verification + Automatic Repair

This skill does everything `/grade` does, then goes further: for each FAIL, it diagnoses the problem, implements a fix, and re-verifies with a new screenshot.

> **Prerequisite — two sibling skills must be installed.** This skill composes `/grade` (the grading protocol) and `/hypothesize` (the 3-agent diagnosis). Before running, verify both exist at `~/.claude/skills/grade/SKILL.md` and `~/.claude/skills/hypothesize/SKILL.md`. If either is missing, install it first:
> ```bash
> git clone https://github.com/brianharms/skill-grade.git       && cp -R skill-grade ~/.claude/skills/grade
> git clone https://github.com/brianharms/skill-hypothesize.git && cp -R skill-hypothesize ~/.claude/skills/hypothesize
> ```
> A browser-screenshot mechanism (e.g. the Claude-in-Chrome MCP) is also required, same as `/grade`.

## Flow

```
For each feature:
  Screenshot → Analyze → PASS? → Done
                       → FAIL? → /hypothesize → Fix → Re-screenshot → PASS or FAIL
```

## Phase 1: Grade Pass

Run the exact same process as `/grade`:
1. Enumerate features from the most recent plan (or specified scope)
2. For each feature: navigate, screenshot, analyze image, PASS or FAIL
3. Track progress: "Graded 4/15"
4. Every feature gets a screenshot — no exceptions, no abandonment

Refer to the `/grade` skill for the full grading protocol. All the same rules apply here, especially the anti-abandonment rules.

## Phase 2: Fix Loop (for FAILs only)

After the full grading pass is complete, process each FAIL:

### Step 1 — Diagnose

Use the `/hypothesize` skill: spawn 3 parallel agents to independently diagnose the failure. Each agent gets:
- The feature name and expected behavior
- What the screenshot showed (the specific visual problem)
- The relevant source file path and approximate line range

The agents return ranked hypotheses. Take the top-ranked one.

### Step 2 — Fix

Implement the suggested fix. Keep it minimal — fix the specific problem, don't refactor surrounding code.

### Step 3 — Re-verify

1. Reload the page (cache-bust: `?cachebust=<timestamp>`)
2. Navigate to the same feature
3. Take a new screenshot
4. Analyze the image: does it now show expected behavior?

### Step 4 — Verdict

- **FIXED**: The new screenshot shows correct behavior. Record both the original fail screenshot and the fix-proof screenshot.
- **STILL FAILING**: The fix didn't work. Note what was tried. Leave as FAIL for user review. Do NOT attempt a second fix — the user will decide.

## Phase 3: Generate AUDIT.html

Build the report with the same structure as `/grade`, but with additional states:

- **PASS** (checkmark): Feature verified on first screenshot
- **FIXED** (wrench): Feature failed initially, fix applied, re-verified with proof. Shows both original and fixed screenshots.
- **FAIL** (X): Feature failed, fix attempted but didn't resolve. Shows original screenshot + notes on what was tried.

Each FIXED element's `<details>` block should show:
```
ORIGINAL (failed):  [screenshot]
Analysis: "Button text invisible — white on white"
Diagnosis: "Stale inline style overriding CSS :hover"
Fix applied: "Cleared inline background/border-color properties"
AFTER FIX:  [screenshot]
Analysis: "Button now shows frosted glass on hover with visible light text"
```

## Phase 4: Cleanup

Same as `/grade` — kill receiver, remove temp files.

## Key Differences from /grade

| | /grade | /grade-and-fix |
|---|---|---|
| Fails | Noted, reported | Diagnosed, fixed, re-verified |
| Diagnosis | Not attempted | /hypothesize with 3 agents |
| Code changes | None | Yes, minimal targeted fixes |
| Report states | PASS, FAIL | PASS, FIXED, FAIL |
| Screenshot count | 1 per element | 1 for PASS, 2 for FIXED (before + after fix) |

## Rules

All `/grade` rules apply, plus:
- **One fix attempt per failure.** Don't loop. If it doesn't work, leave it for the user.
- **Minimal fixes only.** Fix the specific problem. Don't refactor, don't add features, don't clean up surrounding code.
- **The fix-proof screenshot is mandatory.** A FIXED verdict without a re-verification screenshot is not valid.
- **Never skip the /hypothesize step.** Even if the cause seems obvious, run the 3 agents. They might catch something you missed.
