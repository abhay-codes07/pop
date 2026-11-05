# agentmd2cpn — back to the commit, then the twelve points one by one

You asked for three things: put the tree back to what was committed, know how anyone runs any
`.agent.md` with short commands, and fix the mentor's points one at a time with short Copilot
prompts you can read and check. This file is exactly that. No file of mine to copy; every step is a
small prompt, a check you run yourself, and a sentence on when to accept.

What keeps every step honest, whoever writes the code:

- **The grep.** Nothing in the package may name any agent file's actions, rules or paths. After every
  step: `Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"`
  must print nothing. If it prints a line, the step is rejected.
- **The tests.** The 75 tests are not edited by Copilot (new behaviour gets *new* test files). After
  every step `python -m pytest tests -q` must pass with the old count plus whatever new tests were added.
- **A new file.** Step 8 runs everything on an agent file written today, which the package has never
  seen. If it works there, nothing was tuned to an example.

---

## 0. Reset to the commit

Nothing is deleted; the uncommitted attempt is stashed in case a piece of it is wanted later.

```powershell
git status --short                                    # see what is uncommitted
git stash push -u -m "mentor-points attempt"          # tracked changes and new files, out of the way (ignored files such as .env and out/ are untouched)
git status --short                                    # must print nothing
pip install -e .
python -m pytest tests -q                             # 75 passed - the committed state
```

(`git stash list` shows it; `git stash drop` discards it later; `git stash pop` would bring it back.)

## How anyone runs any `.agent.md` — now, and after the steps below

Today (the commit):

```powershell
agentmd2cpn path\to\any.agent.md -o out\any            # the whole pipeline (first run: the model reads the file; later runs reuse it)
python examples\drive.py out\any action1 action2 action3:param=value     # one run on the net, decisions into monitor.log
python out\any\any_cpn.py                               # the generated net re-verifies itself
```

After the steps below:

```powershell
agentmd2cpn path\to\any.agent.md                        # whole pipeline into out\<name>; agent_spec.yaml, REPORT.md, <name>_cpn.py, <name>_cpn.png, logs
agentmd2cpn ingest|extract|compile|compose|verify|check path\to\any.agent.md    # one stage, printed
agentmd2cpn drive out\<name> action1 action2 action3:param=value --png          # one run: ALLOW/BLOCK with the rule, marking.png of the net after it
python out\<name>\<name>_cpn.py action1 action2 action3:param=value             # the same run on the generated file alone (no package needed)
```

Nothing in these commands names an agent; `<name>` is taken from the file name, the actions come
from the file's own DSL in `out\<name>\agent_spec.yaml`.

---

## The steps (easy first; one Copilot prompt each; check before the next)

Paste each prompt into Copilot (agent mode). Every prompt begins with the same rule line so that it
is never forgotten.

### Step 1 — short command, default output folder (point 2)

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; reuse the existing functions; do not edit any existing test; run python -m pytest tests -q at the end and show the count.
In agentmd2cpn/cli.py make -o optional: when it is not given, the output folder is out/<agent_name(path)> (agent_name already exists in cli.py). Update the module docstring and the argparse help to say so. Nothing else changes.
```

Check:

```powershell
agentmd2cpn examples\review\review.agent.md
```

→ the same summary line as before (`8 rules, 14 reachable markings, …`) and the files are in
`out\review\`. Accept if the grep prints nothing and the tests pass (75).

### Step 2 — `drive` as a command, with `--png`, never stale (points 1 and 6)

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; reuse the existing functions (Monitor, build, emit_dot, setup_logging); do not edit any existing test; add tests only in a new file tests/test_drive.py; run python -m pytest tests -q at the end and show the count.
Add a subcommand `agentmd2cpn drive <out dir> <step>... [--png]` to cli.py that replaces examples/drive.py (delete examples/drive.py and examples/release_agent/check.py). A step is `action` or `action:param=value[,param=value]`. It loads <out>/agent_spec.yaml, builds the net with build(), runs the steps on a Monitor with the blame map, then calls finish(). The decisions must appear on the console as `<rule or -> | ALLOW/BLOCK ...` (the rule ids, as monitor.log already has them) and be appended to <out>/monitor.log. With --png, draw the net in the marking reached after the last step to <out>/runs/<YYYYMMDD-HHMMSS>-drive/marking.png (use emit_dot with that marking). Before building, if agent_spec.yaml has source_file and source_hash and the file exists and its hash (parse_markdown(file).sha) differs, log a warning that the spec is stale and the pipeline should be rerun. The old form `agentmd2cpn <file> ...` must keep working: if the first argument is not a known subcommand, treat it as `run`. Update the "end of pipeline run" log line in cli.py to mention `agentmd2cpn drive` instead of examples/drive.py. Any test that asserted "examples/drive.py" in that line: tell me which, do not change it.
```

