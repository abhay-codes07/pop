# agentmd2cpn — the twelve points from the review: what is true, what changed, what to do

Twelve points were raised while the system was explained. This document answers each one as the
system really is, says what was changed because of it, and (Part A) gives the exact steps to apply
the changes. Everything in Part A was built and verified on a copy of the project: the suite goes
from 75 to 79 tests, every example gives the same verdicts as before, and the 168-case DSL sweep is
unchanged.

The short version of what changed:

- **One command, short:** `agentmd2cpn <file>` runs the whole pipeline into `out/<name>` (no `-o`,
  no `--verify full` needed — `auto` already chooses the full state space for files of this size).
- **One stage at a time, no `python -c`:** `agentmd2cpn ingest|extract|compile|verify|check <file>`
  prints that stage's result; the stages before it run quietly and the extraction is reused.
- **A scenario run with its own picture:** `agentmd2cpn drive out/<name> a b c:param=value --png`
  prints every decision with the rule that made it, appends them to `monitor.log`, and writes
  `out/<name>/runs/<date>-drive/marking.png` — the net after the last step, actions green (can fire),
  red (blocked, with the rule names), grey (nothing refuses it but it cannot fire). `--png` is no
  longer mistaken for an action.
- **Every run is kept:** `out/<name>/runs/<date>-pipeline/` and `.../<date>-drive/` hold the logs of
  that run; the files at the top of `out/<name>/` are the latest run. `pipeline.log` is the readable
  log of every stage (`pipeline.jsonl` is the same for machines).
- **The generated file replays runs on its own:** `python out/<name>/<name>_cpn.py a b c:param=value`
  prints ALLOW/BLOCK with the rule names and whether the run can finish, with nothing but cpnpy
  installed; without arguments it re-verifies itself.
- **Less output:** `report.json`, `<name>_cpn.json` and `<name>_cpn.user_code.py` are no longer
  written (cpnpy's JSON comes back with `--json`); the generated file is no longer re-run on every
  pipeline run (`--reverify` does it). `examples/drive.py` is gone — `agentmd2cpn drive` replaces it.
- **The drawing is readable:** `<name>_cpn.png` and `marking.png` are compact (places with their
  tokens, actions coloured by what can fire, one edge per place/action pair) instead of cpnpy's
  full drawing with every guard and arc expression, which was 28 000 pixels wide for the release agent.
- **A stale driver is detected:** `drive` warns when the agent file changed after `agent_spec.yaml`
  was made.

---

## Part A — do this (in order)

### A1. Replace five files (copy from this document; check each hash with the command from DO_THIS A0)

| file | lines | hash |
|---|---|---|
| `agentmd2cpn/cli.py` | 431 | `6d9496a9` |
| `agentmd2cpn/logging_setup.py` | 57 | `71acda6d` |
| `agentmd2cpn/runtime/emit.py` | 299 | `fe4cd2a3` |
| `tests/test_cli.py` | 109 | `e02e51b8` |
| `tests/test_emit.py` | 69 | `e010cfbd` |

The five files are in Part C of this document. Nothing else in the package changes.

### A2. Delete two files

```powershell
git rm examples/drive.py
git rm examples/release_agent/check.py     # if it exists
```

### A3. Check

```powershell
pip install -e .
python -m pytest tests -q
```

→ `79 passed` (two new tests in `test_cli.py`: the stage commands and `drive`; two in `test_emit.py`:
the generated file replays a run with rule names, and its header survives a Windows path — the
`C:\Users` in a docstring that made `ast.parse` fail with *truncated \UXXXXXXXX escape*; the path
is now written with forward slashes). Then, on the review example (its spec is locked,
so no model is called):

```powershell
agentmd2cpn examples\review\review.agent.md
```

```text
examples\review\review.agent.md: 8 rules, 14 reachable markings, consistent=True, traps=0, dead=['merge_pr']
outputs in out\review: REPORT.md, agent_spec.yaml, monitor.log, pipeline.jsonl, pipeline.log, review_cpn.png, review_cpn.py; this run's logs also in out\review\runs\<date>-pipeline
```

```powershell
agentmd2cpn drive out\review read_diff approve_pr:test_status=failing run_tests post_review_summary --png
```

```text
   - | drive: read_diff approve_pr:test_status=failing run_tests post_review_summary
   - | ALLOW read_diff
  R6 | BLOCK approve_pr{'test_status': 'failing'}
   - | ALLOW run_tests
   - | ALLOW post_review_summary
  R4 | BLOCK END
   - | cannot finish: some rule is not satisfied yet
   - | wrote out\review\runs\<date>-drive\marking.png
finished: False - R4 not satisfied
this run: out\review\runs\<date>-drive (marking.png = the net after the last step)
```

Open that `marking.png`: `approve_pr` and `request_changes` are green (the run may finish with either),
`END` is red "blocked by R4", the rule places show their state (`S_R2 SEEN_A`, `S_R3 C1`, ...).

```powershell
python out\review\review_cpn.py read_diff approve_pr:test_status=failing run_tests post_review_summary approve_pr:test_status=passing
```

```text
ALLOW read_diff
BLOCK approve_pr{'test_status': 'failing'} (R6)
ALLOW run_tests
ALLOW post_review_summary
ALLOW approve_pr{'test_status': 'passing'}
DONE: all rules satisfied
```

The same on the release agent, the Friday case:

```powershell
agentmd2cpn examples\release_agent\.Agent.md --extractor yaml:examples\release_agent\spec.yaml --lock
agentmd2cpn drive out\release_agent read_changelog deploy_to_staging get_release_manager_approval deploy_to_production:day=friday deploy_to_production:day=other run_smoke_tests --png
```

```text
   - | ALLOW read_changelog
   - | ALLOW deploy_to_staging
   - | ALLOW get_release_manager_approval
  R8 | BLOCK deploy_to_production{'day': 'friday'}
   - | ALLOW deploy_to_production{'day': 'other'}
   - | ALLOW run_smoke_tests
R4, R7 | BLOCK END
   - | cannot finish: some rule is not satisfied yet
finished: False - R4, R7 not satisfied
```

(R4: the release was never tagged; R7: the last action is not the summary. `--lock` the first time
so that later runs of the release example reuse the reviewed spec; from then on
`agentmd2cpn examples\release_agent\.Agent.md` is enough.)

The stage commands, each printing one stage:

```powershell
agentmd2cpn ingest  examples\review\review.agent.md     # the file as the extractor sees it: segments with line numbers, imports, hash (JSON)
agentmd2cpn extract examples\review\review.agent.md     # the DSL (YAML) - reused when the file is unchanged
agentmd2cpn compile examples\review\review.agent.md     # each rule's automaton and small net, then the fused net with every guard
agentmd2cpn verify  examples\review\review.agent.md     # reachable markings, DONE reachable, traps, dead actions, shortest complete run
agentmd2cpn check   examples\review\review.agent.md     # one refused step per rule (the enforcement check)
```

### A4. Documentation (Copilot) — README, RUNBOOK, ARCHITECTURE, and remove what is gone

```text
Update README.md, docs/ARCHITECTURE.md and docs/RUNBOOK.md (create RUNBOOK.md if it does not exist). No code, no tests. Read agentmd2cpn/cli.py first: the commands and the output files are defined there and the docs must match it exactly. Run the review example (agentmd2cpn examples\review\review.agent.md, then agentmd2cpn drive out\review read_diff run_tests post_review_summary approve_pr:test_status=passing --png, then python out\review\review_cpn.py read_diff run_tests) and paste the real outputs into the RUNBOOK; do not invent any output.

README.md - to the point, in this order: (1) what the tool does in four sentences (markdown agent file -> rule DSL by the model, one net per rule, fused, verified, standalone net + monitor); (2) install: python -m venv .venv; .venv\Scripts\activate; pip install -e .[dev]; copy .env.example .env (Azure values, only needed the first time a file is read); (3) commands, one line each: `agentmd2cpn <file>` (whole pipeline into out/<name>), `agentmd2cpn ingest|extract|compile|verify|check <file>` (one stage, printed), `agentmd2cpn drive out/<name> a b c:param=value [--png]` (one run on the net: decisions with rule names, marking.png), `python out/<name>/<name>_cpn.py [a b c:param=value]` (the generated net on its own: re-verify, or replay a run), options --extractor auto|llm|yaml:<spec>, --lock, --verify, --no-png, --reverify, --json; (4) what out/<name>/ contains: agent_spec.yaml, REPORT.md, <name>_cpn.py, <name>_cpn.png, pipeline.log, pipeline.jsonl, monitor.log, runs/<date>-pipeline/ and runs/<date>-drive/ (one folder per run, never overwritten) - one line each saying what the file is for; (5) the workflow for a new file: run; read "Review first" and "Conflicts" in REPORT.md; correct agent_spec.yaml where a sentence was misread; rerun with --extractor yaml:<that file> --lock; from then on every run is identical and calls no model; (6) reproducibility in one paragraph (model once per file version, rules quote their sentence, locked to the file's hash, everything after the YAML is deterministic); (7) the package layout, one line per file; (8) tests: the number python -m pytest tests -q prints now; (9) limitations (keep the existing ones; add: the picture leaves out the parameter-value places and the RUNNING token; phase_sequence allows a phase with no action of its own).

docs/RUNBOOK.md - a step-by-step for a new agent file: the commands in order with their real outputs from the review example, what to look at in each output, and the Friday scenario on the release agent as a worked example of a blocked step.

docs/ARCHITECTURE.md - the deep part: for each stage the module, its input, its output and the exact mechanism: ingest (markdown-it segments with line numbers, @imports, the hash); extract (3 samples, vote, every rule grounded to its quote at the cited lines, auto/lock); compile (one automaton per rule -> one state place S_<R> holding the rule's state, one transition per action shared by all rules, the rule's part of the guard `_d_R(s_R, '<action>') != 'R:VIOL'`, read-and-write arcs; limit -> budget place B_<R> with n tokens; forbid_when/scope -> a guard part only; phase_sequence -> PHASE_<R> and internal enter_* transitions; ENV_<action>_<param> places for parameter values); compose (synchronous product: transitions with the same name fused, guards ANDed, places united, user code concatenated, the blame map action -> [(rule, guard part)], then RUNNING/END/DONE for finite-trace acceptance; then lowering to cpnpy objects); verify (cpnpy state space or compositional for large files; traps; dead actions; the composition check against the single-rule nets); check (enforcement witnesses: one refused step per rule, implied rules); report (REPORT.md from the results and the quotes - no model involved); emit (the standalone file rendered from the lowered net; what it contains; python <name>_cpn.py). Explain explicitly: nothing in the package knows any particular agent file; the generated file is a rendering of the verified net, not AI output.

Then grep the whole repository (docs, README, examples, comments) for "drive.py", "report.json", "_cpn.json", "user_code", "examples/release_agent/check.py" and fix every mention so it matches the current behaviour (cpnpy JSON only with --json; re-verification only with --reverify). Show me the diff.
```

Read the diff before accepting. The outputs in the RUNBOOK must be the ones you saw in A3.

### A5. Commit

```powershell
python -m pytest tests -q      # 79 passed
git add agentmd2cpn tests examples README.md docs
git status --short            # no out/, no .env, no egg-info, no dist/
git commit -m "agentmd2cpn: stage commands, drive with marking.png, logs per run, standalone replay, compact drawing, smaller output"
git push
```

---

## Part B — the twelve points, answered

### 1. "The whole pipeline should run for each case; every time a new PNG should come; `--png` was taken as an action"

What was true: `examples/drive.py` took every argument as an action, so `--png` became
`BLOCK --png: unknown action`; it drew nothing and kept no record of the run except lines appended
to `monitor.log`.

Now: `agentmd2cpn drive out/<name> a b c:param=value --png`. The decisions are printed with the rule
that made them, appended to `out/<name>/monitor.log`, and written to a folder of their own,
`out/<name>/runs/<date>-drive/` (`monitor.log`, `run.log`, and `marking.png`). `marking.png` is the
net *after the last step*: every rule place with its current state, every action green when it can
fire now, red with the names of the rules refusing it, grey when nothing refuses it but it still
cannot fire; `END` red with the rules that are not satisfied yet. A pipeline run does not need to be
repeated for a scenario — the net is the same net; what changes is the marking, and that is what the
picture shows. The pipeline's own `<name>_cpn.png` is the same picture at the initial marking.

### 2. "Long commands; could a function name run this"

`pip install -e .` installs the console script `agentmd2cpn` (it was already in `pyproject.toml`;
`python -m agentmd2cpn` is the same). The output folder now defaults to `out/<name of the file>`,
and `--verify auto` already picks the full state space for every file with at most 20 rules, so the
whole pipeline is `agentmd2cpn examples\publication\publication.agent.md`. Each stage is a command of
its own (point 3), and a scenario is `agentmd2cpn drive ...`.

### 3. "Step by step means a `python -c` with six imports; is REPORT.md AI; what is re-verify for"

`agentmd2cpn ingest|extract|compile|verify|check <file>` print one stage each. Every stage before
the one asked for runs quietly; the extraction is reused when the file is unchanged (its hash), so
`compile`, `verify` and `check` never call the model when `extract` (or a pipeline run) already did.

`REPORT.md` is not produced by a model. `verification/report.py` is a Python function over the
verification result and the DSL: fixed sentences with the numbers filled in, and the quotes from the
agent file at their line numbers. It is the only place where a conflict is explained as the pair of
sentences that cannot hold together, with the dead end as the Monitor sees it, and where the
"Review first" list (rules the three samples disagreed on; rule-like sentences that produced no rule)
is. It costs one file; if it is not wanted it is one line in `cli.py` (`write_report(...)`) and
`verification/report.py` to delete — nothing else depends on it.

Re-verification (`run_generated`): the pipeline emits `<name>_cpn.py` and ran it in a fresh
interpreter to prove that the standalone file has the same state space as the net that was verified.
That doubled the time of every run. It is now off by default and on with `--reverify`; the file can be
run by hand any time (point 8), which is the same check.

### 4. "How compose.py works, with compile.py"

`compile.py` makes one small net per rule (a `Fragment`: colour sets, places, transitions, arcs, a
piece of user code, an accepting predicate). For an automaton rule (init, end, precedence,
response, forbid, exactly, ...) that is: one place `S_<R>` holding one token, the rule's current
state (`'R2:NO_A'`); one transition per action in the rule's alphabet, named after the action, with
the rule's part of the guard `_d_R2(s_R2, 'post_review_summary') != 'R2:VIOL'`; and two arcs per
action — `S_<R>` → action carrying the state into the variable `s_R2`, action → `S_<R>` putting back
`_d_R2(s_R2, '<action>')`, the next state. `_d_R2` is a table of the automaton written into the
fragment's user code. `limit` makes a place `B_<R>` with n unit tokens and an arc of weight 1 into
the action (each occurrence consumes one). `forbid_when` and `scope` contribute only a guard part
(`not (day == 'friday')`). `phase_sequence` makes a `PHASE_<R>` place, read arcs, and internal
`enter_<phase>` transitions. Every action also gets a bare fragment `ACT_<action>` so that it
exists even with no rule, and a parameterised action gets `ENV_<action>_<param>` places holding the
declared values (read arcs; the binding picks one).

`compose.py` (`compose_sync`) is the synchronous product: places are united (a place of the same id
must be identical); transitions of the same name are fused into one, their guards ANDed and their
variables merged; arcs are deduplicated; user code is concatenated; the accepting predicates are
collected; and the *blame map* records, for every action, which rule contributed which guard part
(and for budget places `tokens:<place>:<n>`). `add_end` adds `RUNNING` (one token every action needs),
the internal transition `END` guarded by the AND of all accepting predicates, and `DONE`: `END`
consumes `RUNNING` and produces `DONE`, so after it nothing can fire — a run is accepted when `END`
can fire. `lower` turns the fragment into cpnpy objects (`FastCPN`, a `CPN` with a per-arc binding
search), the initial marking, and an `EvaluationContext` loaded with the user code.

`agentmd2cpn compile <file>` prints exactly this for any file: each rule's automaton (one line per
state), its place, its guard part and arcs, then the fused net with the initial marking and every
transition's full guard.

