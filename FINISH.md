# agentmd2cpn — steps 6 to 11: clean the output, the picture, the new file, README, playbook, commit

Continues `RESET_AND_FIX.md` after step 5. Same rules as before: one short Copilot prompt per step,
a check you run yourself, the grep that proves nothing is hardcoded, the original test files never
edited by Copilot. Three files in this document are mine and complete — the README (step 9), the
playbook skeleton (step 10) and the `ticket_agent` example (step 8); they are also delivered as
separate files `README.md`, `PLAYBOOK.md`, `ticket_agent.agent.md` so you can place them without
copying from here.

The grep, after every step (must print nothing):

```powershell
Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"
```

---

## Step 6 — less output (points 7 and 11)

Before the prompt, change three lines in `tests\test_cli.py` yourself — this original test pins the
output files, so it must say what the output should be now:

1. in `test_cli_end_to_end_with_reviewed_dsl`, replace the list
   `["agent_spec.yaml", "demo_cpn.py", "demo_cpn.json", "report.json", "pipeline.jsonl"]`
   with `["agent_spec.yaml", "demo_cpn.py", "REPORT.md", "pipeline.jsonl"]`;
2. in the same test, `rep["generated_file_reverified"]` → `rep["generated_file_reverified"] is None`;
3. add the line `assert not (tmp_path / "demo_cpn.json").exists() and not (tmp_path / "report.json").exists()`.

Then:

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; do not edit the original test files (test_cli, test_lock, test_report, test_selfcheck, test_emit, test_extract, test_ingest, test_dsl, test_templates, test_compile, test_compose, test_verify, test_verify_large, test_runtime, test_fastcpn, test_llm_client); the test files you created (test_drive, test_stage_commands, test_logs, test_generated_replay, and new ones) you may update to the current behaviour; run python -m pytest tests -q at the end and show the count.
In cli.py: the generated file is no longer run again on every pipeline run - add --reverify to run (default off; when on, run_generated as before and report it in the summary line; otherwise generated_file_reverified is None and the summary line omits it); report.json is no longer written (the dict run() returns stays as it is); cpnpy's JSON (<name>_cpn.json and its user_code file) is written only with a new flag --json. Update the module docstring and --help.
```

Check:

```powershell
agentmd2cpn examples\review\review.agent.md
Get-ChildItem out\review
```

→ the summary line ends at `dead=['merge_pr']`; the folder holds `agent_spec.yaml`, `REPORT.md`,
`review_cpn.py`, `review_cpn.png`, `pipeline.log`, `pipeline.jsonl`, `monitor.log`, `runs` and nothing
else (delete old files from earlier runs first if you want a clean listing: `Remove-Item out\review -Recurse`
and run again). The grep; the suite. Keep.

## Step 7 — a picture one can read (point 1)

cpnpy's own drawing writes every guard and arc expression; for a 16-rule file it is 28 000 pixels
wide. This replaces it with a picture that shows the marking: every rule's state, every action
coloured by whether it can fire, and the rules that refuse it.

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules or paths; do not edit the original test files (test_cli, test_lock, test_report, test_selfcheck, test_emit, test_extract, test_ingest, test_dsl, test_templates, test_compile, test_compose, test_verify, test_verify_large, test_runtime, test_fastcpn, test_llm_client); the test files you created (test_drive, test_stage_commands, test_logs, test_generated_replay, and new ones) you may update to the current behaviour; run python -m pytest tests -q at the end and show the count.
Rewrite emit_dot(cpn, marking, out, ctx=None, blame=None) in agentmd2cpn/runtime/emit.py using the graphviz package (already a dependency): one ellipse per place except ENV_* places, labelled with the place name and its tokens (state tokens without their 'R1:' prefix, a unit token as a bullet, "n tokens" when more than three); one rounded box per transition - green when it has a binding in this marking, red with "blocked by R1, R2" when it has none and those rules refuse it for every parameter choice (use Monitor.explain from runtime/monitor.py and _param_choices from verification/selfcheck.py), grey otherwise; one edge per place/transition pair (two-headed when arcs go both ways), none from RUNNING except to END; rankdir LR; node ids must not contain ':'. Pass ctx and the blame map from both call sites in cli.py (the pipeline's <name>_cpn.png at the initial marking, drive's marking.png at the marking after the last step). Tests in tests/test_picture.py.
```

Check:

```powershell
agentmd2cpn drive out\review read_diff --png --name after_read_diff
```

→ open the `marking.png` named at the end: `read_diff` and the actions the rules allow next are green,
`END` is red *blocked by R3, R4*, `merge_pr` red *blocked by R7*, the rule places show `RUN`, `NO_A`,
`C0`, …; the picture fits on a screen. The grep; the suite. Keep.

## Step 8 — a file the package has never seen (points 4, 9, 10)

Create `examples\ticket_agent\ticket_agent.agent.md` with exactly this content (it is delivered as a
file too; 26 lines):

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

Then the proof that the package does not know it — must print nothing:

```powershell
Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "ticket|refund|escalat|classify"
```

