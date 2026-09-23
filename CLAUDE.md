# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.


## 0. Hard Constraints

**Use Opus. Never pick Sonnet yourself.** The default model is pinned to Opus for this repo. When you spawn subagents, let them inherit the parent model — do not pass a `model` override. The one exception: if a `superpowers` skill explicitly directs a specific model for a step, follow the skill. Absent that explicit direction, Opus.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.
- If you notice an unrelated bug (in code review, debugging, or otherwise), don't fix it silently - flag it to the user and log it to `BUGS.md` with enough context (location, problem, impact, how found) to pick up later.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

## 5. Maintain Documentation
**Keep docs accurate and up-to-date.**
- If you change behavior, update comments and README.
- If you add features, document them.
- If you remove features, remove docs.
- If you fix bugs, update any related documentation.

## 6. Log Model

**State which model produced each response.** This makes the "Use Opus" rule in section 0 auditable instead of just assumed. End each response with a one-line tag, e.g. `[Model: claude-opus-5]`. If a subagent was spawned and it inherited the parent model, no separate tag is needed; if a skill directed a different model for a step, tag that step too.


## 6. Worklog and Changelog

### WORKLOG.md
Update `WORKLOG.md` at the end of each work session. Add a new session entry with:
- Session number (next in sequence), date, estimated duration
- Bullet points for significant changes (what was built, fixed, or refactored)
- Update the Summary table metrics (calendar days, commit counts, hours)
- Add milestone entries for notable events

### CHANGELOG.md
`CHANGELOG.md` follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format and is generated from [Conventional Commits](https://www.conventionalcommits.org/).

**Commit message format:**
```
feat: add network anomaly detection role
fix: correct VirusTotal quota handling
chore: update ansible.utils collection version
refactor: consolidate MalwareBazaar lookup tasks
docs: add README for executable_scanner role
```

**Regenerate the changelog after tagging a release:**
```bash
source .venv/bin/activate
git tag v0.2.0
git-cliff -o CHANGELOG.md
```

Move items from `[Unreleased]` to the new version section when releasing. Do not manually edit entries below `[Unreleased]` — git-cliff owns those.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.

## Ansible Development

Ansible style and conventions are defined in the `ansible-style` project skill (`.claude/skills/ansible-style/`), which loads automatically when editing files under `ansible/` or `conversion-host/`.
