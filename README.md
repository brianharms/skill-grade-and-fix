# /grade-and-fix

> ## ⚠️ Before you start
>
> Your AI agent can install this skill (copy into `~/.claude/skills/grade-and-fix/`, then restart Claude Code to load it). A few things need **you** or your environment:
>
> - **Also install the `/grade` and `/hypothesize` skills** — this skill builds on both (clone commands are in the Install section).
> - A **browser-screenshot capability** + a running app, same as `/grade`.


> Runs `/grade`, then auto-diagnoses and repairs on failure.

Part of [**vibekit**](https://ritual.industries) — a showcase of small tools and Claude Code skills that make coding with AI feel better.

## What it does

`/grade-and-fix` is a self-healing version of visual verification. It does everything `/grade` does — walk through the features in your most recent plan, screenshot each one, and analyze the image to render an honest PASS/FAIL verdict — but it doesn't stop at "this is broken."

When a feature **fails**, the skill spawns `/hypothesize` to diagnose the root cause with three parallel agents, implements the top-ranked fix, reloads the page (cache-busted), and takes a fresh screenshot to prove the repair worked. The result is a single `AUDIT.html` report where every feature is **PASS**, **FIXED** (with before/after screenshots), or **FAIL** (with notes on what was tried). No silent breakage, no hand-waving — just visual proof.

It is deliberately conservative: **one fix attempt per failure**. If the fix doesn't land, the feature stays marked FAIL for you to review rather than spiraling into an automated rewrite loop.

## Install

Clone (or download) this repo and copy the skill folder into your Claude Code skills directory so that `~/.claude/skills/grade-and-fix/SKILL.md` exists:

```bash
git clone https://github.com/brianharms/skill-grade-and-fix.git
mkdir -p ~/.claude/skills
cp -R skill-grade-and-fix ~/.claude/skills/grade-and-fix
```

That's the whole install — the skill is a single `SKILL.md` with no companion scripts to run.

This skill depends on two sibling skills. Install them the same way:

```bash
# /grade — the underlying grading protocol
git clone https://github.com/brianharms/skill-grade.git
cp -R skill-grade ~/.claude/skills/grade

# /hypothesize — parallel-agent diagnosis
git clone https://github.com/brianharms/skill-hypothesize.git
cp -R skill-hypothesize ~/.claude/skills/hypothesize
```

Once all three folders are in `~/.claude/skills/`, invoke the skill from any Claude Code session by typing:

```
/grade-and-fix
```

## Usage

Run a verification-and-repair pass over your most recent plan:

```
/grade-and-fix
```

Scope it explicitly:

```
/grade-and-fix all          # grade and fix everything from the session
/grade-and-fix "the new settings panel"
```

What happens:

1. **Grade pass** — the skill enumerates the features from your latest plan, navigates to each, takes a screenshot, and analyzes the image. You'll see progress like `Graded 4/15`. Every feature gets a screenshot — no exceptions.
2. **Fix loop** — for each FAIL, it runs `/hypothesize` (three agents diagnose independently and return ranked hypotheses), implements the top hypothesis as a minimal targeted fix, reloads with a cache-bust, and re-screenshots to confirm.
3. **Report** — it writes `AUDIT.html` with three states:
   - **PASS** — verified on the first screenshot.
   - **FIXED** — failed initially, repaired, and re-verified. The report shows the original failing screenshot *and* the fix-proof screenshot, plus the diagnosis and the fix that was applied.
   - **FAIL** — failed, one fix attempted, still broken. The report shows the original screenshot and notes on what was tried, so you can take it from there.
4. **Cleanup** — temp files are removed and the screenshot receiver is shut down.

## Requirements / Dependencies

- **Claude Code CLI** — this is a Claude Code skill; it runs inside a Claude Code session.
- **Sibling skills (required):**
  - [`/grade`](https://github.com/brianharms/skill-grade) — provides the underlying grading protocol and anti-abandonment rules that this skill builds on.
  - [`/hypothesize`](https://github.com/brianharms/skill-hypothesize) — provides the three-agent parallel diagnosis used in the fix loop.
- **Browser screenshots** — verification works by navigating to your running app and capturing real screenshots, so you need a browser the skill can drive (e.g. the Claude-in-Chrome MCP) and a locally running app/dev server to point it at.

## For AI coding agents

If you're an agent working *on* this skill, here's the lay of the land.

**Repo layout:**

```
skill-grade-and-fix/
├── SKILL.md      # the entire skill — this is the contract
├── LICENSE       # MIT
├── README.md     # this file
└── .gitignore
```

**`SKILL.md` is the whole thing.** There are no scripts, no `web/`, no `swift/`, no `template.html` — the skill is pure instructions. The YAML frontmatter (`name`, `description`, `user-invocable`, `arguments`) is the contract Claude Code reads to register and trigger the skill. The `description` field is also the trigger surface: it lists the phrases that activate the skill (`/grade-and-fix`, "grade and fix", "verify and fix"). Edit it carefully — changing those phrases changes when the skill fires.

**How to test a change:** copy the folder into `~/.claude/skills/grade-and-fix/` (see Install), make sure the two sibling skills are also installed, then start a Claude Code session against a project that has a recent plan and a runnable app, and invoke `/grade-and-fix`. Confirm it produces an `AUDIT.html` with the expected PASS / FIXED / FAIL states.

**Invariants — do not break these:**

- **The skill is a composition, not a standalone.** It explicitly delegates to `/grade` (Phase 1, grading) and `/hypothesize` (Phase 2, diagnosis). Don't inline or fork their logic into this file — keep the delegation so the three skills stay in sync. If you change behavior that belongs to grading or diagnosis, change it in those skills.
- **One fix attempt per failure.** The skill must never loop on a single failure. If a fix doesn't resolve a FAIL, it stays FAIL for the user — do not add a retry loop.
- **The fix-proof screenshot is mandatory.** A FIXED verdict is only valid with a re-verification screenshot taken *after* the fix. Never let a FIXED state ship without before/after proof.
- **Never skip `/hypothesize`.** Even when the cause looks obvious, the skill runs the three diagnostic agents. Don't add an "obvious fix" shortcut that bypasses diagnosis.
- **Fixes stay minimal.** The fix step repairs the specific problem only — no refactors, no new features, no cleanup of surrounding code. Preserve that constraint in any edit to the fix-loop instructions.
- **Keep the three report states.** `AUDIT.html` distinguishes PASS, FIXED, and FAIL. The `/grade`-only world has just PASS and FAIL; the FIXED state (and its dual-screenshot `<details>` block) is what makes this skill distinct. Don't collapse it back.

## License

MIT © 2026 Brian Harms / Ritual Industries — [ritual.industries](https://ritual.industries)
