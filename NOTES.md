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
