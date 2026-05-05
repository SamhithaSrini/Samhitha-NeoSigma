# AgentBench OS Project Tracker

## Current Goal

Integrate `auto-harness` with AgentBench OS and improve the AgentBench validation
score through the phased loop from `CLAUDE_CODE_BUILD_PLAN_v2.md`.

## Dataset Status

The cloned AgentBench checkout does not match the plan exactly:

- `data/dev.json` contains 26 tasks.
- The fuller OS standard set is defined by `configs/tasks/os.yaml` as nested files
  under `data/1` through `data/7`.
- Those nested files contain 144 unique standard tasks in this checkout.
- No official `data/test.json` is present locally.

To unblock local validation, we generated a deterministic pseudo-split from the
144 standard tasks:

- Train: 108 tasks
- Validation/test: 36 tasks
- Split seed: 42
- Local data path: `agentbench_local/os_interaction/data`

This is not the official hidden AgentBench test set. It is a local validation set
for development and regression tracking.

## Baseline

Baseline date: 2026-05-05

Configuration:

- Benchmark: `agentbench`
- Model: `gpt-4o`
- Train split: `train`
- Validation split: `test`
- Max concurrency: 5
- Data dir: `agentbench_local/os_interaction`

Baseline validation result:

- `val_score`: 0.4167
- Passed: 15 / 36

Original first baseline before harness answer-extraction correction:

- `val_score`: 0.3056
- Passed: 11 / 36

Recorded in:

- `workspace/results.tsv`
- `workspace/test_traces.json`

## Built So Far

- Cloned `neosigmaai/auto-harness`.
- Cloned `THUDM/AgentBench`.
- Built AgentBench Docker images:
  - `local-os/default`
  - `local-os/packages`
  - `local-os/ubuntu`
- Added `AgentBenchRunner` to `benchmark.py`.
- Added AgentBench template agent:
  - `agent/templates/agentbench.py`
  - copied to `agent/agent.py`
- Added Phase 2/4 scaffolding:
  - `classifier.py`
  - `success_store.py`
- Wired AgentBench into:
  - `prepare.py`
  - `gating.py`
  - `experiment_config.yaml.template`
  - `docker-compose.yml`
  - `.env.example`
  - `PROGRAM.md` templates
- Fixed answer extraction so `ANSWER:` can appear after short rationale text.

## Current Phase

Phase 1: baseline and manual failure analysis.

Next gate to unlock Phase 2:

- Run corrected train split.
- Read 10-15 train failure traces manually.
- Identify repeated failure modes.
- Make one focused improvement to `agent/agent.py`.
- Re-run a small task subset before a full validation gate.

## Iteration Log

### Iteration 0 - Baseline

Result: `val_score=0.4167` on local validation split.

Observation:

- Generalist agent can solve some direct file/count tasks.
- Early failures are likely from command choice, stopping behavior, hidden-file
  search depth, and answer-format discipline.

Next action:

- Generate train traces from the corrected 108-task train split.
- Analyze failures from `workspace/train_traces.json`.

Train run result:

- Train score: 0.4537
- Passed: 49 / 108
- Trace file: `workspace/train_traces.json`

Validation baseline remains:

- `val_score`: 0.4167
- Passed: 15 / 36

Infrastructure correction:

- Fixed answer extraction in `benchmark.py` to ignore `ANSWER:` markers inside
  fenced code blocks.
- This prevented the runner from stopping before executing commands like
  `echo "ANSWER: $total"`.
- Corrected validation baseline increased from 11/36 to 15/36.
- Corrected train score increased from 45/108 to 49/108.

### Iteration 1 - Side-Effect / Session Handling

Hypothesis:

- Many failures came from side-effect tasks where the agent described or answered
  before actually changing the container.
- Some failures were also caused by the runner not preserving AgentBench session
  context from `start` commands such as `cd /home/user2` and `su - jack`.

Changes:

- Updated `agent/agent.py` with explicit side-effect rules:
  create/move/chmod/edit PATH before answering, write scripts with here-docs,
  and inspect existing data instead of inventing sample data.
- Updated `benchmark.py` so bash blocks execute before accepting an `ANSWER:`
  in the same response.
- Added lightweight session context tracking for working directory and `su - user`
  in `AgentBenchRunner`.
- Fixed CLI result bookkeeping so `--split test` writes `workspace/test_results.json`
  instead of overwriting `workspace/train_results.json`.

Focused subset:

- `std-004-Q47-0`: fail -> pass
- `std-006-new-7`: fail -> pass
- `std-004-N4-0`, `std-006-new-0`, `std-006-new-1`: still fail

Validation result:

- `val_score`: 0.4167
- Passed: 15 / 36
- Net validation score change: flat

