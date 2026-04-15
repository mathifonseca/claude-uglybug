# Finding Schema

All agents communicate via strict JSON. This is the authoritative schema.

## Finding

```json
{
  "kind": "FINDING",
  "id": "W1-01",
  "severity": "Critical" | "High" | "Medium" | "Low",
  "confidence": 0-100,
  "title": "string (one line)",
  "location": "path/to/file.ext:LINE" | "path/to/file.ext:LINE-LINE",
  "trigger": "string",
  "expected": "string",
  "actual": "string",
  "evidence": "string (verbatim quoted lines, 2-10 lines)",
  "exploit_poc": "string (required for Critical; optional otherwise)",
  "claim": "string (why it's a bug at this severity)",
  "upgrade_of": "W<n>-<nn>" | null,
  "tags": ["auth", "rls", "migration", ...]
}
```

## Observation

```json
{
  "kind": "OBSERVATION",
  "id": "W1-OBS-01",
  "title": "string",
  "location": "path/to/file.ext:LINE",
  "concern": "string (what worried you)",
  "why_not_finding": "string (e.g., 'no concrete trigger', 'scope-limited by RLS', 'cosmetic')",
  "severity_if_real": "Critical" | "High" | "Medium" | "Low" | null,
  "tags": ["..."]
}
```

## Clean surface report (negative finding)

```json
{
  "kind": "OBSERVATION",
  "id": "W<n>-CLEAN-<nn>",
  "subkind": "clean_surface",
  "title": "Surface verified clean: <name>",
  "location": "file or pattern that was probed",
  "what_was_checked": "string (concrete steps taken)",
  "verdict": "clean",
  "tags": ["..."]
}
```

Negative findings score +1 each and signal coverage breadth.

## Skeptic response

```json
{
  "id": "W<n>-<nn>",
  "verdict_hint": "CONCEDE" | "UPHOLD" | "DOWNGRADE(<severity>)" | "REJECT(<reason>)",
  "defense": "1-3 paragraphs with file:line citations",
  "new_information": "optional amplification or compensating control Hunter missed"
}
```

## Reproducer response

```json
{
  "id": "W<n>-<nn>",
  "reproduced": "TRUE" | "PARTIAL" | "FALSE",
  "evidence": "2-8 lines of actual code with correct line numbers",
  "misquote_flag": true | false,
  "notes": "severity considerations, compensating controls, amplifiers"
}
```

## Referee verdict

```json
{
  "id": "W<n>-<nn>",
  "verdict": "CONFIRMED" | "DOWNGRADED(<severity>)" | "REJECTED" | "REJECTED(duplicate)",
  "rationale": "2-4 sentences",
  "attacker_points": <int>,
  "skeptic_adjustment": <int>,
  "original_claim_severity": "Critical" | "High" | "Medium" | "Low",
  "final_severity": "Critical" | "High" | "Medium" | "Low" | null
}
```

## Wave tally

```
WAVE <n>: Attacker +X / Skeptic +Y | Confirmed A, Downgraded B, Rejected C, Duplicates D | Clean surfaces: E
```

## Consolidator report

Markdown, not JSON. Shape defined in `references/roles/consolidator.md`.

## Hardener proposals

JSON array per `references/roles/hardener.md`.
