# Stage commands — Copilot prompt

Rule: nothing in `agentmd2cpn/` may name any agent file's actions, rules or paths; reuse the existing functions; do not edit any existing test; add tests only in a new file `tests/test_stage_commands.py`; run `python -m pytest tests -q` at the end and show the count.

In `agentmd2cpn/cli.py` add six subcommands next to `run` and `drive`, each taking `<file.md>` and the same `-o`/`--extractor`/`--lock`/`--mode` options as `run` (`verify` and `check` also `--verify`), each printing only its own result to stdout, with the console logger at WARNING, and each reusing the extraction when the file is unchanged exactly as `run` does. To guarantee that, first move the body of `run()`'s extract stage (the auto/yaml/llm choice, `find_spec`, the draft on `ExtractionError`, `write_spec` to `out/agent_spec.yaml`, `--lock`) into a function `extract_stage(path, out, doc, extractor, min_votes, samples, complete, lock) -> (spec, note)` and make `run()` call it; the stage commands call ingest (`parse_markdown`) and `extract_stage` the same way. The commands:

- **ingest:** the Document as JSON (`dataclasses.asdict`, indent 2).
- **extract:** the YAML that was written to `out/<name>/agent_spec.yaml`.
- **compile:** for each fragment of `compile_spec(spec, mode)` that is a rule (skip `ACT_*`; for `ENV_*` print one line: the place and its tokens): the rule id, kind and `english(kind, args)`; each place with its kind, colour set and initial tokens; the automaton from `build_dfa(kind, args).complete(action ids)` as one line per state - the state, "(accepting)" if accepting, and for each target state the actions leading there; the transitions it contributes and its guard part; the number of arcs; its accepting predicate. `compile_spec` is in `compilation/compile.py`, `build_dfa` and `english` in `compilation/templates.py`.
- **compose:** `compose_sync(fragments, name)` from `compilation/compose.py`: the counts; every place with its initial tokens; every transition with its variables and full guard; the blame map as "action: R1, R2"; the accepting predicate; then `add_end()` and what it adds (RUNNING, END, DONE) with the new counts; then `lower()` and the cpnpy net's places/transitions/arcs counts and initial marking.
- **verify:** the same choice as `run` (`verification_method`, `state_space_check` or `verify_large`, `composition_check`): method, reachable markings (and "(bounded)" when not complete), DONE reachable, trap states, dead actions, shortest complete run, conflicts if any, number of composition traces.
- **check:** `enforcement_witnesses` from `verification/selfcheck.py`: for every compiled rule its id, kind, meaning, and either "after <path>: <step> -> BLOCKED by <id> alone" or that no witness was found within the explored markings.

When building sub-nets inside verify (`composition_check`, `verify_large`) disable the `agentmd2cpn.compilation.compile` and `agentmd2cpn.compilation.compose` loggers around the call, as `verification/report.py` does in `_quiet()`, so the output is not flooded. Update the module docstring and `--help`.

## Check

After it finishes (the review spec is locked, so none of these calls the model):

```powershell
agentmd2cpn ingest  examples\review\review.agent.md
agentmd2cpn extract examples\review\review.agent.md
agentmd2cpn compile examples\review\review.agent.md
agentmd2cpn compose examples\review\review.agent.md
agentmd2cpn verify  examples\review\review.agent.md
agentmd2cpn check   examples\review\review.agent.md
Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"
python -m pytest tests -q
```

---

## Follow-up — drive runs folder without `--png`

```text
In tests/test_drive.py the assertion at line 46 pinned the old behaviour (no runs folder without --png). Change only that assertion: without --png the drive's runs/<date>-drive folder must exist and contain run.log and monitor.log, and must not contain marking.png. Change nothing else; run python -m pytest tests -q and show the count.
```

→ `142 passed`. Then the step-4 checks:

```powershell
agentmd2cpn examples\review\review.agent.md
type out\review\pipeline.log
agentmd2cpn drive out\review read_diff run_tests --name two_steps
dir out\review\runs
dir out\review\runs\*-drive-two_steps
Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"
```

→ `pipeline.log` readable
