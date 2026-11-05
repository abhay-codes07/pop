# agentmd2cpn — the `Unknown condition name: project` error: what happened, the fix, then the run goes on

## What happened

The model read line 8 — *You update the dependencies of one project per run, in the order below.* — as
a rule: `scope: update_dependency is allowed only when project == 'one'`. All three samples did (so the
vote kept it), and none of them declared a parameter `project` on `update_dependency` (`params: {}`,
`data: {}`). The DSL requires every name in a condition to be a declared parameter of the action, so
the whole extraction was declared invalid and written to `out\deps_agent\agent_spec.yaml` as a draft.
Copilot's reading of the error is right.

Two causes, two different fixes:

1. **The model's reading.** That sentence describes the agent; the prompt says such sentences give no
   rule, the model made one anyway, and `'one'` as a parameter value shows it was guessing. Three
   samples at temperature 0.2 are not independent enough to cancel a systematic misreading — the vote
   removes noise, not bias. That is what *Review first* in the report is for. Nothing to change here.
2. **The pipeline's reaction — this is the bug.** One rule the DSL cannot accept killed the whole run,
   while a rule with an unknown kind or an ungrounded quote is rejected with a `REJECT` line and the
   run continues. A rule that fails the DSL's checks must be treated the same way: rejected with its
   reason, kept in the spec under `rejected:` for review, listed in the report's *Review first*, and
   the extraction goes on with the rules it can accept. The draft-and-stop path stays only for a spec
   the DSL cannot build at all (a type with duplicate values, say).

## The fix, test first

### `tests/test_reject.py` (25 lines, hash `3b851b07`; delivered as a file)

```python
"""A rule the DSL cannot accept is rejected with its reason and listed for review; it does not stop the extraction."""
import json

from agentmd2cpn.cli import run
from agentmd2cpn.extraction.extract import LLMExtractor, extract_spec
from agentmd2cpn.extraction.ingest import parse_markdown

MD = '---\nname: "t"\n---\n# T\n\nYou handle one project per run.\n\n- Start by reading the request.\n'
GOOD = {"kind": "init", "args": {"action": "read_request"}, "quote": "Start by reading the request.", "line_begin": 8, "line_end": 8}
BAD = {"kind": "scope", "args": {"action": "read_request", "where": "project == 'one'"}, "quote": "You handle one project per run.", "line_begin": 6, "line_end": 6}
ANSWER = json.dumps({"actions": [{"id": "read_request"}], "rules": [BAD, GOOD]})     # no sample declares a parameter `project`


def test_a_rule_the_dsl_cannot_accept_is_rejected_with_its_reason_not_fatal(tmp_path):
    md = tmp_path / "t.agent.md"
    md.write_text(MD, encoding="utf-8")
    spec = extract_spec(parse_markdown(md), "t", LLMExtractor(lambda p: ANSWER, samples=3), min_votes=2)
    assert [(r.id, r.kind) for r in spec.rules] == [("R1", "init")]                   # the good rule, numbered as if alone
    assert len(spec.rejected) == 1 and spec.rejected[0].line_begin == 6 and spec.rejected[0].votes == 3
    assert "project" in spec.rejected[0].reason and spec.actions[0].params == {}
    run(md, tmp_path / "out", complete=lambda p: ANSWER, png=False)                     # the pipeline goes on to the end
    report = (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    assert "## Review first" in report and "You handle one project per run." in report and "project" in report
    assert "## Verdict: CONSISTENT" in report
    assert "rejected:" in (tmp_path / "out" / "agent_spec.yaml").read_text(encoding="utf-8")
```

### One test of `tests/test_cli.py` changes (it pinned the old behaviour)

In `tests\test_cli.py`, replace the whole function `test_invalid_extraction_writes_a_draft_for_review`
(from its `def` line to the blank lines before the next `def`) with:

```python
def test_a_rule_the_dsl_cannot_accept_is_rejected_and_a_spec_it_cannot_build_is_a_draft(tmp_path):
    # a parameter with an unknown type is dropped and the rule that needed it is rejected for review; the run completes
    bad = json.dumps({"actions": [{"id": "edit_file", "params": {"path": "file path"}}],
                      "rules": [{"kind": "scope", "args": {"action": "edit_file", "where": "path == 'deploy'"},
                                 "quote": "Only modify files under src/.", "line_begin": 10, "line_end": 10}]})
    run(FIX / ".Agent.md", tmp_path, extractor="llm", samples=2, min_votes=1, complete=lambda p: bad)
    spec = yaml.safe_load((tmp_path / "agent_spec.yaml").read_text(encoding="utf-8"))
    assert spec["actions"][0]["params"] == {} and spec["rules"] == [] and spec["rejected"][0]["kind"] == "scope"
    assert "path" in spec["rejected"][0]["reason"] and (tmp_path / "a_cpn.py").exists()
    # a spec the DSL cannot build at all (here: a type with duplicate values) is written as a draft to fix by hand
    worse = json.dumps({"actions": [{"id": "edit_file", "params": {"path": "p"}}], "data": {"p": {"type": "enum", "values": ["a", "a"]}}, "rules": []})
    with pytest.raises(SystemExit) as e:
        run(FIX / ".Agent.md", tmp_path / "w", extractor="llm", samples=2, min_votes=1, complete=lambda p: worse)
    assert e.value.code == 2 and (tmp_path / "w" / "agent_spec.yaml").exists() and not (tmp_path / "w" / "a_cpn.py").exists()
```

```powershell
python -m pytest tests\test_reject.py tests\test_cli.py -q
```

→ 2 failed (the new test and the replaced one), the rest of test_cli passing.