Check:

```powershell
agentmd2cpn drive out\review read_diff approve_pr:test_status=failing run_tests post_review_summary --png
```

→ one line per step: `ALLOW read_diff`, `BLOCK approve_pr{'test_status': 'failing'}` with `R6`,
`ALLOW run_tests`, `ALLOW post_review_summary`, then `BLOCK END` with `R4` and `cannot finish`;
`out\review\monitor.log` has the same lines appended after the pipeline's; a file
`out\review\runs\<date>-drive\marking.png` exists and is a readable picture. Copilot will report one
existing test that asserts `examples/drive.py` in the last monitor line
(`test_each_pipeline_run_starts_its_logs_fresh_and_dates_monitor_log`): change that one string
yourself, `"examples/drive.py"` → `"agentmd2cpn drive"`. Accept if the grep prints nothing and the
tests pass (75 + the new ones).

Why `drive` cannot be hardcoded: it has no list of actions. It reads whatever `agent_spec.yaml` the
pipeline wrote for *that* folder, builds the net from it, and asks the net. Run it with a wrong action
name and it says `BLOCK <name>: unknown action`.

### Step 3 — one stage at a time (point 3)

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; reuse the existing functions; do not edit any existing test; add tests only in a new file tests/test_stage_commands.py; run python -m pytest tests -q at the end and show the count.
Add subcommands to cli.py, each taking <file.md> and the same -o/--extractor/--lock/--mode options as run, each running the stages before it quietly (console at WARNING) and reusing the extraction when the file is unchanged, and printing only its own result:
- ingest: the Document as JSON (dataclasses.asdict).
- extract: the DSL YAML (what run writes to out/<name>/agent_spec.yaml).
- compile: for every rule fragment from compile_spec: the rule id, kind and english() meaning; its places with colour set and initial tokens; its automaton from build_dfa(...).complete(actions) as one line per state (state, accepting or not, and for each target state the actions that lead there); the transitions it contributes and its guard part; the number of arcs; its accepting predicate.
- compose: the result of compose_sync on those fragments: places with initial tokens; every transition with its variables and its full guard (the AND of the rules' parts); the blame map as `action: R1, R2, ...`; the accepting predicate; then what add_end adds (RUNNING, END, DONE) and the totals after lower().
- verify: method, reachable markings, DONE reachable, trap states, dead actions, shortest complete run, conflicts if any, the composition check count.
- check: for every compiled rule, its enforcement witness (path, refused step) or that none was found.
```

Check (each prints one stage; none calls the model, the review spec is locked):

```powershell
agentmd2cpn ingest  examples\review\review.agent.md
agentmd2cpn extract examples\review\review.agent.md
agentmd2cpn compile examples\review\review.agent.md
agentmd2cpn compose examples\review\review.agent.md
agentmd2cpn verify  examples\review\review.agent.md
agentmd2cpn check   examples\review\review.agent.md
```

→ `compile` shows, for R2, a state line like `NO_A (accepting): run_tests -> SEEN_A; post_review_summary -> VIOL`
and the guard part `_d_R2(s_R2, ...) != 'R2:VIOL'`; `compose` shows `post_review_summary` with a guard
that is the AND of the R1, R2, R3, R4 parts and the blame line `post_review_summary: R1, R2, R3, R4`;
`verify` says 14 markings, DONE reachable True, dead `merge_pr`; `check` lists eight rules with a
refused step each. Accept if the grep prints nothing and the tests pass.

### Step 4 — a readable log, and every run kept (points 5 and 12)

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; do not edit any existing test; add tests only in a new file tests/test_logs.py; run python -m pytest tests -q at the end and show the count.
In agentmd2cpn/logging_setup.py add a readable sink out/<name>/pipeline.log (format: date time | level | stage | rule | message, same fresh/append behaviour as pipeline.jsonl) and, for every run, a folder out/<name>/runs/<YYYYMMDD-HHMMSS>-<kind>/ (kind = pipeline or drive; add a suffix if the folder exists) that receives a copy of the readable log (run.log) and of the monitor lines (monitor.log) of that run. The pipeline (run) and drive create their folder at start; drive's marking.png from step 2 goes into the same folder. pipeline.jsonl and monitor.log at the top of out/<name>/ stay as they are (latest run). During the verify stage, the per-build compile/compose log lines of the single-rule nets (composition check, verify_large) must not be logged - disable the agentmd2cpn.compilation.compile and agentmd2cpn.compilation.compose loggers around those calls, the same way verification/report.py already does in _quiet().
```

Check:

```powershell
agentmd2cpn examples\review\review.agent.md
type out\review\pipeline.log
dir out\review\runs
```

→ `pipeline.log` has one line per event, each with its stage (`ingest`, `extract`, `compile`,
`verify`, `selfcheck`, `monitor`, `report`, `emit`), no repeated `compiled 1 rules into …` lines; a new
`runs\<date>-pipeline\` folder with `run.log` and `monitor.log`; a second run adds a second folder.
Accept if the grep prints nothing and the tests pass.

### Step 5 — the generated file replays a run on its own (points 7 and 8)

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; do not edit any existing test; add tests only in a new file tests/test_generated_replay.py; run python -m pytest tests -q at the end and show the count.
In agentmd2cpn/runtime/emit.py extend the TEMPLATE of the generated <name>_cpn.py: embed BLAME = the fragment's blame map (action -> [(rule id, guard part)]); give the generated Monitor an explain(action, params) that evaluates each rule's guard part against the current state tokens exactly as agentmd2cpn/runtime/monitor.py does, a verbose mode that prints `ALLOW <action><params>` or `BLOCK <action><params> (R1, R2)`, and a finish() that prints `DONE: all rules satisfied` or `cannot finish: <rules> not satisfied yet`; add a replay(steps) that takes `action` / `action:param=value` strings; in __main__: with arguments replay them and exit 0 if the run can finish, without arguments verify() as now. The generated file must stay standalone (cpnpy only). Any path written into the generated file's docstring must use forward slashes (a Windows path with \U would be read as a unicode escape).
```

Check:

```powershell
agentmd2cpn examples\review\review.agent.md
python out\review\review_cpn.py read_diff approve_pr:test_status=failing run_tests post_review_summary approve_pr:test_status=passing
python out\review\review_cpn.py
```

→ the first prints `ALLOW read_diff`, `BLOCK approve_pr{'test_status': 'failing'} (R6)`, `ALLOW run_tests`,
`ALLOW post_review_summary`, `ALLOW approve_pr{'test_status': 'passing'}`, `DONE: all rules satisfied`;
the second re-verifies (14 markings, DONE reachable True, dead `['merge_pr']`). Accept if the grep
prints nothing and the tests pass.

### Step 6 — less output (points 7 and 11)

Before the prompt, change three lines in `tests/test_cli.py` yourself (this is the one existing test
that pins the output files; it must say what the output *should* be now):

- in `test_cli_end_to_end_with_reviewed_dsl`, replace the list `["agent_spec.yaml", "demo_cpn.py", "demo_cpn.json", "report.json", "pipeline.jsonl"]`
  with `["agent_spec.yaml", "demo_cpn.py", "REPORT.md", "pipeline.jsonl"]`;
- in the same test replace `rep["generated_file_reverified"]` with `rep["generated_file_reverified"] is None`;
- add the line `assert not (tmp_path / "demo_cpn.json").exists() and not (tmp_path / "report.json").exists()`.

Then:

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; do not edit any test; run python -m pytest tests -q at the end and show the count.
In cli.py: the generated file is no longer run again on every pipeline run - add --reverify (default off; when on, run_generated as before and report it in the summary line, otherwise generated_file_reverified is None and the summary line omits it); report.json is no longer written (the dict run() returns stays as it is); cpnpy's JSON (<name>_cpn.json and its user_code file) is written only with --json. Update the module docstring and the README section that lists the output files.
```

Check: `agentmd2cpn examples\review\review.agent.md` → `dir out\review` shows `agent_spec.yaml`,
`REPORT.md`, `review_cpn.py`, `review_cpn.png`, `pipeline.log`, `pipeline.jsonl`, `monitor.log`,
`runs\` and nothing else. Accept if the grep prints nothing and the tests pass.

### Step 7 (optional) — a picture one can read (point 1)

cpnpy's drawing writes every guard and arc expression; for the release agent it is 28 000 pixels
wide. If you want a usable `marking.png`:

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; do not edit any existing test; run python -m pytest tests -q at the end and show the count.
Rewrite emit_dot(cpn, marking, out, ctx=None, blame=None) in agentmd2cpn/runtime/emit.py with the graphviz package: one ellipse per place except ENV_* places, labelled with the place name and its tokens (state tokens without the 'R1:' prefix; a unit token as a bullet); one rounded box per transition, green when it has a binding in this marking, red with "blocked by R1, R2" when it has none and those rules refuse it for every parameter choice (use Monitor.explain from runtime/monitor.py and _param_choices from verification/selfcheck.py), grey otherwise; one edge per place/transition pair (two-headed when arcs go both ways), none from RUNNING except to END; node ids must not contain ':'. Pass ctx and the blame map from both call sites in cli.py.
```

Check: `agentmd2cpn drive out\review read_diff --png` → in the new `marking.png` only the actions
the rules allow after `read_diff` are green, `END` is red with `R3, R4`, and the picture fits on a screen.

### Step 8 — the playbook on a file the package has never seen (points 4, 9, 10 and the "is it hardcoded" doubt)

Create `examples\ticket_agent\ticket_agent.agent.md` yourself (type it; it is data, not code):

```markdown
---
name: "ticket_agent"
description: "Answers one customer support ticket per run"
tools: [read, reply, escalate]
---
# Ticket Agent

You handle one customer support ticket per run, in the order below.

## Run sequence

1. Start by reading the ticket.
2. Classify the ticket before replying to the customer.
3. Reply to the customer exactly once per run.
4. Finish by closing the ticket or escalating it.

## Safety

- Never delete a ticket.
- Never issue a refund on a weekend.
- Notify the on-call manager before escalating a ticket.
- Do not retry a failed reply more than 2 times.

## Style

- Keep replies short and polite.
```

What a correct reading is (to judge the model's): line 12 `init`; 13 `precedence`; 14 `exactly 1`;
15 `end` with two alternatives; 19 `forbid`; 20 `forbid_when` on a `day` parameter the model has to
invent; 21 `precedence`; 22 `limit 2`; line 26 no rule. First the grep that proves the package does
not know this file:

```powershell
Select-String -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "ticket|refund|escalat|classify"
```

→ nothing. Then the playbook prompt:

```text
Create docs/PLAYBOOK.md by actually running these commands in the terminal, in this order, pasting each command's complete real output under it (nothing invented or shortened; if one fails, stop and show me), and adding two or three sentences per step on what the output shows. Change no code and no test. The file is examples/ticket_agent/ticket_agent.agent.md.
0. Select-String -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "ticket|refund|escalat|classify"   (must print nothing)
1. agentmd2cpn ingest examples/ticket_agent/ticket_agent.agent.md
2. agentmd2cpn extract examples/ticket_agent/ticket_agent.agent.md   (the model reads the file: three samples, vote; list the eight sentences and the rule each produced, with votes)
3. agentmd2cpn compile ...   4. agentmd2cpn compose ...   5. agentmd2cpn verify ...   6. agentmd2cpn check ...   (same file)
7. agentmd2cpn examples/ticket_agent/ticket_agent.agent.md, then paste out/ticket_agent/REPORT.md and out/ticket_agent/pipeline.log in full
8. agentmd2cpn drive out/ticket_agent <read> <classify> <refund with day=weekend> <reply> <escalate> --png, with the exact action ids and values from step 2; explain each BLOCK; name the marking.png
9. agentmd2cpn drive out/ticket_agent <read> <classify> <reply> <notify manager> <escalate>   (must finish)
10. python out/ticket_agent/ticket_agent_cpn.py   and   python out/ticket_agent/ticket_agent_cpn.py <the steps of 8>
11. Get-FileHash of agent_spec.yaml, ticket_agent_cpn.py, ticket_agent_cpn.png; run step 7 again; the log must say the extraction was reused (LLM not called) and the three hashes must be unchanged
12. Get-ChildItem out/ticket_agent and out/ticket_agent/runs
End with REPORT.md's "Review first" section and whether the reading needs a correction; then run agentmd2cpn examples/ticket_agent/ticket_agent.agent.md --extractor yaml:out/ticket_agent/agent_spec.yaml --lock and paste its "locked:" line.
```

Read `docs/PLAYBOOK.md` yourself: every output must be one you can reproduce. This is what your
mentor runs and reads.

### Step 9 — README, RUNBOOK, ARCHITECTURE (points 4, 9, 10 in writing)

```text
Update README.md, docs/ARCHITECTURE.md and docs/RUNBOOK.md (create it if missing). No code, no tests. Read agentmd2cpn/cli.py first; the commands and output files must match it. README, to the point: what the tool does (four sentences); install (venv, pip install -e .[dev], .env from .env.example - only needed the first time a file is read); the commands, one line each (run, ingest, extract, compile, compose, verify, check, drive, python out/<name>/<name>_cpn.py); what out/<name>/ contains, one line per file; the workflow for a new file (run; read Review first and Conflicts in REPORT.md; correct agent_spec.yaml; rerun with --extractor yaml:<file> --lock; from then on identical and no model); reproducibility in one paragraph; the layout, one line per file; the test count as pytest prints it; limitations. RUNBOOK: the commands of docs/PLAYBOOK.md in order with what to look at. ARCHITECTURE: per stage the module, input, output and mechanism - compile: one automaton per rule -> one state place S_<R>, one transition per action shared by all rules, the rule's guard part `_d_R(s_R, '<action>') != 'R:VIOL'`, read-and-write arcs, limit -> budget place B_<R>, forbid_when/scope -> guard part only, ENV_<action>_<param> places; compose: synchronous product (same-name transitions fused, guards ANDed, places united, blame map), RUNNING/END/DONE; verify; check; report (no model involved); emit (the standalone file rendered from the verified net). Say explicitly that nothing in the package knows any particular agent file. Then grep the repo for "drive.py", "report.json", "_cpn.json", "user_code" and fix every stale mention. Show me the diff.
```

### Step 10 — commit

```powershell
python -m pytest tests -q
Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"
git add agentmd2cpn tests examples docs README.md
git status --short
git commit -m "agentmd2cpn: stage commands, drive with marking.png, logs per run, standalone replay, smaller output, playbook on a new file"
git push
```

---

## The points this answers, and what is only explanation

| point | what fixes it |
|---|---|
| 1 scenario runs, a new PNG each time, `--png` taken as an action | step 2 (+7) |
| 2 long commands | step 1, step 3 |
| 3 `python -c` per stage; is REPORT.md AI; what is re-verify | step 3; REPORT.md is a Python function over the verifier's result plus the quotes (no model); re-verify runs the generated file to prove it matches — now opt-in (step 6) |
| 4 how compose works | `agentmd2cpn compose` (step 3) shows it on any file; step 9 writes it down |
| 5 logs saved per run | step 4 |
| 6 driver never stale | step 2 (hash check) |
| 7 how the cpnpy code is produced, which files, any AI | rendered from the verified net, no model (step 9 writes it down); files reduced in step 6 |
| 8 run the generated file for any run, same every time | step 5; step 8 point 11 proves "same every time" with hashes |
| 9 compile/compose verified on an example | step 3 on the review file; step 8 on a never-seen file |
| 10 single state / single transition; user_code.py | `compose` output (step 3); user_code.py is cpnpy's JSON side file, gone by default (step 6) |
| 11 clean the output | step 6 |
| 12 where the per-stage log is | step 4 |
