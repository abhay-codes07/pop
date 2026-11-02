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