### Copilot prompt

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules, tools or paths; do not edit the original test files (test_cli, test_lock, test_report, test_selfcheck, test_emit, test_extract, test_ingest, test_dsl, test_templates, test_compile, test_compose, test_verify, test_verify_large, test_runtime, test_fastcpn, test_llm_client, test_tools, test_vote, test_reread, test_reject); the test files you created you may update to the current behaviour; run python -m pytest tests -q at the end and show the count.
Make tests/test_reject.py and the changed test in tests/test_cli.py pass (read both first). A rule the DSL cannot accept must be rejected on its own, not fail the extraction:
1. dsl.py: a model Rejected(kind, args, quote="", line_begin=0, line_end=0, votes=0, reason) and AgentSpec.rejected: list[Rejected] = [] (proposed at extraction, not accepted - for review).
2. extraction/extract.py, ground_and_vote: after the minority DROP, settle the parameters once with all voted rules in force (keep a parameter only when its data type exists in data; a parameter with an unknown type is dropped); then validate each voted rule alone - AgentSpec(name, actions, data, roles, tools, rules=[that rule]) - and on a ValidationError append a Rejected with the error messages as reason (strip the "Value error, " prefix) and log `REJECT rule <kind> (<n>/<samples> votes, line <l>): <english> - <reason>`; accepted rules get ids R1, R2, ... in order, so a rejected rule takes no id; then settle the parameters again with the accepted rules only (a parameter only a rejected rule needed goes, with a data type nothing uses). The spec gets `rejected`. The draft-and-exit-2 path stays for a spec that still fails to validate.
3. runtime/emit.py: write_spec omits `rejected` when it is empty. verification/report.py: in "Review first" (also when only rejected rules exist), first a block "Rules the model proposed that the DSL could not accept (rejected, not enforced - a misreading, or a rule to add by hand):" with one line per rejected rule: `- line <l>: "<quote>"  --  <kind>: <english>  [<votes> votes]  --  <reason>`.
```

→ `python -m pytest tests -q`: the previous count + 1, all passing.

## Then the run goes on

```powershell
agentmd2cpn examples\deps_agent\deps_agent.agent.md --extractor llm
```

(`--extractor llm`: the failed run left a draft in `out\deps_agent` for the same hash, and `auto`
would try to reuse it.) With the reading your model gave:

```text
extract  | REJECT rule scope (3/3 votes, line 8): update_dependency is allowed only when project == 'one' - condition "project == 'one'": unknown name 'project' (allowed: ['role'])
extract  | R1 init: first action must be read_manifest
extract  | R2 precedence: update_dependency only after run_tests
extract  | R3 precedence: update_dependency only after read_changelog
extract  | R4 response: after update_dependency, run_tests must eventually follow
extract  | R5 end: last action must be write_update_summary or report_nothing_to_update
extract  | R6 forbid: edit_lockfile is never allowed
extract  | R7 forbid: publish_package is never allowed
extract  | R8 limit: retry_failed_install at most 2 times
extract  | tool: ... (nine lines, as before)
verify   | state space: 43 markings, 129 arcs
verify   | dead transitions (can never fire): ['edit_lockfile', 'publish_package']
verify   | tools: 5 declared (read, search, web/fetch, execute, edit); 8 of 9 actions are tool calls, 1 need no tool
verify   | with the declared tools only, every rule can be satisfied
examples\deps_agent\deps_agent.agent.md: 8 rules, 43 reachable markings, consistent=True, traps=0, dead=['edit_lockfile', 'publish_package']
```

and at the top of `REPORT.md`:

```text
## Review first

Rules the model proposed that the DSL could not accept (rejected, not enforced - a misreading, or a rule to add by hand):

- line 8: "You update the dependencies of one project per run, in the order below."  --  scope: update_dependency is allowed only when project == 'one'  [3 votes]  --  condition "project == 'one'": unknown name 'project' (allowed: ['role'])
```

The reviewer's decision here is "no rule" (line 8 describes the agent), so nothing to add by hand; the
model's eight real rules match the table in LIVE_RUN.md § 2 one for one (the first three action ids
were cut off in your screenshot — read them from your `extract` lines). One more thing the model
decided, and it is a fair reading: `update_dependency -> edit` — updating a dependency as editing the
manifest, rather than `execute` (an install command). Either name is declared, so the verdict is the
same; if you prefer `execute`, that is a one-word edit in `agent_spec.yaml` before locking.

Continue at LIVE_RUN.md § 2 *Review, then lock*, with your ids:

```powershell
agentmd2cpn examples\deps_agent\deps_agent.agent.md --extractor yaml:out\deps_agent\agent_spec.yaml --lock
agentmd2cpn tools examples\deps_agent\deps_agent.agent.md
agentmd2cpn drive out\deps_agent read_manifest update_dependency run_tests read_changelog update_dependency write_update_summary --png --name early_update
agentmd2cpn drive out\deps_agent read_manifest run_tests read_changelog update_dependency run_tests write_update_summary --name one_update
python out\deps_agent\deps_agent_cpn.py read_manifest retry_failed_install retry_failed_install retry_failed_install edit_lockfile report_nothing_to_update
python out\deps_agent\deps_agent_cpn.py
```

(`read_manifest`, `run_tests`, `read_changelog`: whatever your extract lines named them.) Then § 3
and § 4 as written, with your ids.

## If you would rather not change code now

The by-hand path already works: open `out\deps_agent\agent_spec.yaml`, delete the rule whose `where`
is `project == 'one'` (the whole `- id: R1` block), and run
`agentmd2cpn examples\deps_agent\deps_agent.agent.md --extractor yaml:out\deps_agent\agent_spec.yaml`.
The remaining ids are then R2…R9; the rest is identical. But any new file can trip the same way, and
your mentor will run new files — the fix is the thing to have.
