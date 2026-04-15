---
name: uglybug
description: "Adversarial bug-hunt game. Six isolated agents — Hunter, Skeptic, Reproducer, Referee in-game + Consolidator, Hardener post-game — find real bugs through rebuttal, independent verification, and ruling. Counter-sycophancy by design. Use when the user says /uglybug, asks 'find bugs in this codebase', 'adversarial review', 'bug hunt', 'ugly bug', 'security audit', or wants to stress-test a project before shipping."
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, Agent, AskUserQuestion
argument-hint: "[scope] [--waves=N] [--theme=auth|db|api|...] [--no-consolidator] [--no-hardener] [--resume]"
---

# Uglybug

Run an adversarial bug-hunting game against the current project. Six agents (Hunter, Skeptic, Reproducer, Referee + post-game Consolidator, Hardener) cooperate through structured rebuttal to surface real bugs while keeping false positives low.

**Design principle**: single-agent code review is vulnerable to sycophancy bias (published data shows up to 88% of known vulnerabilities are dismissed when code is framed as "bug-free"). This skill architecturally counters that by forcing every finding through a Skeptic with asymmetric skin-in-the-game, an independent Reproducer that verifies the code, and a Referee that renders the final call.

## Arguments

Parse `$ARGUMENTS` for:

- **Positional (optional)**: initial scope description (e.g., `"auth flow"`, `"supabase migrations"`, a directory path). If omitted, ask the user.
- **`--waves=N`**: max waves to run (default 5). Clamped to [1, 10].
- **`--theme=<topic>`**: restrict the wave to one surface area (e.g., `auth`, `db`, `api`, `payments`, `realtime`). If omitted, Hunter picks surface freely.
- **`--no-consolidator`**: skip post-game pattern synthesis.
- **`--no-hardener`**: skip post-game test/guardrail proposals.
- **`--resume`**: continue from the last saved wave in `.uglybug/state.json`.
- **`--stop`**: end an in-progress game early and proceed to post-game.

## Step 1: Confirm scope and rules

Read `.uglybug/config.json` if it exists (per-project defaults).

Present the rules briefly to the user and confirm before starting. Use `AskUserQuestion` to gather:

1. **Scope**: which files / directories are in bounds? (offer: entire repo, `app/` only, `src/` only, specific path from positional arg)
2. **Max waves**: default 5.
3. **Theme per wave**: is the user driving theme selection per wave, or letting Hunter free-roam? Default: free-roam.
4. **Post-game**: run Consolidator + Hardener after? Default: yes to both.

Show the scoring table (from `references/scoring.md`) and the 6-agent flow, then ask: "Ready to start?"

## Step 2: Initialize state

Create `.uglybug/` directory in the project root (git-ignored by default — add to `.gitignore` if not already, silently).

Initialize `.uglybug/state.json`:

```json
{
  "started_at": "<ISO timestamp>",
  "scope": "...",
  "max_waves": 5,
  "waves_run": 0,
  "scoreboard": {
    "attacker_points": 0,
    "skeptic_points": 0
  },
  "findings": [],
  "observations": [],
  "verdicts": [],
  "prior_ids": [],
  "stop_reason": null
}
```

Write to `.uglybug/wave-<n>-*.json` for each sub-agent's output (preserves context for Referee + Consolidator).

## Step 3: Wave loop

For each wave 1..N:

### 3a. Invoke Hunter

Read `references/roles/hunter.md` to build the prompt. Inject:
- The scope
- The machine-readable list of `prior_ids` + titles (for dedup check)
- The current wave number
- The theme, if set
- The heuristics menu from `references/heuristics.md`

Spawn via `Agent` tool with `subagent_type: general-purpose`. Expect a JSON array response.

Write output to `.uglybug/wave-<n>-hunter.json`.

### 3b. Check for `GAME_OVER` directive

If the Hunter's response includes `{"directive": "GAME_OVER", "reason": "..."}`, skip remaining wave steps and jump to post-game.

### 3c. Parallel: Skeptic + Reproducer

Run both simultaneously via separate `Agent` tool calls in one assistant turn:

- **Skeptic** gets: all FINDINGS from this wave, plus `references/roles/skeptic.md`.
- **Reproducer** gets: all FINDINGS from this wave, plus `references/roles/reproducer.md`, plus read access to the repo.

Write outputs to `.uglybug/wave-<n>-skeptic.json` and `.uglybug/wave-<n>-reproducer.json`.

### 3d. Invoke Referee

If Skeptic and Reproducer agree unanimously on every finding (all CONCEDE + TRUE, or all UPHOLD + FALSE), you may skip the Referee and render verdicts directly using the consensus (save tokens). Otherwise, always invoke Referee.

