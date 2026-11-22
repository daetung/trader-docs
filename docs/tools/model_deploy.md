# Tool: Model Deploy

**File:** `tools/model_deploy.py`
**Standalone CLI tool — a per-chunk guest run through guest_supervisor.md's `run_guest`; it opens no work**

---

## Role

The operator's approval surface for model deployment. PipelineOptimizer's
'exploitation' and 'refit' closing steps write candidates to
`model_candidates` (pipeline_optimizer.md's Model Candidate); this tool
approves, overrides, rejects or rolls back, writing `model_deployments`
(db_schema.md). LiveModeRunner reads the deployed model — the run_id of
`model_deployments`' latest row — at its session load (live_mode_runner.md),
so every decision takes effect at the next session start, under the current
stage (shadow_retraining.md's Promotion).

**DB access role**: a per-chunk guest that opens no work (guest_supervisor.md's
Tool Registry; db_coordination.md's Unit scope):
- run_guest skips New-Run Resolution and work opening;
- `--trigger scheduled`, `--supersede` and `--resume-work` are errors at
  start;
- a self-terminated run ends without a relaunch;
- every unit is opened with `timeout` 0; a unit not admitted names the
  reason — the runner window, an owner active, the guard minutes:
  interactive mode offers a retry, a subcommand exits with an error;
- each read and each decision runs in its own unit; no unit is held while
  waiting for input;
- a decision re-validates its preconditions in its write unit and reports a
  changed state instead of writing;
- Ctrl-C at any prompt exits with nothing written;
- the body calls neither `register()` nor `guest_unit()`.

---

## CLI Usage

```bash
run_guest model_deploy                                  # interactive
run_guest model_deploy show
run_guest model_deploy approve <final_run_id> [--note <text>]
run_guest model_deploy approve <final_run_id> --override --note <text>
run_guest model_deploy reject <final_run_id> [--note <text>]
run_guest model_deploy rollback [<run_id>] [--note <text>]
run_guest model_deploy rollback [<run_id>] --override --note <text>
# python tools/model_deploy.py <args> hands over to run_guest model_deploy <args>
```

- With no subcommand and a non-TTY stdin it exits with an error naming the
  subcommands.
- `show` prints the Main view (below) without its menu.
- Interactive mode and the subcommands share one implementation per
  decision.

---

## Target Checks

Applied by approve and rollback to the run_id a decision deploys. A decision
refuses when:
- the run_id's artifact files are missing — meta.json or a model file
  (base_model.md);
- `utils.deployability_checks()` fails against the current config file and
  lock, each failed check named;
- `utils.regime_gate()` reports blocked for the run_id's
  `regime_holdout_verdict` row under the current config file, its verdict
  named, or that no row exists.

`--override`, its note required, accepts a deployability or regime-gate
refusal, never missing artifacts; the decision then shows the gate that will
block the model at the next session start — Gate 1d or Step 7's regime
holdout gate (live_mode_runner.md). The data-area attestation (utils.md's
Code Versions) is shown as a warning.

**θ at a decision.** With `execution.late_entry_enabled` or
`execution.entry_price_cap_enabled` true and
`execution.late_entry_theta_run_id` differing from the target run_id, approve
and rollback warn that θ will not apply to the model until set
(execution_common.md's θ binding), showing:
- its candidate θ values (`model_candidates.late_entry_theta`);
- its live θ values or count, for a run_id with sessions
  (`utils.live_late_entry_theta()`);
- the cap's effect under `live_mode.entry_price_cap_inert`.

---

## Decisions

Every decision shows what it writes and that it takes effect at the next
session start, and asks y/N.

**approve `<final_run_id>`** refuses when:
- the candidate is not 'awaiting_approval';
- its champion_run_id differs from the current deployed run_id — a NULL
  champion_run_id standing for no deployment — the candidate then turning
  'superseded';
- the Target Checks refuse.
It writes a `model_deployments` row, action 'approve', and sets the candidate
'approved'.

**approve `<final_run_id>` --override**, its note required: accepts a
'rejected' candidate, and a deployability or regime-gate refusal of the
Target Checks; writes action 'override' and sets the candidate 'approved'.

**reject `<final_run_id>`** refuses unless the candidate is
'awaiting_approval'; it sets the candidate 'rejected' with its note.

**rollback** — the deployment stack: `model_deployments` rows taken in
deployed_at order:
- an 'approve' or 'override' row pushes its run_id;
- a 'rollback' row whose run_id is in the stack removes every entry above
  that run_id's topmost entry;
- a 'rollback' row whose run_id is not in the stack pushes it.
Without a run_id, the target is the entry below the stack's top; with none,
rollback refuses. With a run_id, the target is that run_id; rollback refuses
when it is the deployed run_id or no `model_deployments` row names it. The
Target Checks apply to the target; it writes a `model_deployments` row,
action 'rollback', with or without `--override`.

approve and rollback turn every 'awaiting_approval' candidate whose
champion_run_id is no longer the deployed run_id 'superseded', in the same
transaction — a NULL champion_run_id standing for no deployment.

**Status time.** `decided_at` is set at every status change after writing —
approve, approve --override and reject, and each supersede; `decision_note`
holds the operator's note, NULL on a supersede. A candidate's status time is
its decided_at, or its created_at when the status is unchanged since writing.

---

## Interactive Views

Line prompts, without curses.

**Main:**
- the deployed model — run_id, phase, deployed_at, action, note, probation
  state and latest result (shadow_retraining.md's Probation after a
  deployment);
- the model of a session in progress when it differs, marked as taking
  effect next session;
- the `quarantine.data_caveats` entries, when any (migration_tool.md);
- the awaiting candidates — final_run_id, phase, created_at, regime verdict,
  comparison verdict, Δ, CI;
- the counts of rejected and superseded candidates since the latest
  deployment, by status time;
- a menu: candidate detail; all candidates; history and rollback; refresh;
  quit.

**Candidate detail:**
- the regime report items;
- the comparison detail;
- the Target Checks' results;
- the data-area attestation;
- whether its champion is stale;
- its candidate θ values;
- actions: approve; reject; back.

**All candidates:** each candidate with its status; a candidate written
'rejected' shows its causes — the regime gate when regime_gate_blocked is
true, the comparison when its verdict is 'fail'; a candidate rejected by the
operator's reject shows that decision and its note; a candidate written
'superseded' shows its champion_run_id and the deployed run_id. A rejected
candidate offers override, its note required.

**History:** deployments newest first, the stack's rollback target marked.
rollback takes the marked target or a listed run_id other than the deployed
one, chosen by the operator; the chosen target's check results are shown
before y/N.

---

## Constraints

- The only writer of `model_deployments`; rows are only appended
  (db_schema.md)
- A decision takes effect at the next session start — LiveModeRunner reads
  the deployed model once, at its session load
- Units are opened with `timeout` 0, one per read or decision; none is held
  across a prompt
- A decision re-validates in its write unit; a changed state is reported,
  nothing written
- Calls neither `register()` nor `guest_unit()` — the body opens its units
  through the unit opener `run_guest` hands it