### 5. "Every log should be saved for each run"

They are now. `out/<name>/pipeline.log` is the readable log of every stage of the latest pipeline
run (`pipeline.jsonl` the same as JSON records), `monitor.log` the decisions of the replay and of
every drive since. In addition every run writes its own folder: `out/<name>/runs/<date>-pipeline/`
(`run.log`, `monitor.log`) and `out/<name>/runs/<date>-drive/` (`run.log`, `monitor.log`,
`marking.png` with `--png`). Nothing is overwritten there.

### 6. "The driver must not be stale"

`agentmd2cpn drive` reads `out/<name>/agent_spec.yaml` and builds the net from it at every call, so
it always drives the rules of the last pipeline run of that folder. If the agent file was edited after
that run, the spec is stale; `drive` now notices (the spec carries the file's hash) and warns:
*`<file>` has changed since `agent_spec.yaml` was made (hash differs) - this run uses the old rules;
rerun the pipeline first*.

### 7. "How is the cpnpy code produced for each case; which files; any AI; what to remove"

`runtime/emit.py` renders `<name>_cpn.py` from the lowered cpnpy objects of the verified net: every
place with its colour set and initial tokens, every transition with its guard and variables, every
arc with its expression, the colour-set declarations, the user code (the automaton tables), the rules
with their quotes and line numbers, the accepting predicates and the blame map. The only code that is
not generated from the net is copied in verbatim from the package (`FastCPN`, the bounded explorer,
the Monitor), the same for every file. No model is involved after the DSL; no part of the package
knows any particular agent file; a different file gives a different net because its DSL is different.

Output files, after this change: `agent_spec.yaml` (the DSL), `REPORT.md` (the verdict in words),
`<name>_cpn.py` (the net, standalone), `<name>_cpn.png` (the net drawn), `pipeline.log` /
`pipeline.jsonl` (the stages), `monitor.log` (the decisions), `runs/` (one folder per run). Removed
from the default output: `report.json` (its numbers are in `pipeline.log` and `REPORT.md`),
`<name>_cpn.json` and `<name>_cpn.user_code.py` (cpnpy's interchange format and its side file, for
cpnpy's own importer; `--json` writes them). The generated file is no longer re-run on every pipeline
run (`--reverify`).

### 8. "How to run the generated files for any kind of run; they must be the same every run"

`python out/<name>/<name>_cpn.py` re-verifies the net (reachable markings, DONE reachable, traps,
dead actions). `python out/<name>/<name>_cpn.py read_request run_tests publish_change:day=friday`
replays that run: one line per step, `ALLOW ...` or `BLOCK ... (R7)`, then `DONE: all rules
satisfied` or `cannot finish: R5, R8 not satisfied yet`; exit code 0 when the run can finish. It
needs only cpnpy. Same for a program: `m = Monitor(); m.try_action("publish_change", day="friday");
m.why` → `['R7']`.

Same every run: the file is a function of the DSL only. An unchanged agent file reuses its extraction
(or its locked spec), so `agent_spec.yaml`, `<name>_cpn.py` and `<name>_cpn.png` are byte-for-byte
the same from one pipeline run to the next (the file header names the source hash); the picture of
a drive depends only on the net and the steps.

### 9. "Read compile.py and compose.py in detail and verify with an example"

R2 of the review agent, *"Run the automated test suite before posting any comments"* →
`precedence(first=run_tests, then=post_review_summary)`. `agentmd2cpn compile examples\review\review.agent.md` prints:

```text
R2 (precedence): post_review_summary only after run_tests
    place S_R2 (state, colour set CS_R2), initial tokens ['R2:NO_A']
    state NO_A (accepting): run_tests -> SEEN_A; post_review_summary -> VIOL
    state SEEN_A (accepting): run_tests, post_review_summary -> SEEN_A
    on run_tests, post_review_summary - guard: _d_R2(s_R2, 'run_tests') != 'R2:VIOL'
    4 arcs; accepting (what END checks): s_R2 in ('R2:NO_A', 'R2:SEEN_A',)
```

and, in the fused net, the transition `post_review_summary` with the guard
`R1 part and R2 part and R3 part and R4 part`. At the start `S_R2` holds `R2:NO_A`, so
`_d_R2('R2:NO_A', 'post_review_summary')` is `R2:VIOL` and the guard is false — the Monitor says
`BLOCK post_review_summary` and the blame map names R2 (after `read_diff`; before it R1 refuses too,
so the enforcement check reports the step *after read_diff* as R2's own refusal). After `run_tests`
fires, the arc puts `_d_R2('R2:NO_A', 'run_tests')` = `R2:SEEN_A` back into `S_R2`, and
`post_review_summary` passes R2's part. `agentmd2cpn verify` then confirms the whole: 14 reachable
markings, DONE reachable, no trap, `merge_pr` dead (R7 forbids it), shortest complete run
`read_diff -> run_tests -> post_review_summary -> request_changes`.

### 10. "Single state, single transition — how do they combine; what is user_code.py"

Each rule is an automaton with exactly one current state, kept as one token in its own place. Each
action is exactly one transition, shared by every rule that mentions it. When the action fires, every
rule's token moves at once (each rule's arc pair reads its state and writes the next one), and the
action may fire only if every rule's part of the guard holds — the AND that `agentmd2cpn compile`
prints. The net is therefore the synchronous product of the rule automata, plus the budget places and
the parameter-value places; its state space is the product of their states that is actually
reachable, which is what `verify` enumerates.

`<name>_cpn.user_code.py` was the side file of cpnpy's JSON exporter: cpnpy's JSON cannot hold the
Python functions the guards use (the `_d_R` tables), so the exporter writes them next to the JSON for
its importer. It is not needed for `<name>_cpn.py`, which embeds the same code, and it is no longer
written unless `--json` is given.

### 11. "Clean the output"

See point 7: three files removed from the default output, the re-run of the generated file off by
default, `examples/drive.py` and `examples/release_agent/check.py` deleted, the drawing compact.

### 12. "Where is the log of ingest, extract, compile, compose, self-check...; keep a record per run"

`out/<name>/pipeline.log`, one line per event, each tagged with its stage:

```text
2026-11-04 15:29:06 | INFO    | ingest    | - | parsed review.agent.md: 14 segments, 0 code spans, hash 77213e09110f5935
2026-11-04 15:29:06 | INFO    | extract   | - | reviewed spec examples/review/spec.yaml matches this file (hash ...) - reused, LLM not called (9 actions, 8 rules)
2026-11-04 15:29:06 | INFO    | compile   | - | compiled 8 rules into 20 fragments
2026-11-04 15:29:06 | INFO    | compile   | - | fused 20 fragments -> 9 places, 9 transitions, 51 arcs
2026-11-04 15:29:06 | INFO    | compile   | - | lowered to cpnpy: 11 places, 10 transitions, 86 arcs
2026-11-04 15:29:06 | INFO    | verify    | - | state space: 14 markings, 46 arcs
2026-11-04 15:29:06 | INFO    | verify    | - | DONE reachable: True  (rules consistent)
2026-11-04 15:29:06 | INFO    | verify    | - | composition OK: 200 random traces, composed == AND of 8 single-rule nets
2026-11-04 15:29:06 | INFO    | selfcheck | - | enforcement witnesses: 8 of 8 rules refuse a step on their own within 11 markings
2026-11-04 15:29:06 | INFO    | monitor   | - | ALLOW read_diff
2026-11-04 15:29:06 | WARNING | monitor   | R6 | BLOCK approve_pr{'test_status': 'failing'}
2026-11-04 15:29:06 | INFO    | report    | - | wrote out/review/REPORT.md
2026-11-04 15:29:06 | INFO    | emit      | - | wrote out/review/review_cpn.py (366 lines)
```

(compose is part of the compile stage: the `fused ...` and `lowered ...` lines.) The same lines are
in `runs/<date>-pipeline/run.log`, one folder per run, never overwritten; `pipeline.jsonl` holds them
as JSON records with the module, function and line that wrote each one.

---

## Part C — the five files

### C1. `agentmd2cpn/cli.py` (431 lines, hash `6d9496a9`)

```python
"""The command line.

    agentmd2cpn <file.md> [-o out/<name>] [--extractor auto|llm|yaml:<spec>] [--lock] [--verify auto|full|compositional]
        the whole pipeline: ingest -> extract -> compile -> verify -> check -> report -> emit
    agentmd2cpn ingest|extract|compile|verify|check <file.md> [-o out/<name>]
        one stage, its result printed (every stage before it runs quietly; the extraction is reused)
    agentmd2cpn drive <out dir> <action> [<action>:<param>=<value> ...] [--png]
        one run on the net of a finished pipeline: every step ALLOWed or BLOCKed by name, then whether it can finish

Without -o the output folder is out/<name of the agent file>."""
from __future__ import annotations

import argparse
import datetime
import json
import os
import sys
from contextlib import contextmanager
from dataclasses import asdict
from pathlib import Path

os.environ.setdefault("PM4PY_SHOW_PROGRESS_BAR", "False")

import yaml
from loguru import logger

from agentmd2cpn.compilation.compile import compile_spec
from agentmd2cpn.compilation.compose import DONE, END
from agentmd2cpn.compilation.templates import VIOL, build_dfa, english
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.extraction.extract import ExtractionError, LLMExtractor, extract_spec
from agentmd2cpn.extraction.ingest import parse_markdown
from agentmd2cpn.logging_setup import run_dir, setup_logging, stage
from agentmd2cpn.runtime.emit import emit_dot, emit_json, emit_python, run_generated, write_spec
from agentmd2cpn.runtime.monitor import Monitor
from agentmd2cpn.verification.report import write_report
from agentmd2cpn.verification.selfcheck import enforcement_witnesses, fmt_path, replay
from agentmd2cpn.verification.verify import build, composition_check, state_space_check
from agentmd2cpn.verification.verify_large import verify_large

MAX_FULL_RULES, MAX_FULL_BUDGET = 20, 50
COMMANDS = ("run", "ingest", "extract", "compile", "verify", "check", "drive")


def verification_method(spec: AgentSpec, verify: str = "auto") -> str:
    """full = exhaustive state space; compositional = conflicts + bounded exploration (verify_large).
    auto: full while the rule set is small (<= 20 compiled rules) and the budgets of limit/exactly rules
    add up to <= 50 tokens (each token multiplies the state space)."""
    if verify in ("full", "compositional"):
        return verify
    compiled = [r for r in spec.rules if r.modality != "should"]
    budget = sum(int(r.args["n"]) for r in compiled if r.kind in ("limit", "exactly"))
    return "full" if len(compiled) <= MAX_FULL_RULES and budget <= MAX_FULL_BUDGET else "compositional"


def agent_name(path: Path) -> str:
    """orchestrator.agent.md -> orchestrator, team.skills.md -> team, AGENTS.md -> AGENTS; a dot-file such as
    .Agent.md takes the name of its folder."""
    if path.name.startswith("."):
        return path.parent.name or "agent"
    return path.name.split(".")[0] or "agent"


def default_out(path: Path) -> Path:
    return Path("out") / agent_name(path)


def lock_path(path: Path) -> Path:
    """Where a reviewed spec for this file lives: <dir>/<agent_name>.spec.yaml, or the older <dir>/spec.yaml if present."""
    named, plain = path.parent / f"{agent_name(path)}.spec.yaml", path.parent / "spec.yaml"
    return plain if plain.is_file() and not named.is_file() else named


def load_spec(src: Path) -> AgentSpec:
    try:
        return AgentSpec.model_validate(yaml.safe_load(src.read_text(encoding="utf-8")))
    except FileNotFoundError:
        logger.error("no such spec file: {}", src)
        raise SystemExit(2)
    except Exception as e:                                           # pydantic ValidationError, yaml errors
        msg = "; ".join(x["msg"] for x in e.errors()) if hasattr(e, "errors") else str(e)
        logger.error("{} is not a valid AgentSpec: {}", src, msg)
        raise SystemExit(2)


def find_spec(path: Path, out: Path, sha: str) -> tuple[Path, str] | None:
    """The spec to reuse for exactly this version of the file: the reviewed spec next to it, else the previous
    extraction in `out`. A spec whose source_hash is missing or differs is named and skipped (the file changed)."""
    for cand, what in ((lock_path(path), "reviewed spec"), (out / "agent_spec.yaml", "previous extraction")):
        if not cand.is_file():
            continue
        try:
            got = (yaml.safe_load(cand.read_text(encoding="utf-8")) or {}).get("source_hash", "")
        except Exception:
            got = ""
        if got == sha:
            return cand, what
        logger.warning("{} {} is for another version of {} (source_hash {} != {}) - not reused", what, cand, path.name, got or "missing", sha)
    return None


# ------------------------------------------------------------------ the stages

@contextmanager
def _quiet():
    """Build sub-nets without their per-build compile/compose log lines (the fused net was already logged)."""
    for m in ("agentmd2cpn.compilation.compile", "agentmd2cpn.compilation.compose"):
        logger.disable(m)
    try:
        yield
    finally:
        for m in ("agentmd2cpn.compilation.compile", "agentmd2cpn.compilation.compose"):
            logger.enable(m)


def ingest_stage(path: Path):
    with stage("ingest"):
        doc = parse_markdown(path)
        logger.info("parsed {}: {} segments, {} code spans, hash {}", path.name, len(doc.segments), len(doc.code_spans), doc.sha)
    return doc


def extract_stage(path: Path, out: Path, doc, extractor: str = "auto", min_votes: int = 2, samples: int = 3,
                  complete=None, lock: bool = False) -> tuple[AgentSpec, str]:
    """extractor: auto = the reviewed spec next to the file or the previous extraction in `out` when either matches
    the file's hash, else the LLM; llm = always the LLM; yaml:<file> = that spec. Writes out/agent_spec.yaml.
    lock = also write the spec, stamped with the file's hash, next to the file so every later run reuses it."""
    with stage("extract"):
        reused = find_spec(path, out, doc.sha) if extractor == "auto" else None
        if extractor.startswith("yaml:"):
            src = Path(extractor[5:])
            spec = load_spec(src)
            if spec.source_hash and spec.source_hash != doc.sha:
                logger.warning("{} was made for another version of {} (source_hash {} != {}): line numbers and quotes may be off", src, path.name, spec.source_hash, doc.sha)
            note = f"rules from {src} ({spec.provenance or 'no provenance recorded'})"
            logger.info("loaded DSL from {} ({} actions, {} rules)", src, len(spec.actions), len(spec.rules))
        elif reused:
            src, what = reused
            try:
                spec = load_spec(src)
            except SystemExit:
                logger.error("{} matches this file but does not load (an unfinished draft?): fix it and pass --extractor yaml:{}, or delete it to extract again", src, src)
                raise
            note = f"rules reused from the {what} {src} ({spec.provenance or 'no provenance recorded'}) - same file content, the LLM was not called"
            logger.info("{} {} matches this file (hash {}) - reused, LLM not called ({} actions, {} rules)", what, src, doc.sha, len(spec.actions), len(spec.rules))
        else:
            if complete is None:
                from agentmd2cpn.extraction.llm_client import complete
            try:
                spec = extract_spec(doc, agent_name(path), LLMExtractor(complete, samples=samples), min_votes=min_votes)
            except ExtractionError as e:
                draft = out / "agent_spec.yaml"
                out.mkdir(parents=True, exist_ok=True)
                draft.write_text(yaml.safe_dump(e.raw, sort_keys=False), encoding="utf-8")
                logger.error("extracted DSL is invalid: {}", "; ".join(x["msg"] for x in e.error.errors()))
                logger.error("draft written to {} - fix it, then rerun with --extractor yaml:{}", draft, draft)
                raise SystemExit(2)
            spec.provenance = (f"llm {os.environ.get('AZURE_OPENAI_DEPLOYMENT_NAME', '?')} samples={samples} min_votes={min_votes} "
                               f"{datetime.date.today().isoformat()}")
            note = f"rules extracted now ({spec.provenance}) - a draft until reviewed and locked"
        spec.source_hash = doc.sha
        out.mkdir(parents=True, exist_ok=True)
        write_spec(spec, out / "agent_spec.yaml")
        if lock:
            dst = lock_path(path)
            if not spec.provenance.startswith("reviewed"):
                spec.provenance = f"reviewed {datetime.date.today().isoformat()}" + (f"; {spec.provenance}" if spec.provenance else "")
            write_spec(spec, dst)
            logger.info("locked: {} will be reused for this version of {} until the file changes", dst, path.name)
    return spec, note


def verify_stage(spec: AgentSpec, cpn, m0, ctx, mode: str, verify: str) -> tuple[dict, int]:
    with stage("verify"):
        compiled = sum(1 for r in spec.rules if r.modality != "should")
        if verification_method(spec, verify) == "full":
            res = state_space_check(cpn, m0, ctx)
            res["method"] = "full"
        else:
            logger.info("{} rules: full state space not enumerable, using compositional verification", compiled)
            with _quiet():
                res = verify_large(spec, mode)
        with _quiet():                                                   # the single-rule nets are built again here
            n = composition_check(spec, samples=200)
        logger.info("verified: {} markings{}, DONE reachable {}, trap states {}; composition check on {} random traces",
                    res["states"], "" if res.get("complete", True) else " (bounded)", res["done_reachable"], res["trap_states"], n)
    return res, n


@logger.catch(reraise=True)
def run(path: Path, out: Path | None = None, extractor: str = "auto", mode: str = "enforce", min_votes: int = 2, samples: int = 3,
        complete=None, verify: str = "auto", png: bool = True, lock: bool = False, reverify: bool = False, json_net: bool = False) -> dict:
    """The whole pipeline. Writes into `out` (default out/<name>): agent_spec.yaml, REPORT.md, <name>_cpn.py, <name>_cpn.png,
    pipeline.log, pipeline.jsonl, monitor.log, and a dated copy of the logs under runs/. reverify = run the generated
    file in a fresh interpreter and compare; json_net = also write cpnpy's JSON interchange file (+ its user code)."""
    path = Path(path)
    out = Path(out) if out is not None else default_out(path)
    rd = run_dir(out, "pipeline")
    setup_logging(out, run=rd)
    doc = ingest_stage(path)
    spec, note = extract_stage(path, out, doc, extractor, min_votes, samples, complete, lock)
    with stage("compile"):
        frag, cpn, m0, ctx = build(spec, mode)
        logger.info("built the net: {} places, {} transitions, {} arcs", len(cpn.places), len(cpn.transitions), len(cpn.arcs))
    res, n = verify_stage(spec, cpn, m0, ctx, mode, verify)
    compiled = sum(1 for r in spec.rules if r.modality != "should")
    with stage("selfcheck"):
        res["enforcement"] = enforcement_witnesses(spec, frag, cpn, m0, ctx, res.get("witness"))
        enf = res["enforcement"]
        logger.info("enforcement witnesses: {} of {} rules refuse a step on their own within {} markings{}{}",
                    len(enf["witnesses"]), compiled, enf["states"],
                    f"; {len(enf['implied'])} implied by other rules ({', '.join(enf['implied'])})" if enf["implied"] else "",
                    f"; none found for {', '.join(enf['unwitnessed'])}" if enf["unwitnessed"] else "")
        replay(spec, frag, cpn, m0, ctx, res.get("witness"), res["enforcement"])
    with stage("report"):
        write_report(doc, spec, res, out, mode, note)
    with stage("emit"):
        py = emit_python(spec, frag, cpn, m0, out / f"{spec.name}_cpn.py")
        ok = run_generated(py) if reverify else None
        if json_net:
            emit_json(cpn, m0, ctx, out / f"{spec.name}_cpn.json")
        if png:
            emit_dot(cpn, m0, out / f"{spec.name}_cpn.png", ctx, frag.blame)
    report = {"source": str(path), "actions": [a.id for a in spec.actions],
              "rules": [json.loads(r.model_dump_json()) for r in spec.rules],
              "state_space": {k: v for k, v in res.items() if k != "analyzer"},
              "composition_traces_checked": n, "generated_file_reverified": ok, "out": str(out), "run_dir": str(rd),
              "files": sorted(f.name for f in out.iterdir() if f.is_file())}
    logger.bind(stage="monitor").info("end of pipeline run of {} (hash {}); a run on this net: agentmd2cpn drive {} <action> [<action>:<param>=<value> ...]", path.name, doc.sha, out)
    return report


def summary_line(path: Path, rep: dict) -> str:
    s = rep["state_space"]
    tail = f", generated file re-verified={rep['generated_file_reverified']}" if rep["generated_file_reverified"] is not None else ""
    if s.get("method") == "compositional":
        return (f"{path}: {len(rep['rules'])} rules, compositional check: conflicts={len(s['conflicts'])}, consistent={s['done_reachable']}, "
                f"explored={s['states']} markings{'' if s['complete'] else ' (bounded)'}, traps={s['trap_states']}, dead={s['dead_transitions']}{tail}")
    return f"{path}: {len(rep['rules'])} rules, {s['states']} reachable markings, consistent={s['done_reachable']}, traps={s['trap_states']}, dead={s['dead_transitions']}{tail}"


# ------------------------------------------------------------------ one stage at a time

def _steps(args: list[str]) -> list[tuple[str, dict]]:
    """'publish_change:day=friday' -> ('publish_change', {'day': 'friday'})."""
    out = []
    for arg in args:
        action, _, rest = arg.partition(":")
        out.append((action, dict(kv.split("=", 1) for kv in rest.split(",") if kv)))
    return out


def drive(out: Path, steps: list[str], png: bool = False, level: str = "INFO") -> bool:
    """One run on the net of a finished pipeline folder: the decisions go to the console, to <out>/monitor.log and to
    <out>/runs/<date>-drive/ (with marking.png - the net after the last step - when png is set)."""
    out = Path(out)
    spec_file = out / "agent_spec.yaml"
    if not spec_file.is_file():
        raise SystemExit(f"agentmd2cpn drive: {spec_file} not found - run the pipeline on the agent file first")
    rd = run_dir(out, "drive")
    setup_logging(out, level=level, fresh=False, run=rd, console="monitor")   # console: only the decisions, with the rule
    log = logger.bind(stage="monitor")
    log.info("drive: {}", " ".join(steps) or "(finish only)")
    spec = load_spec(spec_file)
    src = Path(spec.source_file) if spec.source_file else None
    if src is not None and src.is_file() and spec.source_hash and parse_markdown(src).sha != spec.source_hash:
        log.warning("{} has changed since {} was made (hash differs) - this run uses the old rules; rerun the pipeline first", src, spec_file)
    frag, cpn, m0, ctx = build(spec)
    m = Monitor(cpn, m0, ctx, blame=frag.blame)
    for action, params in _steps(steps):
        m.try_action(action, **params)
    ok = m.finish()
    if png:
        with stage("monitor"):
            emit_dot(cpn, m.marking, rd / "marking.png", ctx, frag.blame)
    print(f"finished: {ok}" + ("" if ok else f" - {', '.join(m.explain(END, {})) or 'a rule'} not satisfied"))
    print(f"this run: {rd}" + (" (marking.png = the net after the last step)" if png else ""))
    return ok


def _quiet_spec(a) -> tuple[Path, Path, object, AgentSpec]:
    """ingest + extract for a stage command: console shows warnings only; the extraction is reused when it matches."""
    path = Path(a.agent_md)
    out = Path(a.out) if a.out else default_out(path)
    setup_logging(None, level="WARNING")
    doc = ingest_stage(path)
    spec, _ = extract_stage(path, out, doc, a.extractor, lock=a.lock)
    return path, out, doc, spec


def cmd_ingest(a) -> None:
    setup_logging(None, level="WARNING")
    doc = ingest_stage(Path(a.agent_md))
    print(json.dumps(asdict(doc), indent=2, ensure_ascii=False))


def cmd_extract(a) -> None:
    _, out, _, spec = _quiet_spec(a)
    print((out / "agent_spec.yaml").read_text(encoding="utf-8"))


def cmd_compile(a) -> None:
    """Each rule as its own small net, then the fused net - what build() makes from the DSL."""
    _, _, _, spec = _quiet_spec(a)
    frags = compile_spec(spec, a.mode)
    by_id = {r.id: r for r in spec.rules}
    for f in frags:
        if f.id.startswith("ACT_"):
            continue
        if f.id.startswith("ENV_"):
            p = f.places[0]
            print(f"{f.id}: parameter values of an action - place {p.id} with tokens {p.tokens}")
            continue
        r = by_id.get(f.id)
        print(f"{f.id} ({r.kind}): {english(r.kind, r.args)}" if r else f.id)
        for p in f.places:
            print(f"    place {p.id} ({p.kind}, colour set {p.colorset}), initial tokens {p.tokens}")
        dfa = build_dfa(r.kind, r.args) if r else None
        if dfa is not None:                                              # the rule's automaton, one line per state
            dfa = dfa.complete([x.id for x in spec.actions])
            for st in dfa.states:
                moves: dict[str, list[str]] = {}
                for act in dfa.alphabet:
                    moves.setdefault(dfa.delta.get((st, act), VIOL), []).append(act)
                print(f"    state {st}{' (accepting)' if st in dfa.accepting else ''}: " +
                      "; ".join(f"{', '.join(acts) if len(acts) < 4 else f'{len(acts)} other actions'} -> {nxt}" for nxt, acts in moves.items()))
        acts = [t for t in f.transitions if not t.internal]
        if acts:
            g = next((t.guard for t in acts if t.guard), None)
            print(f"    on {', '.join(t.id for t in acts)}" + (f" - guard: {g}" if g else ""))
        for t in f.transitions:
            if t.internal:
                print(f"    internal transition {t.id} - guard: {t.guard}")
        print(f"    {len(f.arcs)} arcs" + (f"; accepting (what END checks): {f.accepting}" if f.accepting else ""))
    frag, cpn, m0, ctx = build(spec, a.mode)
    print(f"\nfused net: {len(cpn.places)} places, {len(cpn.transitions)} transitions, {len(cpn.arcs)} arcs")
    print("places (initial marking):")
    for p in cpn.places:
        print(f"    {p.name}: {[t.value for t in m0.get_multiset(p.name).tokens]}")
    print("transitions (one per action, plus END; the guard is the AND of every rule's part):")
    for t in cpn.transitions:
        print(f"    {t.name}" + (f"  guard: {t.guard_expr}" if t.guard_expr else ""))


def cmd_verify(a) -> None:
    _, _, _, spec = _quiet_spec(a)
    frag, cpn, m0, ctx = build(spec, a.mode)
    res, n = verify_stage(spec, cpn, m0, ctx, a.mode, a.verify)
    print(f"method: {res['method']}")
    print(f"reachable markings: {res['states']}" + ("" if res.get("complete", True) else " (bounded)"))
    print(f"DONE reachable (every rule can be satisfied in one run): {res['done_reachable']}")
    print(f"trap states (runs that can neither continue nor finish): {res['trap_states']}")
    print(f"dead actions (can never happen): {[d for d in res.get('dead_transitions', []) if d != END] or 'none'}")
    w = res.get("witness")
    if w:
        print("shortest complete run: " + " -> ".join(str(x) for x in w))
    for c in res.get("conflicts") or []:
        print(f"conflict: {c['why']} among {', '.join(c['rules'])} - removing any one of {', '.join(c['culprits'])} repairs it")
    print(f"composition check: {n} random traces agree with the single-rule automata")


def cmd_check(a) -> None:
    _, _, _, spec = _quiet_spec(a)
    frag, cpn, m0, ctx = build(spec, a.mode)
    res, _ = verify_stage(spec, cpn, m0, ctx, a.mode, a.verify)
    enf = enforcement_witnesses(spec, frag, cpn, m0, ctx, res.get("witness"))
    for r in spec.rules:
        if r.modality == "should":
            continue
        head = f"{r.id} ({r.kind}): {english(r.kind, r.args)}"
        if r.id in enf["witnesses"]:
            w = enf["witnesses"][r.id]
            what = "finishing" if w["step"] == END else w["step"] + (f"{w['params']}" if w["params"] else "")
            print(f"{head}\n    after {fmt_path(w['path'])}: {what} -> BLOCKED by {r.id} alone")
        elif r.id in enf["implied"]:
            print(f"{head}\n    never refuses a step on its own - implied by {enf['implied'][r.id]['by']}: {enf['implied'][r.id]['why']}")
        else:
            print(f"{head}\n    no step refused by this rule alone within {enf['states']} markings")


def main(argv=None):
    argv = list(sys.argv[1:] if argv is None else argv)
    if argv and argv[0] not in COMMANDS and not argv[0].startswith("-"):    # agentmd2cpn <file> ... == agentmd2cpn run <file> ...
        argv.insert(0, "run")
    ap = argparse.ArgumentParser(prog="agentmd2cpn", description="any markdown agent file -> rule DSL -> one verified coloured Petri net -> runtime monitor")
    sub = ap.add_subparsers(dest="cmd", required=True, metavar="command")

    def common(p, verify=True):
        p.add_argument("agent_md", type=Path, help="the agent file (.Agent.md, *.agent.md, AGENTS.md, any .md)")
        p.add_argument("-o", "--out", type=Path, default=None, help="output folder (default: out/<name of the file>)")
        p.add_argument("--extractor", default="auto",
                       help="auto (default) = reuse the reviewed spec next to the file or the previous extraction when its hash matches the file, else the LLM | llm = always the LLM | yaml:<spec.yaml>")
        p.add_argument("--lock", action="store_true", help="write the spec used in this run, stamped with the file's hash, next to the file (<name>.spec.yaml) for reuse")
        p.add_argument("--mode", default="enforce", choices=["enforce", "monitor"])
        if verify:
            p.add_argument("--verify", default="auto", choices=["auto", "full", "compositional"],
                           help="full = exhaustive state space (default up to 20 rules); compositional = conflicts + witness search")

    r = sub.add_parser("run", help="the whole pipeline (default when the first argument is a file)")
    common(r)
    r.add_argument("--min-votes", type=int, default=2)
    r.add_argument("--samples", type=int, default=3)
    r.add_argument("--no-png", action="store_true", help="skip the Graphviz drawing (slow and unreadable for large nets)")
    r.add_argument("--reverify", action="store_true", help="also run the generated <name>_cpn.py in a fresh interpreter and compare")
    r.add_argument("--json", action="store_true", help="also write cpnpy's JSON interchange file (<name>_cpn.json + its user code)")
    for name, h in (("ingest", "print the file as the extractor sees it (segments with line numbers, imports, hash)"),
                    ("extract", "print the DSL (reused when the file is unchanged, else extracted)"),
                    ("compile", "print each rule's small net and the fused net"),
                    ("verify", "print the state-space result"),
                    ("check", "print the enforcement check: one refused step per rule")):
        common(sub.add_parser(name, help=h), verify=name in ("verify", "check"))
    d = sub.add_parser("drive", help="one run on the net of a finished pipeline folder")
    d.add_argument("out", type=Path, help="the pipeline's output folder (it has agent_spec.yaml)")
    d.add_argument("steps", nargs="*", help="actions in order, parameters as action:param=value[,param=value]")
    d.add_argument("--png", action="store_true", help="draw the net after the last step into runs/<date>-drive/marking.png")
    a = ap.parse_args(argv)
    if a.cmd != "drive" and not a.agent_md.is_file():
        raise SystemExit(f"agentmd2cpn: no such file: {a.agent_md}")
    if a.cmd == "run":
        rep = run(a.agent_md, a.out, a.extractor, a.mode, a.min_votes, a.samples, verify=a.verify, png=not a.no_png, lock=a.lock,
                  reverify=a.reverify, json_net=a.json)
        print(summary_line(a.agent_md, rep))
        print(f"outputs in {rep['out']}: {', '.join(rep['files'])}; this run's logs also in {rep['run_dir']}")
    elif a.cmd == "drive":
        drive(a.out, a.steps, png=a.png)
    else:
        {"ingest": cmd_ingest, "extract": cmd_extract, "compile": cmd_compile, "verify": cmd_verify, "check": cmd_check}[a.cmd](a)


if __name__ == "__main__":
    main()
```

### C2. `agentmd2cpn/logging_setup.py` (57 lines, hash `71acda6d`)

```python
"""Loguru configuration: one logger, context bound per stage / rule.

Sinks of a pipeline run on <out>:  the console; <out>/pipeline.log (every stage, readable); <out>/pipeline.jsonl (the
same, one JSON record per line); <out>/monitor.log (runtime decisions only); and, when a run folder is given, a copy
of the readable log and of the decisions in <out>/runs/<date>-<kind>/ - one folder per run, never overwritten."""
from __future__ import annotations

import datetime
import sys
from pathlib import Path

from loguru import logger

FMT = "<green>{time:HH:mm:ss}</green> | <level>{level:<7}</level> | <cyan>{extra[stage]}</cyan> | {message}"
FILE_FMT = "{time:YYYY-MM-DD HH:mm:ss} | {level:<7} | {extra[stage]:<9} | {extra[rule]} | {message}"
MONITOR_FMT = "{time:YYYY-MM-DD HH:mm:ss} | {extra[rule]} | {message}"


def _monitor(record) -> bool:
    return record["extra"].get("stage") == "monitor"


def run_dir(out: Path, kind: str) -> Path:
    """<out>/runs/<YYYYMMDD-HHMMSS>-<kind>/, a new folder for this run (a suffix keeps two runs in one second apart)."""
    base = Path(out) / "runs" / f"{datetime.datetime.now():%Y%m%d-%H%M%S}-{kind}"
    d, i = base, 1
    while d.exists():
        i += 1
        d = base.with_name(f"{base.name}-{i}")
    d.mkdir(parents=True)
    return d


def setup_logging(out_dir: Path | None = None, level: str = "INFO", fresh: bool = True, run: Path | None = None,
                  console: str = "stages") -> None:
    """fresh=True (a pipeline run) starts pipeline.log, pipeline.jsonl and monitor.log empty, so the folder always holds
    the latest run; fresh=False (a drive on an existing folder) appends to them. `run` = this run's own folder.
    console="stages" shows every stage line; console="monitor" shows only the decisions, each with its rule."""
    logger.remove()
    logger.configure(extra={"stage": "-", "rule": "-"})
    if console == "monitor":
        logger.add(sys.stderr, level=level, filter=_monitor, format="<cyan>{extra[rule]:>4}</cyan> | {message}", colorize=True)
    else:
        logger.add(sys.stderr, level=level, format=FMT, colorize=True)
    if out_dir is not None:
        out_dir.mkdir(parents=True, exist_ok=True)
        mode = "w" if fresh else "a"
        logger.add(out_dir / "pipeline.log", level="INFO", mode=mode, format=FILE_FMT)
        logger.add(out_dir / "pipeline.jsonl", level="DEBUG", serialize=True, mode=mode)
        logger.add(out_dir / "monitor.log", level="INFO", mode=mode, filter=_monitor, format=MONITOR_FMT)
    if run is not None:
        logger.add(run / "run.log", level="INFO", format=FILE_FMT)
        logger.add(run / "monitor.log", level="INFO", filter=_monitor, format=MONITOR_FMT)


def stage(name: str):
    return logger.contextualize(stage=name)
```

### C3. `agentmd2cpn/runtime/emit.py` (299 lines, hash `fe4cd2a3`)

```python
"""Emit the composed net as (a) a standalone runnable cpnpy Python file, (b) a Graphviz PNG of a marking,
(c) cpnpy's own JSON interchange file (on request).

The Python file is generated from the LOWERED cpnpy objects - every place, transition, arc and guard is written out
from the net that was verified, nothing is composed again.  It carries the rules with their quotes and line numbers,
the blame map (which rule guards which action), and a Monitor, so it runs on its own with cpnpy installed:
    python <name>_cpn.py                       re-verify (reachable markings, DONE reachable, traps, dead actions)
    python <name>_cpn.py a b c:param=value     replay a run: every step ALLOWed or BLOCKed with the rule's name
"""
from __future__ import annotations

import ast
import inspect
import json
import shutil
import subprocess
import sys
from pathlib import Path

from jinja2 import Environment
from loguru import logger

from agentmd2cpn.compilation.compose import DONE, END, RUNNING
from agentmd2cpn.dsl import AgentSpec, Fragment
from agentmd2cpn.compilation.templates import english
from agentmd2cpn.compilation.fastcpn import FastCPN
from agentmd2cpn.verification.verify_large import explore

_ENV = Environment()
_ENV.filters["pyrepr"] = repr
TEMPLATE = _ENV.from_string('''"""{{ name }} - coloured Petri net generated by agentmd2cpn from {{ source }}{% if source_hash %} (hash {{ source_hash }}){% endif %}
{% if provenance %}rules: {{ provenance }}
{% endif %}
Every rule below was compiled from one sentence of the agent file (see RULES for the quotes),
each rule is one small net, and the nets were fused on shared action transitions.
Re-verify:                  python {{ filename }}
Replay a run:               python {{ filename }} read_request run_tests publish_change:day=friday
Use it as a monitor:        m = Monitor(); m.try_action("git_push", branch="main"); m.why
"""
import copy
import heapq
import itertools
import sys

from cpnpy.cpn.cpn_imp import CPN, Place, Transition, Arc, Marking, EvaluationContext
from cpnpy.cpn.colorsets import ColorSetParser


{{ fastcpn }}


{{ explore }}

COLORSETS = """
{{ colorsets }}
"""

USER_CODE = """
{{ user_code }}
"""

# rule id -> (kind, meaning, source line, verbatim quote)
RULES = {
{%- for r in rules %}
    {{ r.id | pyrepr }}: ({{ r.kind | pyrepr }}, {{ r.meaning | pyrepr }}, {{ r.line }}, {{ r.quote | pyrepr }}),
{%- endfor %}
}

ACTIONS = {{ actions | pyrepr }}

# rule id -> accepting predicate over its state variable (what END checks)
ACCEPTING = {{ accepting | pyrepr }}

# action -> [(rule id, that rule's part of the guard, or 'tokens:<place>:<n>' for a budget place)]
BLAME = {{ blame | pyrepr }}


def build():
    cs = ColorSetParser().parse_definitions(COLORSETS)
    cpn = FastCPN()
    P, T = {}, {}
{%- for p in places %}
    P[{{ p.name | pyrepr }}] = Place({{ p.name | pyrepr }}, cs[{{ p.cs | pyrepr }}]); cpn.add_place(P[{{ p.name | pyrepr }}])
{%- endfor %}
{%- for t in transitions %}
    T[{{ t.name | pyrepr }}] = Transition({{ t.name | pyrepr }}, guard={{ t.guard | pyrepr }}, variables={{ t.variables | pyrepr }}); cpn.add_transition(T[{{ t.name | pyrepr }}])
{%- endfor %}
{%- for a in arcs %}
    cpn.add_arc(Arc({{ a.src }}, {{ a.dst }}, {{ a.expr | pyrepr }}))
{%- endfor %}
    return cpn


def initial_marking():
    m = Marking()
{%- for p in places %}
    m.set_tokens({{ p.name | pyrepr }}, {{ p.tokens }})
{%- endfor %}
    return m


def context():
    ctx = EvaluationContext(user_code=USER_CODE)
    ctx.env["__original_user_code__"] = USER_CODE
    return ctx


class Monitor:
    """Runtime guard: one agent tool call = one attempt to fire the action's transition.
    After every call `why` holds the rules that refused it ([] when it was allowed)."""

    def __init__(self, verbose=False):
        self.cpn, self.marking, self.ctx = build(), initial_marking(), context()
        self.verbose, self.why = verbose, []

    def explain(self, action, params):
        """Which rule(s) refuse the action in the current marking (each rule's own part of the guard)."""
        binding = dict(params)
        for p, ms in self.marking._marking.items():
            if p.startswith(("S_", "PHASE_")) and ms.tokens:
                binding[("s_" + p[2:]) if p.startswith("S_") else ("ph_" + p[6:])] = ms.tokens[0].value
        who = []
        for rid, part in BLAME.get(action, []):
            if part.startswith("tokens:"):
                _, place, w = part.split(":")
                if len(self.marking.get_multiset(place).tokens) < int(w):
                    who.append(rid)
            else:
                try:
                    if not eval(part, self.ctx.env, dict(binding)):
                        who.append(rid)
                except Exception:
                    who.append(rid)
        return who

    def _say(self, text):
        if self.verbose:
            print(text)

    def try_action(self, action, **params):
        t = self.cpn.get_transition_by_name(action)
        shown = action + (repr(params) if params else "")
        if t is None:
            self.why = ["unknown action"]
            self._say("BLOCK " + shown + " (unknown action)")
            return False
        for b in self.cpn._find_all_bindings(t, self.marking, self.ctx):
            if all(b.get(k) == v for k, v in params.items()):
                self.cpn.fire_transition(t, self.marking, self.ctx, binding=b)
                self.why = []
                self._say("ALLOW " + shown)
                return True
        self.why = self.explain(action, params) or ["no binding"]
        self._say("BLOCK " + shown + " (" + ", ".join(self.why) + ")")
        return False

    def can_finish(self):
        t = self.cpn.get_transition_by_name({{ END | pyrepr }})
        return bool(self.cpn._find_all_bindings(t, self.marking, self.ctx))

    def finish(self):
        ok = self.can_finish()
        self.why = [] if ok else self.explain({{ END | pyrepr }}, {})
        self._say("DONE: all rules satisfied" if ok else "cannot finish: " + ", ".join(self.why) + " not satisfied yet")
        return ok


def replay(steps):
    """steps like ['read_request', 'publish_change:day=friday'] -> the decisions, then whether the run can finish."""
    m = Monitor(verbose=True)
    for arg in steps:
        action, _, rest = arg.partition(":")
        m.try_action(action, **dict(kv.split("=", 1) for kv in rest.split(",") if kv))
    return m.finish()


def verify(max_markings=5000):
    """Bounded best-first exploration; complete (a full state space) when the bound is not hit."""
    cpn, m0, ctx = build(), initial_marking(), context()
    r = explore(cpn, m0, ctx, ACCEPTING, max_markings)
    print("reachable markings:", r["states"], "" if r["complete"] else "(bounded)")
    print("DONE reachable (rules consistent):", r["done_reachable"], "" if r["complete"] or r["done_reachable"] else "(not found within bound)")
    print("trap states:", r["trap_states"], "" if r["complete"] else "(within bound)")
    print("dead transitions:", r["dead_transitions"] or "none", "" if r["complete"] else "(not fired within bound)")
    return r["done_reachable"] and not r["trap_states"]


if __name__ == "__main__":
    ok = replay(sys.argv[1:]) if len(sys.argv) > 1 else verify()
    raise SystemExit(0 if ok else 1)
''')


def emit_python(spec: AgentSpec, frag: Fragment, cpn, marking, out: Path) -> Path:
    places = [{"name": p.name, "cs": p.colorset.name or p.colorset.__class__.__name__,
               "tokens": repr([t.value for t in marking.get_multiset(p.name).tokens])} for p in cpn.places]
    transitions = [{"name": t.name, "guard": t.guard_expr, "variables": list(t.variables)} for t in cpn.transitions]
    arcs = []
    for a in cpn.arcs:
        if hasattr(a.source, "colorset"):
            arcs.append({"src": f'P[{a.source.name!r}]', "dst": f'T[{a.target.name!r}]', "expr": a.expression})
        else:
            arcs.append({"src": f'T[{a.source.name!r}]', "dst": f'P[{a.target.name!r}]', "expr": a.expression})
    rules = [{"id": r.id, "kind": r.kind, "meaning": english(r.kind, r.args),
              "line": r.source.line_begin if r.source else 0, "quote": r.source.quote if r.source else ""} for r in spec.rules]
    code = TEMPLATE.render(fastcpn=inspect.getsource(FastCPN), explore=inspect.getsource(explore),
                           accepting=dict(frag.blame.get(END, [])), blame={k: [tuple(x) for x in v] for k, v in frag.blame.items()}, name=spec.name, source=spec.source_file.replace("\\", "/"), source_hash=spec.source_hash, provenance=spec.provenance, filename=out.name,
                           colorsets="\n".join(frag.colorsets.values()), user_code=frag.user_code,
                           rules=rules, actions=[a.id for a in spec.actions],
                           places=places, transitions=transitions, arcs=arcs, END=END, DONE=DONE)
    ast.parse(code)                                            # must be valid Python
    out.write_text(code, encoding="utf-8")
    logger.info("wrote {} ({} lines)", out, code.count("\n"))
    return out


def run_generated(path: Path) -> bool:
    """The emitted file is executed in a fresh interpreter: it must re-verify on its own."""
    env = {"PM4PY_SHOW_PROGRESS_BAR": "False"}
    r = subprocess.run([sys.executable, str(path)], capture_output=True, text=True, timeout=600, env={**__import__("os").environ, **env})
    for line in r.stdout.strip().splitlines():
        logger.bind(stage="emit").info("generated file says: {}", line)
    if r.returncode != 0:
        logger.bind(stage="emit").error("generated file failed: {}", r.stderr[-800:])
    return r.returncode == 0


def emit_json(cpn, marking, ctx, out: Path) -> Path:
    from cpnpy.cpn import exporter
    exporter.export_cpn_to_json(cpn, marking, ctx, str(out), str(out.with_suffix(".user_code.py")))
    logger.info("wrote {}", out)
    return out


def emit_dot(cpn, marking, out: Path, ctx=None, blame: dict | None = None) -> Path | None:
    """The net in one readable picture, for this marking: every rule's state place with its token, every action as a
    box - green when it can fire now, red with the names of the rules that refuse it, grey when nothing refuses it
    but it still cannot fire (no token) - and one edge per place/action pair (two-headed when the action reads and
    writes the place).  Parameter-value places (ENV_*) and the RUNNING token every action needs are left out."""
    if not shutil.which("dot"):
        logger.warning("graphviz 'dot' not found - skipping diagram")
        return None
    import graphviz
    from agentmd2cpn.runtime.monitor import Monitor
    from agentmd2cpn.verification.selfcheck import _param_choices
    g = graphviz.Digraph("net", graph_attr={"rankdir": "LR", "nodesep": "0.25", "ranksep": "0.8", "fontname": "Helvetica", "splines": "true"},
                         node_attr={"fontname": "Helvetica", "fontsize": "10"}, edge_attr={"color": "#777777", "arrowsize": "0.6"})
    shown = set()
    for p in cpn.places:
        if p.name.startswith("ENV_"):
            continue
        toks = [t.value for t in marking.get_multiset(p.name).tokens]
        text = ", ".join("\u2022" if t == () else str(t).split(":", 1)[-1] for t in toks) or "(empty)"
        if len(toks) > 3:
            text = f"{len(toks)} tokens"
        g.node("P__" + p.name, f"{p.name}\n{text}", shape="ellipse", style="filled", fillcolor="#fff3b0" if toks else "white")
        shown.add(p.name)
    mon = Monitor(cpn, marking, ctx, blame=blame) if ctx is not None else None
    for t in cpn.transitions:
        if mon is None:
            g.node("T__" + t.name, t.name, shape="box", style="rounded")
            continue
        if cpn._find_all_bindings(t, marking, ctx):
            g.node("T__" + t.name, t.name, shape="box", style="filled,rounded", fillcolor="#c6efce")
            continue
        who = None
        for params in _param_choices(cpn, marking, t.name) if t.name != END else [{}]:
            w = set(mon.explain(t.name, params))
            who = w if who is None else who & w
        if who:
            g.node("T__" + t.name, f"{t.name}\nblocked by {', '.join(sorted(who, key=lambda x: (len(x), x)))}", shape="box", style="filled,rounded", fillcolor="#f8c9c9")
        else:
            g.node("T__" + t.name, t.name, shape="box", style="filled,rounded", fillcolor="#e0e0e0")
    pairs: dict[tuple, set] = {}
    for a in cpn.arcs:
        if hasattr(a.source, "colorset"):
            pname, tname, d = a.source.name, a.target.name, "in"
        else:
            pname, tname, d = a.target.name, a.source.name, "out"
        if pname not in shown or (pname == RUNNING and tname != END):
            continue
        pairs.setdefault((pname, tname), set()).add(d)
    for (pname, tname), ds in pairs.items():
        if ds == {"in", "out"}:
            g.edge("P__" + pname, "T__" + tname, dir="both")
        elif ds == {"in"}:
            g.edge("P__" + pname, "T__" + tname)
        else:
            g.edge("T__" + tname, "P__" + pname)
    g.render(filename=str(out.resolve().with_suffix("")), format="png", cleanup=True)
    logger.info("wrote {}", out)
    return out


def write_spec(spec: AgentSpec, out: Path) -> Path:
    import yaml
    out.write_text(yaml.safe_dump(json.loads(spec.model_dump_json()), sort_keys=False), encoding="utf-8")
    logger.info("wrote {} (the DSL)", out)
    return out
```

### C4. `tests/test_cli.py` (109 lines, hash `e02e51b8`)

```python
import json
from pathlib import Path

import pytest
import yaml
from loguru import logger

from agentmd2cpn.cli import main, run
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.logging_setup import setup_logging
from agentmd2cpn.runtime.monitor import Monitor
from agentmd2cpn.verification.verify import build

FIX = Path(__file__).parent / "fixtures" / "a"


def test_cli_end_to_end_with_reviewed_dsl(tmp_path):
    rep = run(FIX / ".Agent.md", tmp_path, extractor=f"yaml:{FIX / 'spec.yaml'}")
    assert rep["state_space"]["done_reachable"] and rep["generated_file_reverified"] is None       # not re-run by default
    assert [r["id"] for r in rep["rules"]] == ["R1", "R2"] and rep["composition_traces_checked"] == 200
    for f in ["agent_spec.yaml", "REPORT.md", "demo_cpn.py", "pipeline.log", "pipeline.jsonl", "monitor.log"]:
        assert (tmp_path / f).exists(), f
    for f in ["demo_cpn.json", "demo_cpn.user_code.py", "report.json"]:                       # only on request
        assert not (tmp_path / f).exists(), f
    lines = [json.loads(l) for l in (tmp_path / "pipeline.jsonl").read_text(encoding="utf-8").splitlines()]
    assert {l["record"]["extra"]["stage"] for l in lines} >= {"ingest", "extract", "compile", "verify", "selfcheck", "report", "emit"}
    text = (tmp_path / "pipeline.log").read_text(encoding="utf-8")
    assert "| ingest    | - | parsed .Agent.md" in text and "| verify    | - | verified:" in text and "| monitor   | R1 | BLOCK git_commit" in text
    runs = sorted((tmp_path / "runs").iterdir())
    assert len(runs) == 1 and runs[0].name.endswith("-pipeline") and (runs[0] / "run.log").read_text(encoding="utf-8") == text
    assert (runs[0] / "monitor.log").read_text(encoding="utf-8") == (tmp_path / "monitor.log").read_text(encoding="utf-8")
    rep = run(FIX / ".Agent.md", tmp_path, extractor=f"yaml:{FIX / 'spec.yaml'}", png=False, reverify=True, json_net=True)
    assert rep["generated_file_reverified"] is True
    assert (tmp_path / "demo_cpn.json").exists() and len(list((tmp_path / "runs").iterdir())) == 2


def test_monitor_log_sink(tmp_path):
    setup_logging(tmp_path)
    spec = AgentSpec.model_validate(yaml.safe_load((FIX / "spec.yaml").read_text(encoding="utf-8")))
    frag, cpn, m0, ctx = build(spec)
    mon = Monitor(cpn, m0, ctx, blame=frag.blame)
    assert not mon.try_action("git_commit") and mon.try_action("run_tests")
    logger.complete()
    text = (tmp_path / "monitor.log").read_text(encoding="utf-8")
    assert "| R1 | BLOCK git_commit" in text and "| - | ALLOW run_tests" in text


def test_invalid_extraction_writes_a_draft_for_review(tmp_path):
    bad = json.dumps({"actions": [{"id": "edit_file", "params": {"path": "file path"}}],
                      "rules": [{"kind": "scope", "args": {"action": "edit_file", "where": "path == 'deploy'"},
                                 "quote": "Only modify files under src/.", "line_begin": 10, "line_end": 10}]})
    with pytest.raises(SystemExit) as e:
        run(FIX / ".Agent.md", tmp_path, extractor="llm", samples=2, min_votes=1, complete=lambda p: bad)
    assert e.value.code == 2
    draft = yaml.safe_load((tmp_path / "agent_spec.yaml").read_text(encoding="utf-8"))
    assert draft["actions"][0]["params"] == {"path": "file path"} and draft["rules"][0]["kind"] == "scope"
    assert not (tmp_path / "a_cpn.py").exists()


def test_verification_method_depends_on_size_and_budgets():
    from agentmd2cpn.cli import verification_method
    from agentmd2cpn.dsl import AgentSpec
    acts = [{"id": "a"}, {"id": "b"}]
    small = AgentSpec.model_validate({"name": "t", "actions": acts, "rules": [{"id": "R1", "kind": "limit", "args": {"action": "a", "n": 2}}]})
    big_budget = AgentSpec.model_validate({"name": "t", "actions": acts, "rules": [{"id": "R1", "kind": "limit", "args": {"action": "a", "n": 60}}]})
    many = AgentSpec.model_validate({"name": "t", "actions": acts, "rules": [{"id": f"R{i}", "kind": "precedence", "args": {"first": "a", "then": "b"}} for i in range(21)]})
    assert verification_method(small) == "full" and verification_method(big_budget) == "compositional" and verification_method(many) == "compositional"
    assert verification_method(many, "full") == "full" and verification_method(small, "compositional") == "compositional"


def test_each_pipeline_run_starts_its_logs_fresh_and_dates_monitor_log(tmp_path):
    for _ in range(2):
        run(FIX / ".Agent.md", tmp_path, extractor=f"yaml:{FIX / 'spec.yaml'}", png=False)
    jl = (tmp_path / "pipeline.jsonl").read_text(encoding="utf-8")
    assert jl.count('"parsed ') == 1                                   # not two runs appended
    mon = (tmp_path / "monitor.log").read_text(encoding="utf-8").splitlines()
    assert sum("end of pipeline run" in l for l in mon) == 1 and "agentmd2cpn drive" in mon[-1]   # one run, dated
    assert len(list((tmp_path / "runs").iterdir())) == 2                                          # but every run is kept


def test_drive_appends_decisions_and_keeps_each_run(tmp_path, capsys):
    run(FIX / ".Agent.md", tmp_path, extractor=f"yaml:{FIX / 'spec.yaml'}", png=False)
    main(["drive", str(tmp_path), "git_commit", "run_tests", "git_push:branch=main", "git_commit"])
    out = capsys.readouterr().out
    assert "finished: True" in out
    mon = (tmp_path / "monitor.log").read_text(encoding="utf-8")
    assert "| - | drive: git_commit run_tests git_push:branch=main git_commit" in mon
    assert "| R1 | BLOCK git_commit" in mon and "| R2 | BLOCK git_push{'branch': 'main'}" in mon and "| - | ALLOW git_commit" in mon
    assert "DONE: all rules satisfied" in mon and "end of pipeline run" in mon                    # appended after the pipeline's lines
    drives = [d for d in (tmp_path / "runs").iterdir() if d.name.endswith("-drive")]
    assert len(drives) == 1 and "BLOCK git_push" in (drives[0] / "monitor.log").read_text(encoding="utf-8")


def test_one_stage_at_a_time(tmp_path, capsys):
    md, spec = str(FIX / ".Agent.md"), f"yaml:{FIX / 'spec.yaml'}"
    main(["ingest", md])
    doc = json.loads(capsys.readouterr().out)
    assert doc["sha"] and any(s["text"].startswith("Never push directly to main") for s in doc["segments"])
    main(["extract", md, "-o", str(tmp_path), "--extractor", spec])
    assert "kind: forbid_when" in capsys.readouterr().out and (tmp_path / "agent_spec.yaml").exists()
    main(["compile", md, "-o", str(tmp_path)])                          # auto: the extraction above is reused
    out = capsys.readouterr().out
    assert "R1 (precedence)" in out and "place S_R1" in out and "fused net:" in out and "git_push  guard:" in out
    main(["verify", md, "-o", str(tmp_path)])
    out = capsys.readouterr().out
    assert "method: full" in out and "DONE reachable (every rule can be satisfied in one run): True" in out and "trap states" in out
    main(["check", md, "-o", str(tmp_path)])
    out = capsys.readouterr().out
    assert "after (start): git_commit -> BLOCKED by R1 alone" in out and "git_push{'branch': 'main'} -> BLOCKED by R2 alone" in out
```

### C5. `tests/test_emit.py` (69 lines, hash `e010cfbd`)

```python
import importlib.util
import subprocess
import sys

from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.runtime.emit import emit_dot, emit_json, emit_python, run_generated, write_spec
from agentmd2cpn.verification.verify import build

SPEC = AgentSpec.model_validate({
    "name": "demo", "source_file": "demo/.Agent.md",
    "data": {"BRANCH": {"type": "enum", "values": ["main", "other"]}},
    "actions": [{"id": "run_tests"}, {"id": "git_commit"}, {"id": "git_push", "params": {"branch": "BRANCH"}},
                {"id": "write_secret"}],
    "rules": [
        {"id": "R1", "kind": "precedence", "args": {"first": "run_tests", "then": "git_commit"},
         "source": {"line_begin": 9, "quote": "Always run the test suite before committing."}},
        {"id": "R2", "kind": "limit", "args": {"action": "run_tests", "n": 3}},
        {"id": "R3", "kind": "forbid_when", "args": {"action": "git_push", "where": "branch == 'main'"}},
        {"id": "R4", "kind": "forbid", "args": {"action": "write_secret"}},
    ]})


def test_generated_file_is_standalone(tmp_path):
    frag, cpn, m0, ctx = build(SPEC)
    py = emit_python(SPEC, frag, cpn, m0, tmp_path / "demo_cpn.py")
    text = py.read_text(encoding="utf-8")
    assert "Always run the test suite before committing." in text and "'R1': ('precedence'" in text
    r = subprocess.run([sys.executable, str(py)], capture_output=True, text=True, timeout=300)
    assert r.returncode == 0, r.stderr
    assert "reachable markings: 8" in r.stdout and "DONE reachable (rules consistent): True" in r.stdout
    assert "trap states: 0" in r.stdout and "dead transitions: ['write_secret']" in r.stdout
    assert run_generated(py)
    s = importlib.util.spec_from_file_location("demo_cpn", py)
    mod = importlib.util.module_from_spec(s)
    s.loader.exec_module(mod)
    mon = mod.Monitor()
    assert not mon.try_action("git_commit") and mon.try_action("run_tests") and mon.try_action("git_commit")
    assert not mon.try_action("git_push", branch="main") and mon.can_finish()


def test_json_yaml_png(tmp_path):
    frag, cpn, m0, ctx = build(SPEC)
    j = emit_json(cpn, m0, ctx, tmp_path / "demo_cpn.json")
    assert j.exists() and "S_R1" in j.read_text(encoding="utf-8")
    y = write_spec(SPEC, tmp_path / "agent_spec.yaml")
    assert "kind: precedence" in y.read_text(encoding="utf-8")
    png = emit_dot(cpn, m0, tmp_path / "demo_cpn.png")
    assert png is None or png.exists()                        # None only when graphviz 'dot' is not installed


def test_generated_file_replays_a_run_with_rule_names(tmp_path):
    frag, cpn, m0, ctx = build(SPEC)
    py = emit_python(SPEC, frag, cpn, m0, tmp_path / "demo_cpn.py")
    r = subprocess.run([sys.executable, str(py), "git_commit", "run_tests", "git_commit", "git_push:branch=main", "write_secret", "nothing"],
                       capture_output=True, text=True, timeout=300)
    lines = r.stdout.strip().splitlines()
    assert lines == ["BLOCK git_commit (R1)", "ALLOW run_tests", "ALLOW git_commit", "BLOCK git_push{'branch': 'main'} (R3)",
                     "BLOCK write_secret (R4)", "BLOCK nothing (unknown action)", "DONE: all rules satisfied"], r.stdout
    assert r.returncode == 0
    r = subprocess.run([sys.executable, str(py), "run_tests", "run_tests", "run_tests", "run_tests"], capture_output=True, text=True, timeout=300)
    assert "BLOCK run_tests (R2)" in r.stdout and r.returncode == 0                  # the fourth run_tests exceeds the limit of 3


def test_generated_file_header_survives_a_windows_path(tmp_path):
    spec = SPEC.model_copy(update={"source_file": "C:\\Users\\abhay\\demo\\.Agent.md"})     # backslash-U would be a unicode escape
    frag, cpn, m0, ctx = build(spec)
    py = emit_python(spec, frag, cpn, m0, tmp_path / "demo_cpn.py")
    assert "from C:/Users/abhay/demo/.Agent.md" in py.read_text(encoding="utf-8")
    assert run_generated(py)
```