The expected reading of its eight rule sentences is the table in `PLAYBOOK.md` Part B (init,
precedence, exactly 1, end with two alternatives, forbid, forbid_when on a `day` parameter,
precedence, limit 2; the style sentence gives no rule). The first run is step 10 — it is part of the
playbook, so that its real output is recorded there.

## Step 9 — README

`README.md` (delivered with this document, 537 lines) replaces the current one entirely. It explains the
whole architecture from the file to the running monitor: the stages and their mechanism, the rule
language with all seventeen kinds, a worked example of one rule becoming a net and being fused,
what verification proves and how, reproducibility, the generated file, the workflow for a new
file, the layout, the limitations. Everything in it is written against the code as it is after
step 7.

Put the file in place (copy from GitHub, or paste the delivered file over `README.md`), then let
Copilot check it against the code — not rewrite it:

```text
README.md was just replaced. Do not rewrite it. Check it against the code and report every statement that does not match: run each command in § 1, § 2 and § 9 and compare with the text; compare the option table in § 2 with cli.py's argparse; compare the file list in § 3 with what a run writes; compare the layout in § 11 with the repository. Fix only mismatches, each as a minimal edit, and show me the diff. Then delete docs/ARCHITECTURE.md and docs/RUNBOOK.md if they exist (the README now holds that content) and grep README.md, docs/ and examples/ for "drive.py", "report.json", "_cpn.json", "user_code", "check.py" and fix any stale mention.
```

Read the diff. Accept wording fixes that follow from the code; reject anything that changes the
structure or adds text of its own.

## Step 10 — the playbook, run live

`PLAYBOOK.md` (delivered, goes to `docs\PLAYBOOK.md`) is complete except for the outputs: every
command is there, with what to look at and the expected facts; under each command a block
`<OUTPUT>` waits for the real output. Copilot runs the commands and pastes the outputs — nothing else:

```text
docs/PLAYBOOK.md contains commands followed by a code block holding the placeholder <OUTPUT>. For each, in order from top to bottom, run exactly that command (or those commands) in the terminal and replace the placeholder with the complete real output, verbatim - do not shorten, tidy, reorder or invent anything, and change no other text in the file. Part B calls the model once (B1); run it as written. If a command fails, stop and show me. In Part B, if the action ids the model chose differ from the ones written in the B4-B6 commands, substitute the real ids in those commands (and only there) before running them. Change no code and no test. When done, show me git diff --stat.
```

Then read `docs\PLAYBOOK.md` once, top to bottom, with the "What to look at" lines: every fact they
name must be in the output above them (14 markings, `merge_pr`, 8 of 8, `R6 | BLOCK …`,
`finished: False`, identical hashes, `reused, LLM not called`, …). For Part B compare the model's
reading with the table; if it differs, B3 is where it was corrected, and the playbook says so.

## Step 11 — final checks, commit, pull request

```powershell
python -m pytest tests -q
Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"
git status --short
```

→ all tests pass; the grep prints nothing; `git status` shows no `.env`, no `out/`, no `dist/`, no
`*.egg-info`, no `examples/orchestrator`. Then:

```powershell
git add agentmd2cpn tests examples docs README.md pyproject.toml .gitignore
git commit -m "agentmd2cpn: stage commands, drive with marking.png, per-stage logs per run, standalone replay, smaller output, README and playbook"
git push
```

Pull request title: `agentmd2cpn: .agent.md -> rule DSL -> verified CPN with runtime monitor`. Description:

```text
What this adds
- agentmd2cpn, a package that turns any markdown agent file (.Agent.md, *.agent.md, AGENTS.md, ...) into a typed rule DSL (model extraction, 3 samples, majority vote, every rule quoting the sentence it came from), compiles each rule into a cpnpy net, fuses them into one, verifies the result (full state space, or compositional for large files), checks that every rule enforces something, and emits a standalone, self-verifying <name>_cpn.py plus a runtime monitor.
- REPORT.md per run: what to review first, the verdict, conflicts as pairs of sentences with the dead end as the monitor sees it, actions that can never happen, one refused step per rule (replayed into monitor.log).
- Reproducibility by construction: a file is read by the model once per version (its hash); a reviewed spec is locked to that hash and reused; a changed file is detected and read again; everything after the YAML is deterministic.
- Commands: agentmd2cpn <file> (pipeline), ingest|extract|compile|compose|verify|check (one stage each), drive (a run on the net with rule-named decisions and a picture of the resulting marking); logs per stage and per run.
- README.md explains the architecture end to end; docs/PLAYBOOK.md is a complete run with real outputs on the locked example and on a file the package had never seen.

How to try it
pip install -e .[dev]; copy .env.example to .env (needed only the first time a file is read)
agentmd2cpn examples\review\review.agent.md
agentmd2cpn drive out\review read_diff approve_pr:test_status=failing run_tests post_review_summary --png
python out\review\review_cpn.py read_diff run_tests
python -m pytest tests -q

Architecture after nl2spec (Cosler et al., CAV 2023): the model proposes, the formal part verifies.
```

---

## Appendix — the files delivered with this document

- `README.md` → the repository root (step 9)
- `PLAYBOOK.md` → `docs\PLAYBOOK.md` (step 10)
- `ticket_agent.agent.md` → `examples\ticket_agent\ticket_agent.agent.md` (step 8)
