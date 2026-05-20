# axum-agents-api-review: Reviewer Anti-Patterns

These are mistakes the REVIEWER agent makes while running this skill. They are
not Axum code defects (those are the 16 anti-patterns in `methods.md`). Each
entry is a real failure mode of the review process, why it fails, and the
correct behavior.

## 1. Teaching the fix inline instead of delegating

### Wrong

```
[BLOCKER] New database pool created per request
  Location : src/main.rs:46
  Fix      : Move PgPoolOptions::new().max_connections(5)
             .acquire_timeout(Duration::from_secs(3)).connect(&url) into
             main, then .with_state(pool), and read it with
             State(pool): State<PgPool>. Also add ...
```

### Why it fails

The corrective code is duplicated from `axum-impl-database`. The owning skill
is the single source of the fix. A copy in a review finding drifts out of date
the moment the owning skill changes, and it bloats the review past the point of
being scannable. This skill DETECTS and ROUTES; it does not re-teach.

### Correct

One corrective sentence plus the `Fix skill` name. The user opens that skill
for the full fix:

```
[BLOCKER] New database pool created per request
  Location : src/main.rs:46
  Symptom  : PgPool::connect runs inside the handler, opening a fresh pool
             per request and exhausting the server connection limit.
  Anti-pattern : AP-12
  Fix skill : axum-impl-database
  Action   : build one pool in main and share it via with_state.
```

## 2. Inventing a severity outside the three levels

### Wrong

```
[CRITICAL] Hardcoded JWT secret in source
[INFO] Extension used where State would be compile-checked
[NIT] Missing #[instrument] on the handler
```

### Why it fails

`critical`, `info`, and `nit` are not severities this skill defines. The
severity model has exactly three levels: `blocker`, `warning`, `suggestion`.
A consumer of the review (a human or a tool) expects exactly those three and
cannot sort or filter an unknown level.

### Correct

Map every finding to one of `{blocker, warning, suggestion}`. A hardcoded
secret is a `blocker`. An `Extension` versus `State` divergence is a
`suggestion`. NEVER emit any other word in the severity slot.

## 3. Skipping the summary header

### Wrong

```
[BLOCKER] 0.7 path syntax on Axum 0.8
  Location : src/main.rs:27
  ...
```

The output jumps straight to the first finding.

### Why it fails

Without the summary header the reader does not know how many findings exist,
the blocker count (the single number that decides whether code can ship), or
which Axum version the review assumed. A reviewer who skips the header has
produced an output that cannot be triaged at a glance.

### Correct

ALWAYS emit the header first:

```
Axum API Review : 9 findings
  blockers : 4   warnings : 4   suggestions : 1
  Target Axum version : 0.8
```

## 4. Emitting findings out of severity order

### Wrong

```
[SUGGESTION] Extension used where State would be compile-checked
[BLOCKER] Hardcoded JWT secret in source
[WARNING] Missing graceful shutdown
[BLOCKER] Body extractor not in final position
```

### Why it fails

A reader scanning for ship-stoppers must read the entire output to be sure no
blocker is buried below a suggestion. Interleaved severities defeat the purpose
of a triage-ordered report.

### Correct

Order strictly: all blockers, then all warnings, then all suggestions. Within a
severity, order by file then by line. The reader sees every blocker before any
warning.

## 5. Reporting a finding not anchored to a check

### Wrong

```
[SUGGESTION] This handler name could be more descriptive
[WARNING] The author should add more unit tests here
```

### Why it fails

Neither finding ties to any of the 10 checklist categories or any of the 16
anti-patterns. This skill audits Axum-specific correctness, reliability, and
security. General style opinions and test-coverage notes are out of scope, and
emitting them dilutes the signal of the real findings. The determinism rule:
the agent NEVER reports a finding it cannot tie to a specific check.

### Correct

Report ONLY findings that map to a checklist category or an `AP-<n>` number.
If a concern fits no check, it is out of scope for this skill: omit it.

## 6. Rounding severity down when uncertain

### Wrong

A `std::sync::MutexGuard` held across `.await`: the reviewer is unsure whether
it compiles, so labels it a `warning` to be cautious.

### Why it fails

A `!Send` future fails the `Handler` trait bound at compile time; it is a
`blocker`. Labeling a possible blocker as a warning hides a ship-stopper. A
false blocker costs one review comment; a missed blocker costs an outage.

### Correct

When uncertain between two severities, ALWAYS choose the HIGHER one. Uncertain
between `blocker` and `warning` means `blocker`.

## 7. A silent empty result

### Wrong

The review finds nothing, so the agent emits nothing, or emits only a bare line
like `Looks fine`.

### Why it fails

An empty output is ambiguous: the reader cannot tell a clean review from a
review that never ran or crashed. There is no count, no version, no proof the
checklist was applied.

### Correct

A zero-finding review STILL emits the full summary header with all counts at 0
and an explicit `No issues found.` line:

```
Axum API Review : 0 findings
  blockers : 0   warnings : 0   suggestions : 0
  Target Axum version : 0.8

No issues found.
```

## 8. Citing a wrong or nonexistent owning skill

### Wrong

```
  Fix skill : axum-database
  Fix skill : axum-impl-db-pool
  Fix skill : axum-errors
```

### Why it fails

None of those names exist in the package. A user cannot open a skill that does
not exist, so the finding routes nowhere and the delegation breaks. The fix
knowledge becomes unreachable.

### Correct

Cite the EXACT package skill name. The database fix skill is `axum-impl-database`.
The error-handling fix skill is `axum-errors-handling`. The complete list of
the 28 valid names is the cross-reference map in `SKILL.md`. Use a name from
that map verbatim, or the finding cannot be acted on.