Decision:

- Keep the runner fixes.
- Treat the prompt change as provisional; continue with classifier/manual failure
  analysis before making another prompt change.

Fresh train run after fixes:

- Train score: 0.5185
- Passed: 56 / 108

Classifier result:

- Classified 52 train failures.
- Failure type counts:
  - `incomplete_execution`: 23
  - `wrong_command`: 12
  - `wrong_interpretation`: 10
  - `env_interaction_failure`: 4
  - `format_error`: 2
  - `output_formatting`: 1
- Affected lever counts:
  - `tool_use_strategy`: 45
  - `output_formatting`: 5
  - `context_construction`: 2

Next target:

- Tool-use strategy for incomplete execution: finish the command/script/test loop
  instead of stopping after partial setup or repeated empty output.

### Iteration 2 - No-Action / Empty-Output Feedback

Hypothesis:

- A major `incomplete_execution` pattern came from agent turns with neither a
  bash command nor an `ANSWER`, especially after setup-like responses.
- Empty command output was ambiguous, causing repeated or stalled behavior.

Change:

- Updated `AgentBenchRunner` to inject corrective feedback when a turn contains
  no executable bash block and no `ANSWER`.
- Updated command feedback so truly empty stdout/stderr becomes
  `The command produced no output.`

Focused subset:

- 1 / 6 previously incomplete-execution examples passed.

Validation result:

- `val_score`: 0.5000
- Passed: 18 / 36
- Change from corrected baseline: +0.0833
- Change from previous validation run: +0.0833

Decision:

- Keep this runner change. It targets the dominant classifier category and
  improved local validation.

Fresh train run after Iteration 2:

- Train score: 0.5185
- Passed: 56 / 108
- Trace file refreshed: `workspace/train_traces.json`
- Refreshed `workspace/failure_taxonomy.json` for the current 52 train failures.
- Fixed `classifier.py` so passing tasks from the current split are removed from
  the active taxonomy instead of lingering from older runs.

### Iteration 3 - Stock-Log Parsing and Unresolved Answers

Hypothesis:

- Several failures came from recurring stock-log interpretation mistakes:
  treating `head` output as complete evidence, using fragile pipe delimiters,
  confusing "times" with summed share counts, and using `index` as an awk loop
  variable.
- Some failures came from unresolved shell variables being accepted as final
  answers, for example `ANSWER: $total_lines`.

Changes:

- Updated `agent/agent.py` with explicit stock-log rules:
  - use the full file after inspecting the format with `head`;
  - parse `name | action | stock_index | count` with `awk -F' *\\| *'`;
  - count rows for "how many times";
  - sum the count column for "total number of stocks";
  - count distinct stock-index values for "number of types";
  - avoid `index` as an awk variable.
- Updated `benchmark.py` so unresolved shell variables in `ANSWER:` are ignored
  and the agent gets another turn.

Focused subset:

- Stock-log subset: 4 / 4 passed after the prompt refinement.
- Earlier 7-task mixed subset exposed two line-count tasks where the local
  AgentBench reference command double-counts `wc` total rows; we did not teach
  the agent that incorrect behavior.

Validation result:

- `val_score`: 0.5833
- Passed: 21 / 36
- Change from corrected baseline: +0.1666
- Change from previous validation run: +0.0833

Decision:

- Keep the prompt and unresolved-answer extraction changes.
- Next target: inspect the remaining 15 validation failures for another narrow
  pattern that can improve score without overfitting to known-bug check scripts.

### Iteration 4 - Session Env and Multi-Block Execution

Hypothesis:

- AgentBench init scripts sometimes set environment variables such as
  `TARGET_DIR`, but separate `docker exec` calls were losing them.
- The runner only executed the first bash block in an agent response, which left
  "create script, then test it" responses half-finished.

Changes:

- Updated `AgentBenchRunner` to persist simple exported environment variables
  across init, agent, and evaluation commands.
- Passed the persisted environment into check/example scripts so reference
  commands that depend on exported variables score correctly.
- Updated bash extraction to execute all fenced bash/sh blocks in a response in
  order inside one shell command.

Focused subset:

- `std-007-bootstrap-8`: fail -> pass after environment persistence.
- `std-005-new-2`, `std-007-bootstrap-25`, `std-007-bootstrap-84`: 3 / 3 passed
  after multi-block execution.

Validation result:

- `val_score`: 0.6111
- Passed: 22 / 36
- Change from corrected baseline: +0.1944
- Change from previous best: +0.0278

Decision:

- Keep both harness changes.
- Next target: remaining failures are mostly command semantics and checker quirks,
  especially hidden-file filtering, line-count variants, and system-stat tasks.
