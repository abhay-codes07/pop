# agentmd2cpn — the tools run: what it showed, what to decide, what to fix

The two screenshots of `agentmd2cpn tools examples\review\review.agent.md -o out\review_tools` show three
things, in this order of importance:

1. **The names are the file's own.** `read`, `git`, `comment` are the `tools: [read, git, comment]` line of
   `review.agent.md`'s front matter. The prompt hands the model that list and nothing else; the model
   chose among the names it was given (`none` is the one word the package knows — it means *needs no tool*).
   § 1 proves on your laptop that no tool name is in the package.
2. **The `tool:` line on every action is the part I added beyond what you asked.** You asked that the
   tools an `.agent.md` declares are captured and carried through the pipeline. I also had the model say,
   for every action, which declared tool performs it, and had the verifier ask whether the rules can be
   met with those tools. § 2 says what that is and is not, and leaves the decision to you.
3. **Three bugs in that run**, all in the vote and the `tools` command, fixed test-first in § 3:
   `Unknown condition name: edit_location` (the samples disagreed on a parameter's name and the run died),
   `create_security_ticket -> none (1/3 samples)` (a one-vote tool was accepted), and the last line
   *the file declares no tools* printed after a failed extraction (wrong, and the command should have
   stopped at the error).

Then § 4 (the review example's lock predates the tools), § 5 (README text), and FINISH steps 9–11.

---

## 1. No tool name in the package

```powershell
Get-ChildItem agentmd2cpn -Recurse -Include *.py,*.yaml,*.yml,*.md,*.txt,*.j2 | Select-String -CaseSensitive -Pattern "\b(git|comment|fetch|escalate|reply)\b"
```

→ prints nothing: the tool names of both examples do not occur anywhere in the package, not even in
a comment. (`read` is left out of the pattern only because it is also the name of a Petri-net arc type,
`FArc(type="read")`, in compile.py and compose.py.) The sentence that mentions tools in the prompt is
built at run time from the file:

```powershell
Select-String -Path agentmd2cpn\extraction\*.py, agentmd2cpn\extraction\prompt\* -Pattern "may use only these tools"
```

→ one line, `"- The agent may use only these tools: " + ", ".join(doc.tools) + ...` — the list is
`doc.tools`, read by `ingest` from the front matter of whatever file is run.

---

## 2. What you asked for, what was built — your decision

| | what it is | where it shows |
|---|---|---|
| **asked:** capture and carry | `Document.tools` → `AgentSpec.tools` → `agent_spec.yaml` (`tools:`), REPORT, `DECLARED_TOOLS` in the generated file, `agentmd2cpn tools` | every stage names the list |
| **added:** the per-action reading | the model sets `tool` on each action to one of the declared names, `none` (no tool needed) or `null` (needs a tool none declared performs); voted, reviewed and locked with the rules | `tool:` in `agent_spec.yaml`, `[tool x]` in `compose`, `TOOLS` in the generated file |
| **added:** the verdict | with the declared tools only, can every rule still be satisfied? which rules cannot, and which actions they need | REPORT *Tools* section, `agentmd2cpn tools`, one `verify` log line |

Why I added the two rows: a list that is only carried changes nothing in any stage — no place, no
guard, no verdict depends on it — and "use the tools in the entire pipeline" needs something that
depends on them. The smallest such thing is the per-action reading, and the verdict follows from it
in a few lines. I should have put that to you as a choice instead of building it; it is your call now.

What it is **not**: the *Tool mapping* box of your mentor's diagram. That box maps **real tool calls** —
the `toolName` and `toolArgs` a hook sees, built-in and MCP tools, e.g. `execute` with `pytest` in the
command = `run_tests` — onto transitions, so a gate can decide; it needs the real tool names and
argument shapes, and the policy format the other teams will provide. Nothing here decides how that is
done: the per-action `tool` is a reading of the file's own list, the same kind of reading as the
rules (text → DSL, three samples, majority, reviewed, locked). When the mapping research is done, it
sits on top of this field (`tool` + an argument condition → transition) or replaces it; either way
nothing in the net changes.

Two ways forward:

- **A — strip it back** to capture-and-carry: no `tool` on actions, no verdict; `tools` command and
  REPORT print the declared list. One Copilot prompt and a smaller `tests/test_tools.py` — say so and
  I deliver both.
- **B — keep it, fixed** (§ 3): a file without `tools:` is never asked about tools and carries no tool
  field anywhere; a tool needs a majority of the samples; a reviewed spec older than the tools is
  named, not judged; and the README calls it what it is (§ 5).

My recommendation is B: it is the only thing that makes the declared tools matter, it is reviewable
like everything else in `agent_spec.yaml`, and it does not pre-empt the mapping research. If you and
your mentor would rather wait for that research, A is one prompt away.

---

## 3. The fix (B), test first

### 3a. `tests/test_vote.py` (81 lines, hash `79435f8c`; delivered as a file too)

```python
"""What the vote makes of samples that disagree about an action's parameters, and about tools."""
import json

import yaml

from agentmd2cpn.cli import main, run
from agentmd2cpn.extraction.extract import LLMExtractor, extract_spec
from agentmd2cpn.extraction.ingest import parse_markdown
from agentmd2cpn.runtime.emit import write_spec

FRONT = '---\nname: "t"\n{line}\n---\n'
BODY = '# T\n\n- Start by reading the request.\n- Never edit outside the branch.\n'
RULES = [{"kind": "init", "args": {"action": "read_request"}, "quote": "Start by reading the request.", "line_begin": 7, "line_end": 7},
         {"kind": "scope", "args": {"action": "edit", "where": "location == 'branch'"}, "quote": "Never edit outside the branch.", "line_begin": 8, "line_end": 8}]
ENUM = {"type": "enum", "values": ["branch", "other"]}


def _sample(acts, rules=RULES, data=None):
    return json.dumps({"actions": acts, "data": data if data is not None else {"location_t": ENUM}, "rules": rules})


def _doc(tmp_path, line):
    md = tmp_path / "t.agent.md"
    md.write_text(FRONT.format(line=line) + BODY, encoding="utf-8")
    return parse_markdown(md)


def _extract(doc, answers):
    it = iter(answers)
    return extract_spec(doc, "t", LLMExtractor(lambda p: next(it), samples=3), min_votes=2)


def test_a_file_that_declares_no_tools_is_not_asked_about_them_and_gets_no_mapping(tmp_path):
    doc = _doc(tmp_path, 'description: "no tools here"')
    assert doc.tools == []
    prompt = LLMExtractor(lambda p: "", samples=1).prompt(doc)
    assert "`tool`" not in prompt and '"tool"' not in prompt                   # neither the sentence nor the schema field
    acts = [{"id": "read_request", "tool": "read"}, {"id": "edit", "params": {"location": "location_t"}, "tool": "git"}]
    spec = _extract(doc, [_sample(acts)] * 3)                                  # the model volunteers tools anyway
    assert spec.tools == [] and all(a.tool is None for a in spec.actions)
    write_spec(spec, tmp_path / "spec.yaml")
    text = (tmp_path / "spec.yaml").read_text(encoding="utf-8")
    assert "tool" not in text


def test_a_tool_needs_a_majority_of_the_samples(tmp_path):
    doc = _doc(tmp_path, "tools: [read, git]")
    edit = {"id": "edit", "params": {"location": "location_t"}}
    spec = _extract(doc, [_sample([{"id": "read_request", "tool": "read"}, dict(edit, tool="git")]),
                          _sample([{"id": "read_request", "tool": "read"}, dict(edit, tool="none")]),
                          _sample([{"id": "read_request", "tool": "read"}, edit])])
    assert {a.id: a.tool for a in spec.actions} == {"read_request": "read", "edit": None}   # git 1, none 1: no majority


def test_a_parameter_is_kept_by_majority_or_because_a_kept_rule_needs_it(tmp_path):
    doc = _doc(tmp_path, 'description: "x"')
    other = [dict(RULES[0]), dict(RULES[1], args={"action": "edit", "where": "scope == 'branch'"})]
    spec = _extract(doc, [_sample([{"id": "read_request"}, {"id": "edit", "params": {"scope": "scope_t"}}], other, {"scope_t": ENUM}),
                          _sample([{"id": "read_request"}, {"id": "edit", "params": {"location": "location_t"}}]),
                          _sample([{"id": "read_request"}, {"id": "edit", "params": {"location": "location_t"}}])])
    edit = next(a for a in spec.actions if a.id == "edit")
    assert edit.params == {"location": "location_t"} and set(spec.data) == {"location_t"}   # the minority name and its type are gone
    assert [r.kind for r in spec.rules] == ["init", "scope"] and spec.rules[1].args["where"] == "location == 'branch'"
    spec = _extract(doc, [_sample([{"id": "read_request"}, {"id": "edit", "params": {"location": "location_t"}}]),
                          _sample([{"id": "read_request"}, {"id": "edit"}]),
                          _sample([{"id": "read_request"}, {"id": "edit"}], [RULES[0]])])
    edit = next(a for a in spec.actions if a.id == "edit")
    assert edit.params == {"location": "location_t"}                           # declared once, but the kept rule (2 votes) needs it


def test_a_spec_made_before_the_tools_were_read_is_named_not_judged(tmp_path, capsys):
    doc = _doc(tmp_path, "tools: [read, git]")
    spec = {"name": "t", "actions": [{"id": "read_request"}, {"id": "edit", "params": {"location": "location_t"}}], "data": {"location_t": ENUM},
            "rules": [dict(r, id=f"R{i}") for i, r in enumerate(RULES, 1)]}                       # no tools: an older reviewed spec
    y = tmp_path / "old.yaml"
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    run(tmp_path / "t.agent.md", tmp_path / "out", extractor=f"yaml:{y}", png=False)
    report = (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    assert "## Tools" in report and "declares read, git but the spec in use carries no tool mapping" in report
    main(["tools", str(tmp_path / "t.agent.md"), "-o", str(tmp_path / "out"), "--extractor", f"yaml:{y}"])
    assert "carries no tool mapping" in capsys.readouterr().out
```

```powershell
python -m pytest tests\test_vote.py -q
```

→ **4 failed** before any change. The third test is the `edit_location` crash of your screenshot,
reduced to two names.

### 3b. Copilot prompt 1 — the vote

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules, tools or paths; do not edit the original test files (test_cli, test_lock, test_report, test_selfcheck, test_emit, test_extract, test_ingest, test_dsl, test_templates, test_compile, test_compose, test_verify, test_verify_large, test_runtime, test_fastcpn, test_llm_client, test_tools, test_vote); the test files you created you may update to the current behaviour; run python -m pytest tests -q at the end and show the count.
Make the first three tests of tests/test_vote.py pass (read the file first; the fourth comes next). The changes:
1. extraction/extract.py, prompt(): when the document declares no tools, remove the `tool` property from the Action schema shown to the model (the tools sentence is already empty then).
2. ground_and_vote: (a) when the document declares no tools, ignore every `tool` the samples return - no vote, no warning, no `tool:` log line, every action's tool stays None; (b) when it does, an action gets the tool most samples chose only if at least min_votes samples chose it; otherwise None and one warning `tool: <action> -> no majority (<value> <n>, ...) - left null (no declared tool performs it); review it`; (c) parameters: count per (action, parameter) how many samples declared it; an action keeps a parameter when at least min_votes samples declared it, or when a kept forbid_when/scope rule on that action names it in its condition (add dsl.condition_names(expr) -> set[str], the ast.Name ids of the expression, empty when it does not parse); a parameter's type is the one the first sample that declared it gave; a data type that no kept parameter uses is dropped.
3. runtime/emit.py: write_spec omits `tools` when it is empty and `tool` from every action when the spec declares no tools; the generated file has the TOOLS and DECLARED_TOOLS lines only when the spec declares tools.
```

→ `python -m pytest tests -q`: the previous count + 3 passing, one failing (`test_a_spec_made_before…`).

### 3c. Copilot prompt 2 — a spec older than the tools; the `tools` command

```text
Same rule as before (no agent file's names in agentmd2cpn/, original test files untouched, your own test files may be updated, show the pytest count).
Make the fourth test of tests/test_vote.py pass:
1. verification/tools.py: unmapped_note(declared: list[str], spec) -> str | None - None unless the file declares tools and the spec carries none; otherwise the sentence "the file declares <names> but the spec in use carries no tool mapping (it was made before tools were read): map them with the model (--extractor llm -o <another folder>), or add `tools:` and `tool:` to the spec by hand, review, then --lock".
2. cli.py: run passes doc.tools into the verify stage; the verify stage logs that note as a warning and keeps it in res["tools_note"]; verification/report.py prints it as the "## Tools" section when there is no tool check; the tools command prints the note first, else "the file declares no tools (no `tools:` in its front matter)", else the tool lines - with the tools logger disabled while tool_check runs, the lines carry the verdict; after a failed extraction the tools command stops with exit code 2 exactly like the pipeline, printing nothing more.
```

→ `python -m pytest tests -q`: the previous count + 4, all passing; the grep of § 1 prints nothing.

---

## 4. The review example — its lock predates the tools

The reviewed spec `examples\review\spec.yaml` was written before `tools:` was read, so it has no
mapping. After § 3:

```powershell
agentmd2cpn tools examples\review\review.agent.md
```

```text
the file declares read, git, comment but the spec in use carries no tool mapping (it was made before tools were read): map them with the model (--extractor llm -o <another folder>), or add `tools:` and `tool:` to the spec by hand, review, then --lock
```

Take the second way: it keeps the locked action names (the playbook's commands and outputs stay as
they are), and the mapping is yours to defend rather than the model's. Edit `examples\review\spec.yaml`:
add one line `tools: [read, git, comment]` right before `actions:`, and under each action id one
`tool:` line — with the ids of your spec; mine are:

```yaml
tools: [read, git, comment]
actions:
- id: read_diff
  tool: read
- id: run_tests
  tool: null          # nothing declared runs tests
- id: post_review_summary
  tool: comment
- id: approve_pr
  tool: comment       # an approval is a review comment; git cannot approve
- id: request_changes
  tool: comment
- id: merge_pr
  tool: git
- id: find_security_vulnerability
  tool: none          # a finding, not a tool call
- id: create_ticket
  tool: null          # no tracker among the declared tools
- id: edit_files_pr_branch
  tool: null          # no edit tool declared
```

(the other fields of each action stay as they are). Then lock it — this restamps the file's hash:

```powershell
agentmd2cpn examples\review\review.agent.md --extractor yaml:examples\review\spec.yaml --lock
agentmd2cpn tools examples\review\review.agent.md
```

```text
declared tools: read, git, comment

- read_diff -> read
- run_tests -> NO DECLARED TOOL performs it
- post_review_summary -> comment
- approve_pr -> comment
- request_changes -> comment
- merge_pr -> git
- find_security_vulnerability -> no tool needed (a decision, a message, an event)
- create_ticket -> NO DECLARED TOOL performs it
- edit_files_pr_branch -> NO DECLARED TOOL performs it

tools shared by several actions (a gate on the tool name alone cannot tell them apart): comment: post_review_summary, approve_pr, request_changes

With the declared tools only (an action that needs an undeclared tool cannot happen) the rules cannot all be satisfied; removing any ONE of R2, R3 would repair it:
- R2 (line 14: "Run the automated test suite before posting any comments.") - needs run_tests, which no declared tool performs
- R3 (line 15: "Post the review summary exactly once per run.") - depends on the rules above through actions no declared tool performs
```

(your line numbers). The pipeline run adds one `verify | tools: 3 declared …` line and the warning,
`REPORT.md` gets the *Tools* section with the same text, `compose` shows `[tool read]` etc., and the
generated file has `TOOLS` and `DECLARED_TOOLS`; the net itself is unchanged — 8 rules, 14 markings,
consistent, `merge_pr` dead. From now on every command on this file reuses the lock and never calls
the model.

If you would rather declare real Copilot tool names in `review.agent.md` (so VS Code stops warning),
change the front matter, then the same hand mapping with those names and the same `--lock`; the
verdict follows from what the names can do.

The ticket agent needs nothing: its spec was mapped by the model with `read, search, fetch` and locked
in B3. Run `agentmd2cpn tools examples\ticket_agent\ticket_agent.agent.md` once more; if the extract
log of a fresh run ever shows `-> no majority`, set that one `tool:` in the spec yourself and lock.

---

## 5. README — the *Tools* subsection (use this instead of the text in TOOLS_NOW U5)

Insert in `README.md` after the table of § 7:

```markdown
### Tools

An agent file may declare the tools the agent is allowed to use (`tools:` in its front matter). The
pipeline reads that list and nothing else about tools: a file without `tools:` is never asked about
them, and its DSL, report and generated file carry no tool field. For a file that declares tools, the
extractor passes the list to the model and asks, for every action, which declared tool performs it —
one of the names, *none* (a decision, a message, an event: no tool needed) or *null* (it needs a tool
that no declared tool performs). A name outside the list is ignored; a tool is kept only when most
samples agree on it. The result is a reading of the file, like the rules: it is in `agent_spec.yaml`
(`tools:`, and `tool:` on each action), reviewed, corrected by hand where the model is wrong, and
locked with the rules. `compose` shows it on the transitions; the generated file carries it as
`TOOLS` and `DECLARED_TOOLS`. The verifier then asks one more question: **with the declared tools
only — every action that needs an undeclared tool forbidden — can every rule still be satisfied?**
The *Tools* section of `REPORT.md` (and `agentmd2cpn tools <file>`) answers it: the mapping, the tools
several actions share, the declared tools no action uses, and either a complete run with those tools
or the rules that cannot be met and the actions they need. A reviewed spec written before the tools
were read is named as such, not judged.

This is a reading of the file's own list. It is not the mapping of real tool calls — a tool name with
its arguments, built-in or MCP — onto transitions, which a gate on hook events needs; that mapping,
and the hooks that use it, are the next step and build on this field.
```

and in the command table of § 2 the row `agentmd2cpn tools <file.md>` — *the declared tools, the
action → tool mapping, and whether the rules can be met with them*. The playbook steps A3b and B1b
from TOOLS_NOW U5 stay as described; their outputs are the ones above.

Then FINISH steps 9, 10 and 11 as written.
