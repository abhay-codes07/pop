# Notes

**Author:** abhay-codes07

That is the edge case, and the extraction itself is right: `AgentSpec(... actions=[], data={}, roles=[], rules=[])` — the model found neither an obligation nor a concrete action in the file, which is the correct reading of it. The crash is in `composition_check`: it builds random traces by picking from the action list, and the list is empty. Fix it test-first (I applied the same change to the reference implementation: all tests pass and the file then runs through).

1. Append to `tests/test_verify.py` (4 lines, two blank lines before it):

```python
def test_composition_check_with_no_actions():
    from agentmd2cpn.dsl import AgentSpec
    from agentmd2cpn.verify import composition_check
    assert composition_check(AgentSpec(name="t", actions=[], rules=[]), samples=5) == 0
```

2. Copilot:

```text
One change in agentmd2cpn/verify.py so that the new test in tests/test_verify.py passes and all other tests still pass; do not touch tests/. At the start of composition_check, before the random generator is created: if the spec has no actions, log "composition check skipped: no actions" at info level with stage verify and return 0. Nothing else changes. Then run python -m pytest tests -q and paste the full output; it must say 53 passed.
```

3. Rerun — the draft was written before the crash, so no LLM call is needed:

```powershell
python -m agentmd2cpn examples/universal/universal.agent.md -o out/universal --extractor yaml:out/universal/agent_spec.yaml
```

Expected (my run on an empty spec):

---

## Fix B — append to `tests/test_verify_large.py`

```python
def test_explore_handles_phase_sequence():
    s = spec([{"kind": "phase_sequence", "args": {"phases": [{"id": "p1", "allows": ["setup"]}, {"id": "p2", "allows": ["work"]}]}}])
    frag, cpn, m0, ctx = build(s)
    full = state_space_check(cpn, m0, ctx, quiet=True)
    ex = explore(cpn, m0, ctx, dict(frag.blame.get(END, [])), max_markings=1000)
    assert ex["complete"] and ex["done_reachable"] and ex["states"] == full["states"] and ex["trap_states"] == full["trap_states"] == 0
```

Copilot:

```text
One change in agentmd2cpn/verify_large.py so that the new test in tests/test_verify_large.py passes and all other tests still pass; do not touch tests/. In explore(), the inner function unsatisfied(mk) builds its binding only from places whose name starts with "S_" (variable "s_" + the rest of the name). Extend it: for every place that holds at least one token, a name starting with "S_" binds "s_" + name[2:] and a name starting with "PHASE_" binds "ph_" + name[6:], both to the first token's value. Nothing else changes; explore stays self-contained because it is copied into the generated file with inspect.getsource. Then run python -m pytest tests -q and paste the output; it must say 54 passed.
```

Then the file. Save it from SiemensGPT as `examples\expert\expert.agent.md` (21 lines, exactly as shown — front matter, heading, the two-sentence paragraph, `## Workflow` with the five bold numbered steps, `## Guidelines` with three bullets). Check:

```powershell
python -c "from pathlib import Path; from agentmd2cpn.ingest import parse_markdown; d=parse_markdown(Path('examples/expert/expert.agent.md')); print(len(d.segments), d.code_spans)"
python -m agentmd2cpn examples/expert/expert.agent.md -o out/expert
```

Expected ingest: `14 []` — the bold markers are stripped, the paragraph splits into 2 sentences, step 2 splits into 2 (Plan: Outline a step-by-step approach. / For multi-step tasks, create a clear plan.), 3 headings, 3 guidelines.

---

## Fix C — grounding for quotes that span list items

Test-first; fails on the current code, passes after a 3-line change in the reference implementation. Append to `tests/test_ingest.py`:

```python
def test_quote_spanning_list_items_is_grounded(tmp_path):
    from agentmd2cpn.ingest import parse_markdown, quote_is_grounded
    p = tmp_path / "a.md"
    p.write_text("# T\n\n## Steps\n1. **Analyze:** Parse the input.\n2. **Plan:** Outline a plan.\n", encoding="utf-8")
    doc = parse_markdown(p)
    assert quote_is_grounded(doc, "Analyze: Parse the input. Plan: Outline a plan.", 4, 5)
    assert not quote_is_grounded(doc, "Analyze: Parse the input. Plan: Deliver the plan.", 4, 5)
```

Copilot:

```text
One change in agentmd2cpn/ingest.py so that the new test in tests/test_ingest.py passes and all other tests still pass; do not touch tests/. quote_is_grounded(doc, quote, line_begin, line_end) currently checks the normalised quote against the normalised text of the cited lines of the file. Add a second window: the texts of doc.segments whose file equals doc.file and whose line_begin >= line_begin and line_end <= line_end, joined with single spaces (this is what the extractor was shown: list markers and markdown stripped). The quote is grounded if it is non-empty after normalisation and is a substring of either normalised window. Add one sentence to the docstring. Nothing else changes. Then run python -m pytest tests -q and paste the output; it must say 55 passed.
```

(55 = 52 + no-actions + phase explore + this one. Make sure fix B is in — the phase rule will now be compiled, and without fix B the generated file's self-check fails on `ph_R5`.)

Rerun live: `python -m agentmd2cpn examples/expert/expert.agent.md -o out/expert`. I replayed your three samples with the fix on the reference implementation (same action names as in your log): expected R1–R3 precedence (3 votes), R4 require (3 votes), R5 phase_sequence (2 votes) → 5 rules, 80 reachable markings, consistent=True, traps=0, dead=[], witness `enter_plan -> create_plan -> enter_execute -> enter_verify -> enter_deliver`. The model may vote slightly differently this time; the thing to check is that the phase rule is in.

### Review (edit `out/expert/agent_spec.yaml`, rerun with `--extractor yaml:`)

- **R4 require create_plan** — from *For multi-step tasks, create a clear plan*: conditional → `modality: should`.
- **R1–R3** — line 19 is under Guidelines, but it is an explicit ordering ("check files, search data, and reason through edge cases before asking") found 3/3. Keeping them is the literal reading; setting them to `should` is the "guidelines are advice" reading, consistent with what you did for the orchestrator's manner sentences. Either is defensible — decide once and say why. Keep: 5 rules, 48 reachable markings; should: 5 rules, 6 reachable markings. Witness either way `enter_plan -> enter_execute -> enter_verify -> enter_deliver` (nothing requires a step to happen; only the order is constrained).
- **R5** stays as extracted.

### Drive it

Verified outputs; the `enter_<phase>` transitions are the phase changes an agent or its harness signals explicitly — that is how `phase_sequence` works at runtime.

```powershell
python examples/drive.py out/expert analyze_requirements enter_plan create_plan enter_execute execute_work enter_verify verify_results enter_deliver deliver_output
```

→ every line ALLOW, ALLOW END, DONE: all rules satisfied.

```powershell
python examples/drive.py out/expert create_plan ask_user check_files search_data reason_edge_cases ask_user analyze_requirements enter_plan enter_execute execute_work deliver_output enter_verify enter_deliver deliver_output
```

→ R5 | BLOCK create_plan (still in the analyze phase), R1, R2, R3 | BLOCK ask_user (nothing checked yet), then ALLOWs, R5 | BLOCK deliver_output (in the execute phase), ALLOWs, ALLOW END, DONE — if you kept R1–R3; with them as `should` the ask_user line is an ALLOW.

Then copy `out\expert\agent_spec.yaml` to `examples\expert\spec.yaml`, and this file becomes example number four, with its own lesson for the log: problem (9), a correct multi-line extraction was rejected by the grounding check; fixed test-first.