Referee gets: all prior artifacts for this wave + `references/roles/referee.md`.

Write output to `.uglybug/wave-<n>-referee.json`.

### 3e. Update state

- Append confirmed/downgraded findings to `state.findings`.
- Append observations (incl. clean surfaces) to `state.observations`.
- Append rejected findings to `state.rejected`.
- Append verdicts to `state.verdicts`.
- Update `state.scoreboard` from Referee's per-finding `attacker_points` and `skeptic_adjustment`.
- Append this wave's IDs to `state.prior_ids` (for next wave's dedup check).
- Increment `state.waves_run`.

### 3f. Print wave summary

Use the format from `references/scoring.md`:

```
🏴 WAVE <n> Results

| ID | Title | Claimed | Verdict | Points |
... (table)

Wave <n> score: +N points · X CONFIRMED, Y DOWNGRADED, Z REJECTED
Cumulative: +N | Attacker: +N / Skeptic: +N
```

### 3g. Check stop conditions

(See `references/scoring.md`.) If any triggered, record `stop_reason` in state and jump to post-game.

### 3h. Continue to next wave

If you have themes queued or the user's cadence requires confirmation between waves, use `AskUserQuestion`. Otherwise roll straight into wave N+1.

## Step 4: Post-game — Consolidator

If `--no-consolidator` not set, invoke Consolidator.

Consolidator gets: the full `state.json`, plus `references/roles/consolidator.md`.

Produces a markdown report — write to `.uglybug/report.md`.

## Step 5: Post-game — Hardener

If `--no-hardener` not set, invoke Hardener.

Hardener gets: the `state.json` + Consolidator report + `references/roles/hardener.md`.

Produces a JSON array of test + guardrail proposals — write to `.uglybug/prevention.json`.

Append a `## Prevention` section to `.uglybug/report.md` summarizing the Hardener's proposals.

## Step 6: Final output to user

Print the final scoreboard and path to the full report:

```
🏁 GAME OVER

Cumulative score: +N
Findings: X confirmed (Critical: A, High: B, Medium: C, Low: D)
Observations: E (including F clean-surface verifications)
FP rate: G%
Skeptic calibration: +H

Full report: .uglybug/report.md
Prevention proposals: .uglybug/prevention.json

Next steps:
  - Review .uglybug/report.md
  - Decide which prevention proposals to adopt
  - Fix findings (optionally via `/gsd-new-milestone` or similar workflow)
```

Offer (via `AskUserQuestion`): "Want me to open a fix milestone / issue tracker entry / PR draft for these?"

## Integration with GSD / planning workflows

If `.planning/` exists in the repo (GSD project), after the final output, ask:

> "Detected a GSD project. Create a Hardening milestone / phase from these findings?"

If yes, invoke `/gsd-new-milestone` or `/gsd-add-phase` with the findings structured as requirements.

## Error handling

- **Sub-agent returns non-JSON**: retry once with explicit "respond in strict JSON only" instruction. If second attempt fails, record the error in state and continue the wave with partial data.
- **File read errors during verification**: Reproducer flags `misquote: true` if cited lines don't exist. Referee penalizes accordingly.
- **State corruption / partial writes**: always write state to a temp file and rename atomically.

## Invoking individual agents

Advanced: a user may invoke a specific role against existing findings:

- `/uglybug --role=hardener`: run Hardener against existing `.uglybug/state.json`
- `/uglybug --role=consolidator`: run Consolidator against existing state
- `/uglybug --rerun-wave=3 --role=skeptic`: re-run Skeptic for wave 3's findings

This is useful when iterating on role prompts or after manual edits to findings.

## Configuration file

Optional `.uglybug/config.json` for project-level defaults:

```json
{
  "default_scope": ["app/", "lib/", "supabase/"],
  "default_max_waves": 5,
  "scope_excludes": [".next/", "node_modules/", "dist/"],
  "theme_rotation": ["auth", "db", "api", "realtime", "payments"],
  "always_run_consolidator": true,
  "always_run_hardener": true,
  "stop_on_zero_findings": true
}
```

## Notes on agent isolation

Each sub-agent runs in its own context (`Agent` tool's subagent_type). This is load-bearing — it prevents anchoring bias (Skeptic would rubber-stamp if they saw the Hunter's reasoning trace). Structure the prompts to pass only the strict JSON contracts between agents, never their reasoning prose.

## Notes on agent-to-agent referencing

Agents refer to each other by role name, not by tool call. The Hunter does NOT know which subagent_type the Skeptic runs as. All coordination is through the JSON contracts and the orchestrator (this skill).
