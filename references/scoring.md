# Scoring System

## Attacker (Hunter) score

| Severity  | Confirmed | False positive | Downgraded |
|-----------|-----------|----------------|------------|
| Critical  | +10       | −10            | credit at new tier |
| High      | +5        | −5             | credit at new tier |
| Medium    | +2        | −2             | credit at new tier |
| Low       | +1        | −1             | —          |
| OBSERVATION | 0 | 0 | — |

**Duplicates**: `REJECTED(duplicate)` → 0 points (no reward, no penalty). Duplicates are a process error, not a judgment error.

**Upgrades**: if the Referee decides the bug is MORE severe than claimed, the Hunter still gets the CLAIMED tier's points. This prevents defensive underclaiming. (Acknowledge the Hunter's correctness in the rationale but don't reward the hedge.)

**Negative findings** (verifying a surface is clean): +1 per explicitly-probed-and-reported-clean surface. These are marked as `OBSERVATION` with `kind: "clean_surface"` and never scored negative.

## Skeptic score

Asymmetric to offset sycophancy bias:

| Skeptic action | Referee outcome | Skeptic points |
|---|---|---|
| Correctly dismissed (UPHOLD / DOWNGRADE) | Matches Skeptic's call | +1 |
| **Wrongly dismissed real bug** (UPHOLD but Referee CONFIRMED at original severity) | Skeptic's rebuttal was invalid | **−2** |
| Correctly conceded a real bug | Matches Skeptic's CONCEDE | +1 |
| Wrongly conceded a false finding | REJECTED by Referee, Skeptic had conceded | 0 |

The 2x penalty on wrong dismissals is **load-bearing** — it forces the Skeptic to concede when the evidence is clear rather than rubber-stamp dismissals. Published research shows LLMs have up to an 88% sycophancy-dismissal rate when a change is framed as "bug-free" (see SycEval, arXiv 2502.08177). This scoring structure is the counter-measure.

## Reproducer

Not directly scored. Their `reproduced: TRUE/PARTIAL/FALSE` and `misquote_flag` feed the Referee's decision. Quality shows up indirectly — if Reproducer flags a misquote and the Hunter's finding is then rejected, that's a signal they're doing the job.

## Referee

Not scored. Their verdicts define the ground truth that everything else is scored against.

## Game-level calibration metrics

Tracked across all waves:

- **Attacker FP rate**: rejected-FP findings / total findings. Target: < 10%.
- **Skeptic dismissal accuracy**: (correct dismissals / total dismissals). Target: > 80%.
- **Reproducer misquote catch rate**: misquote_flags / findings with real misquotes. Hard to measure directly; if the Referee never overrules Reproducer, assume near-perfect.
- **Wave signal degradation**: are later waves still finding Medium+? If 2 consecutive waves produce only Low, the attack surface is exhausted → auto-stop.

## Stop conditions

The game auto-terminates when any of these are met:

1. **Max waves reached** (default 5, configurable via `--waves`).
2. **Two consecutive waves** with zero Critical/High findings and no rejected FPs.
3. **Zero findings** filed in a wave (including OBSERVATIONS).
4. **Attacker declared coverage exhausted** in their own report (they can emit a `GAME_OVER` directive in their JSON output).
5. **User manually stops** via `/uglybug stop`.

After stop, always run Consolidator + Hardener before producing the final report (unless `--no-consolidator` / `--no-hardener`).
