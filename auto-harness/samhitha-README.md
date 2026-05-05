# Samhitha README

This file documents what was added or changed on top of the upstream repos:

- `neosigmaai/auto-harness`
- `THUDM/AgentBench`

## Repository Setup

Both repos were cloned side by side under:

```text
/Users/samhitha/mygalaxy/Recruiting/NeoSigma/
```

Layout:

```text
NeoSigma/
  auto-harness/
  AgentBench/
```

All implementation work happens in `auto-harness/`. The `AgentBench/` checkout is
used as a read-only source for OS task data, scripts, and Dockerfiles.

## Added AgentBench Support

The upstream `auto-harness` repo did not have AgentBench OS support. We added it.

Changed or added files:

- `benchmark.py`
  - Added `AgentBenchRunner`.
  - Loads AgentBench OS task JSON.
  - Runs each task in a Docker container.
  - Executes bash blocks from `agent/agent.py`.
  - Scores answers with `evaluation.match` or AgentBench check scripts.
  - Writes split-specific traces and results.
  - Added session-context handling for AgentBench-style `cd ...` and `su - user`.

- `prepare.py`
  - Added `agentbench` benchmark branch.
  - Validates Docker and required local OS images.
  - Copies the AgentBench agent template.
  - Initializes AgentBench workspace artifacts.

- `gating.py`
  - Added `AgentBenchRunner` wiring.
  - Added cluster regression check scaffolding.

- `agent/templates/agentbench.py`
  - New AgentBench OS baseline agent template.

- `agent/agent.py`
  - Current editable benchmark agent copied from `agent/templates/agentbench.py`.
  - Contains prompt rules for OS tasks, side-effect tasks, script creation, and data inspection.

- `classifier.py`
  - New failure classifier for `workspace/{split}_traces.json`.
  - Writes `workspace/failure_taxonomy.json`.

- `success_store.py`
  - New success trace clustering and few-shot retrieval utility.

- `program_templates/agentbench.md`
  - New AgentBench-specific optimization instructions.

- `program_templates/base.md`
  - Updated generic gate wording.

- `PROGRAM.md`
  - Generated from the base template plus AgentBench template.

## Docker Changes

Changed files:

- `Dockerfile`
  - Added Docker CLI support for AgentBench workflows.

- `docker-compose.yml`
  - Mounts sibling `../AgentBench`.
  - Mounts Docker socket.
  - Mounts new AgentBench helper files.
  - Adds AgentBench environment variables.

- `.env.example`
  - Added AgentBench environment variables.

Local Docker images built from AgentBench:

```text
local-os/default
local-os/packages
local-os/ubuntu
```

## Config Changes

Changed files:

- `experiment_config.yaml.template`
  - Added an AgentBench OS config example.

Local ignored config:

- `.env`
  - Stores local API keys and runtime settings.
  - Gitignored.

- `experiment_config.yaml`
  - Points this run at the local AgentBench split.
  - Gitignored by upstream `.gitignore`.

Current local config uses:

```text
benchmark: agentbench
agent_model: gpt-4o
split: train
gate_split: test
agentbench_data_dir: agentbench_local/os_interaction
```

## Local AgentBench Split

The checked-out AgentBench repo did not include an official `data/test.json`.

The plan expected a larger dev/test split, but this AgentBench checkout contains:

```text
data/dev.json: 26 tasks
standard nested data/1 through data/7: 144 tasks
train_0317/training.json: 1000 synthetic/training tasks
```

To unblock local validation, a deterministic pseudo-split was generated from the
144 standard OS tasks:

```text
agentbench_local/os_interaction/data/train.json  108 tasks
agentbench_local/os_interaction/data/test.json    36 tasks
agentbench_local/os_interaction/data/all.json    144 tasks
```

This directory is generated and gitignored:

```text
agentbench_local/
```

## Harness Fixes Made During Testing

These were needed so the local AgentBench score reflects agent behavior instead
of runner bugs:

- Extract `ANSWER:` even when it appears after short rationale text.
- Ignore `ANSWER:` markers inside fenced bash code blocks.
- Execute bash blocks before accepting an `ANSWER:` in the same response.
- Preserve lightweight session context from AgentBench setup commands:
  - current working directory from `cd ...`
  - user/home context from `su - user`
- Provide explicit feedback when:
  - a command produces no output
  - a response has no executable bash block and no answer
- Write `workspace/test_results.json` for validation runs instead of overwriting
  `workspace/train_results.json`.
- Clean stale passing tasks out of `workspace/failure_taxonomy.json`.

## Tracking Files

Added:

- `project_tracker.md`
  - Tracks dataset decisions, baseline scores, iteration notes, and next targets.

- `samhitha-README.md`
  - This file.

Runtime/generated files:

```text
workspace/train_traces.json
workspace/test_traces.json
workspace/train_results.json
workspace/test_results.json
workspace/results.tsv
workspace/failure_taxonomy.json
workspace/success_store.json
```

## Current Scores

Local validation split:

```text
val_score: 0.5000
passed: 18 / 36
```

Current train split:

```text
train_score: 0.5185
passed: 56 / 108
```

These are local pseudo-split scores, not official AgentBench hidden-test scores.

## Intended Optimization Rule Going Forward

Infrastructure is mostly built now. Future optimization should primarily edit:

```text
agent/agent.py
project_tracker.md
```

Only change infrastructure files again if a real harness bug is found.
