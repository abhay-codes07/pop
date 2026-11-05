# agentmd2cpn — the agent file's tools, through the whole pipeline

What was wrong: `tools:` in the front matter of an `.agent.md` was parsed into `Document.front_matter`
and never used. The actions in the DSL had no connection to the tools the agent is allowed to use, so
nothing in the pipeline could ask the one question the tools make answerable: *can this agent, with
these tools, obey these rules at all?*

What this adds, in the order the pipeline runs:

| stage | before | after |
|---|---|---|
| ingest | front matter kept as text | `Document.tools` — the `tools:` list (`agentmd2cpn ingest` prints it) |
| extract | the model names actions | the prompt names the declared tools; the model sets `tool` on every action to one of them, to `"none"` (the action needs no tool: a decision, a message to the user, an event), or to `null` (it needs a tool and none of the declared ones performs it); a tool outside the list is ignored with a warning; each action gets the value most samples chose |
| DSL | — | `Action.tool`, `AgentSpec.tools`, the constant `NO_TOOL = "none"`; an action's tool must be in the declared list, `"none"`, or `null` |
| compile / compose | — | `compose` shows `[tool read]` next to each transition that is a tool call |
| verify | — | the **tool check**: with the declared tools only (every action that needs an undeclared tool forbidden; actions that need no tool stay free), can every rule still be satisfied? If not, which rules, and which actions they need that no declared tool performs |
| report | — | a **Tools** section: the mapping, tools shared by several actions, declared tools no action uses, the feasibility verdict with the sentences |
| emit | — | `TOOLS` and `DECLARED_TOOLS` in `<name>_cpn.py` |
| commands | — | `agentmd2cpn tools <file.md>` prints the mapping and the verdict |

No tool name is known to the package. The names come from the file; the mapping is proposed by the
model from the file's own list, reviewed, and locked with the rules. Hooks and a gate on real tool
calls are a later step (`TOOLS_AND_HOOKS.md`); this is the part the pipeline needed first.

---

## U1 — the tests, first

Create `tests/test_tools.py` (84 lines, hash `ce43f236`; it is also delivered as a file). Then:

```powershell
python -m pytest tests\test_tools.py -q
```

→ it must **fail** (`ModuleNotFoundError: No module named 'agentmd2cpn.verification.tools'`). That is
the point: the tests say what the feature is before any code exists.

```python
import json

import pytest
import yaml

from agentmd2cpn.cli import main, run
from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.extraction.extract import LLMExtractor, extract_spec
from agentmd2cpn.extraction.ingest import parse_markdown
from agentmd2cpn.verification.tools import tool_check

MD = ('---\nname: "t"\ntools: [read, comment]\n---\n# T\n\n- Start by reading the request.\n- Run the checks before answering.\n'
      '- Answer exactly once per run.\n')
ACTS = [{"id": "read_request", "tool": "read"}, {"id": "run_checks"}, {"id": "answer", "tool": "comment"}, {"id": "decide", "tool": "none"}]
RULES = [{"kind": "init", "args": {"action": "read_request"}, "quote": "Start by reading the request.", "line_begin": 7, "line_end": 7},
         {"kind": "precedence", "args": {"first": "run_checks", "then": "answer"}, "quote": "Run the checks before answering.", "line_begin": 8, "line_end": 8},
         {"kind": "exactly", "args": {"action": "answer", "n": 1}, "quote": "Answer exactly once per run.", "line_begin": 9, "line_end": 9}]


def _answer(acts=ACTS):
    return json.dumps({"actions": acts, "rules": RULES})


def test_ingest_reads_the_declared_tools(tmp_path):
    md = tmp_path / "t.agent.md"
    md.write_text(MD, encoding="utf-8")
    assert parse_markdown(md).tools == ["read", "comment"]
    md.write_text("# No front matter\n\n- Never do a.\n", encoding="utf-8")
    assert parse_markdown(md).tools == []


def test_extraction_maps_actions_to_declared_tools_by_vote_and_drops_undeclared_ones(tmp_path):
    md = tmp_path / "t.agent.md"
    md.write_text(MD, encoding="utf-8")
    doc = parse_markdown(md)
    answers = iter([_answer(), _answer([{"id": "read_request", "tool": "read"}, {"id": "run_checks", "tool": "shell"}, {"id": "answer", "tool": "comment"}, {"id": "decide", "tool": "none"}]),
                    _answer([{"id": "read_request", "tool": "read"}, {"id": "run_checks"}, {"id": "answer"}, {"id": "decide"}])])
    spec = extract_spec(doc, "t", LLMExtractor(lambda p: next(answers), samples=3), min_votes=2)
    assert spec.tools == ["read", "comment"]
    assert {a.id: a.tool for a in spec.actions} == {"read_request": "read", "run_checks": None, "answer": "comment", "decide": "none"}   # 'shell' is not declared
    prompt = LLMExtractor(lambda p: "", samples=1).prompt(doc)
    assert "read, comment" in prompt and "`tool`" in prompt and '"none"' in prompt


def test_a_tool_outside_the_declared_list_is_rejected_by_the_dsl():
    with pytest.raises(ValueError):
        AgentSpec.model_validate({"name": "t", "tools": ["read"], "actions": [{"id": "a", "tool": "edit"}], "rules": []})
    AgentSpec.model_validate({"name": "t", "tools": ["read"], "actions": [{"id": "a", "tool": "none"}], "rules": []})     # "none" = needs no tool
    AgentSpec.model_validate({"name": "t", "actions": [{"id": "a", "tool": "edit"}], "rules": []})          # no declared list: nothing to check


def test_tool_check_tells_whether_the_rules_can_be_met_with_the_declared_tools():
    spec = AgentSpec.model_validate({"name": "t", "tools": ["read", "comment"], "actions": ACTS,
                                     "rules": [dict(r, id=f"R{i}") for i, r in enumerate(RULES, 1)]})
    t = tool_check(spec)
    assert t["unmapped"] == ["run_checks"] and t["free"] == ["decide"] and t["unused"] == [] and t["shared"] == {}
    assert t["feasible"] is False and t["alone"] == []                                              # each rule alone is fine ...
    assert sorted(b["rule"] for b in t["blocked"]) == ["R2", "R3"]                                   # ... together they need the checks
    assert next(b for b in t["blocked"] if b["rule"] == "R2")["needs"] == ["run_checks"]
    spec2 = AgentSpec.model_validate({"name": "t", "tools": ["read"], "actions": ACTS[:1] + [{"id": "run_checks"}, {"id": "answer"}, {"id": "decide", "tool": "none"}],
                                      "rules": [dict(RULES[2], id="R1")]})
    t2 = tool_check(spec2)
    assert t2["feasible"] is False and t2["alone"] == [{"rule": "R1", "needs": ["answer"]}]           # R1 needs answer, which has no tool
    spec.actions[1].tool = "read"                                                                   # now every action has a tool
    t = tool_check(spec)
    assert t["feasible"] is True and t["shared"] == {"read": ["read_request", "run_checks"]}
    assert tool_check(AgentSpec.model_validate({"name": "n", "actions": [{"id": "a"}], "rules": []})) is None   # no tools declared


def test_tools_reach_the_report_the_tools_command_and_the_generated_file(tmp_path, capsys):
    md = tmp_path / "t.agent.md"
    md.write_text(MD, encoding="utf-8")
    spec = {"name": "t", "tools": ["read", "comment"], "actions": ACTS, "rules": [dict(r, id=f"R{i}") for i, r in enumerate(RULES, 1)]}
    y = tmp_path / "spec.yaml"
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    run(md, tmp_path / "out", extractor=f"yaml:{y}", png=False)
    report = (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    assert "## Tools" in report and "- run_checks -> NO DECLARED TOOL" in report and "- decide -> no tool needed" in report
    assert "needs run_checks, which no declared tool performs" in report
    gen = (tmp_path / "out" / "t_cpn.py").read_text(encoding="utf-8")
    assert "TOOLS = {'read_request': 'read', 'run_checks': None, 'answer': 'comment', 'decide': 'none'}" in gen and "DECLARED_TOOLS = ['read', 'comment']" in gen
    main(["tools", str(md), "-o", str(tmp_path / "out")])
    out = capsys.readouterr().out
    assert "declared tools: read, comment" in out and "- answer -> comment" in out and "removing any ONE of R2, R3" in out
```

## U2 — the implementation (Copilot)

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules, tools or paths; do not edit tests/test_tools.py or any original test file; the test files you created you may update to the current behaviour; run python -m pytest tests -q at the end and show the count.
Make tests/test_tools.py pass (read it first; it defines the behaviour). The changes, by module:
1. dsl.py: a constant NO_TOOL = "none"; Action gets `tool: str | None = None` - a declared tool, NO_TOOL (the action needs no tool), or None (it needs a tool and none declared performs it); AgentSpec gets `tools: list[str] = []` (the tools the agent file declares). In AgentSpec's validator: when `tools` is non-empty, an action whose tool is neither None, nor NO_TOOL, nor in the list is a ValueError naming the action and the declared tools; when `tools` is empty nothing is checked (sub-specs built elsewhere must keep working).
2. extraction/ingest.py: Document gets `tools: list[str]`; when the front matter is parsed, read its `tools` entry with yaml.safe_load - a YAML list, or a comma-separated string - into a list of distinct stripped names; empty when absent, not a list, or not valid YAML.
3. extraction/prompt/extraction.yaml and extraction/extract.py: add a placeholder {tools} to the prompt right before the {known} line, filled, when the document declares tools, with: "- The agent may use only these tools: <names>. For every action set `tool` to the one tool from this list that performs it; to \"none\" when the action needs no tool (a decision, a message to the user, an event that happens to the agent); to null when it needs a tool and none of the listed tools performs it." - and with "" otherwise. The JSON schema shown to the model already includes `tool` once Action has the field. In ground_and_vote: count, per action, which value each sample chose (a declared tool or "none"); a tool that is not declared is ignored with a warning naming the action, the tool and the declared list; each action gets the value most samples chose (ties broken by name), or None; log `tool: <action> -> <value> (k/n samples)`; the spec gets `tools` = the document's list.
4. New module agentmd2cpn/verification/tools.py with tool_check(spec, mode="enforce", max_markings=5000) -> dict | None and tool_lines(spec, result, by_id=None) -> list[str]. tool_check returns None when the spec declares no tools. Otherwise: mapping {action: tool}; unmapped = actions whose tool is None (they need a tool that no declared tool provides); free = actions whose tool is NO_TOOL; unused = declared tools no action has; shared = {tool: [actions]} for tools with more than one action. Feasibility: the spec's compiled rules plus one forbid rule per unmapped action (ids T1, T2, ...: an action that needs an undeclared tool cannot happen; free actions stay possible), explored with verification.verify_large.explore (bounded, max_markings): feasible = DONE reachable, complete, witness. When not feasible: `alone` = the rules infeasible even on their own with those forbids, each with `needs` = the unmapped actions named in its args; if `alone` is empty and there are at most 20 compiled rules, `blocked` = the culprits (rules whose removal alone makes the rest feasible), each with its `needs`. Log one line at stage verify with the counts and the verdict (a warning when not feasible); build sub-nets with the compile/compose loggers disabled as verification/report.py does in _quiet(). tool_lines renders: "declared tools: ...", one line per action - `- a -> tool`, `- a -> no tool needed (a decision, a message, an event)`, or `- a -> NO DECLARED TOOL performs it` - the shared and unused lines when any, then the verdict: feasible -> "With the declared tools only (an action that needs an undeclared tool cannot happen) every rule can be satisfied; a complete run: ..." (with "(bounded exploration)" when not complete); infeasible with `alone` -> "... these rules cannot be satisfied:" and one line per rule `- R3 (line 14: "quote") - needs x, y`; infeasible with culprits -> "... the rules cannot all be satisfied; removing any ONE of R2, R3 would repair it:" and one line per culprit with `- needs x, which no declared tool performs` or `- depends on the rules above through actions no declared tool performs`.
5. cli.py: in the verify stage, after the composition check, `res["tools"] = tool_check(spec, mode)`; a stage command `tools` (same options as extract) that prints tool_lines, or "the file declares no tools (no `tools:` in its front matter)"; in the compose command show `[tool <name>]` after each transition that is a tool call.
6. verification/report.py: after the "## Actions" section, when res["tools"] is set, a section "## Tools - what the file declares, and whether the rules can be met with them" with tool_lines.
7. runtime/emit.py: after ACTIONS in the generated file, `TOOLS = {action: tool}` and `DECLARED_TOOLS = [...]`, rendered from the spec.
Update docstrings and --help.
```

→ `python -m pytest tests -q`: the previous count + 5, all passing; the grep
(`Select-String -CaseSensitive -Path agentmd2cpn/*.py, agentmd2cpn/*/*.py -Pattern "read_diff|deploy_to_|retry_deployment|approve_pr|publish_change|ticket|review_agent|release_agent"`)
prints nothing. If Copilot's `tools.py` does not pass the tests after one correction round, the
appendix has the reference implementation (125 lines) — paste it over Copilot's.

## U3 — the real run: the review agent with its tools

The review file declares `tools: [read, git, comment]`. Its locked spec predates the tools, so this
run reads the file with the model once more (3 samples, with the tools in the prompt):

```powershell
agentmd2cpn examples\review\review.agent.md --extractor llm -o out\review_tools --no-png
agentmd2cpn tools examples\review\review.agent.md -o out\review_tools
```

What to expect — the mapping is the model's reading, so judge it, do not assume it:

```text
declared tools: read, git, comment

- read_diff -> read
- run_tests -> NO DECLARED TOOL performs it
- post_review_summary -> comment
- approve_pr -> git            (or comment - either is defensible)
- request_changes -> comment
- merge_pr -> git
- find_security_vulnerability -> no tool needed (a decision, a message, an event)
- create_ticket -> NO DECLARED TOOL performs it
- edit_files_pr_branch -> NO DECLARED TOOL performs it

tools shared by several actions (a gate on the tool name alone cannot tell them apart): git: approve_pr, merge_pr; comment: post_review_summary, request_changes

With the declared tools only (an action that needs an undeclared tool cannot happen) the rules cannot all be satisfied; removing any ONE of R2, R3 would repair it:
- R2 (line 14: "Run the automated test suite before posting any comments.") - needs run_tests, which no declared tool performs
- R3 (line 15: "Post the review summary exactly once per run.") - depends on the rules above through actions no declared tool performs
```

This is a real finding about the example: the file tells the agent to run the test suite before
commenting and to post exactly one summary, but gives it no tool that runs tests — with `read`, `git`
and `comment` the rules cannot all be obeyed. Nothing in the net changed (still 8 rules, 14
markings, consistent); the tools add a second question, and here the answer is no.

If the model mapped something you disagree with (for instance `run_tests -> git`), edit
`out\review_tools\agent_spec.yaml` and rerun with `--extractor yaml:out\review_tools\agent_spec.yaml`.
When the mapping is right, lock it — this rewrites `examples\review\spec.yaml` with the tools:

```powershell
agentmd2cpn examples\review\review.agent.md --extractor yaml:out\review_tools\agent_spec.yaml --lock
```

Then every stage shows the tools:

```powershell
agentmd2cpn ingest  examples\review\review.agent.md      # "tools": ["read", "git", "comment"]
agentmd2cpn extract examples\review\review.agent.md      # tools: [...] and tool: on every action
agentmd2cpn compose examples\review\review.agent.md      # read_diff  [tool read] ...
agentmd2cpn examples\review\review.agent.md              # pipeline.log: verify | tools: 3 declared ...; REPORT.md: ## Tools
Select-String -Path out\review\review_cpn.py -Pattern "^TOOLS|^DECLARED_TOOLS"
```

## U4 — the ticket agent with real tools

Change one line of `examples\ticket_agent\ticket_agent.agent.md`: `tools: [read, reply, escalate]` →
`tools: [read, search, fetch]` (names from your *Configure Tools* list; VS Code no longer warns). Run it:

```powershell
agentmd2cpn examples\ticket_agent\ticket_agent.agent.md --extractor llm --no-png
agentmd2cpn tools examples\ticket_agent\ticket_agent.agent.md
```

Expected verdict (with `read_ticket -> read`, `classify_ticket -> no tool needed`, and the rest needing
a tool that is not declared):

```text
declared tools no action uses: search, fetch

With the declared tools only (an action that needs an undeclared tool cannot happen) these rules cannot be satisfied:
- R3 (line 14: "Reply to the customer exactly once per run.") - needs reply_to_customer
- R4 (line 15: "Finish by closing the ticket or escalating it.") - needs close_ticket, escalate_ticket
```

Which is exactly right: a Copilot agent whose tools read, search and fetch cannot reply, close or
escalate — those are not things its tools do. The sentences are fine as a policy; this agent cannot
carry it out. When the policy files come from other teams, this is the check that tells you whether
a given agent can obey a given policy, before the gate ever runs.

## U5 — README and playbook

Insert in `README.md`, after the table of § 7, a subsection:

```markdown
### Tools

An agent file declares the tools the agent may use (`tools:` in its front matter). They go through the
whole pipeline: `ingest` reads them; the extractor names them in the prompt and sets, for every action,
the one declared tool that performs it, *none* (a decision, a message, an event — no tool needed), or
*null* (it needs a tool and no declared one performs it); the mapping is kept
in `agent_spec.yaml` (`tools:`, `tool:` on each action), validated against the declared list, reviewed
and locked with the rules; `compose` shows it on the transitions; the generated file carries it as
`TOOLS`. The verifier then asks a second question: **with the declared tools only — every action that
needs an undeclared tool forbidden — can every rule still be satisfied?** `REPORT.md`'s *Tools* section (and
`agentmd2cpn tools <file>`) answers it with the mapping, the tools several actions share, the declared
tools no action uses, and either a complete run with those tools or the rules that cannot be met and
the actions they need that no declared tool performs — a policy may be sound and still impossible
for a particular agent.
```

and in the command table of § 2 a row for `agentmd2cpn tools <file.md>`. In `docs\PLAYBOOK.md`, after
A3, a step **A3b** with `agentmd2cpn tools examples\review\review.agent.md` and its real output, and
after B1 a step **B1b** with the same for the ticket agent. (Copilot can insert these; tell it to
change nothing else.)

---

## Appendix — reference `agentmd2cpn/verification/tools.py` (132 lines, hash `73b46e38`)

```python
"""The tools of the agent file against its rules.

An agent file declares the tools the agent may use (front matter `tools:`); the extractor maps every action to the
declared tool that performs it, or to none (a decision, a message, an event).  This module answers, for a spec that
declares tools: which action has which tool, which declared tools no action uses, which tools several actions share,
and - the check that matters - whether every rule can still be satisfied when the agent can only do what its tools
do (every action without a tool is forbidden).  Nothing here knows any tool or action name."""
from __future__ import annotations

from contextlib import contextmanager

from loguru import logger

from agentmd2cpn.compilation.compose import END
from agentmd2cpn.dsl import NO_TOOL, AgentSpec, Rule
from agentmd2cpn.verification.verify import build
from agentmd2cpn.verification.verify_large import explore


@contextmanager
def _quiet():
    for m in ("agentmd2cpn.compilation.compile", "agentmd2cpn.compilation.compose"):
        logger.disable(m)
    try:
        yield
    finally:
        for m in ("agentmd2cpn.compilation.compile", "agentmd2cpn.compilation.compose"):
            logger.enable(m)


def _with_tools_only(spec: AgentSpec, rules: list[Rule], unmapped: list[str]) -> AgentSpec:
    """`rules` plus one forbid per action that no declared tool performs."""
    extra = [Rule(id=f"T{i}", kind="forbid", args={"action": a}) for i, a in enumerate(unmapped, 1)]
    return AgentSpec(name=spec.name, actions=spec.actions, data=spec.data, roles=spec.roles, tools=spec.tools, rules=list(rules) + extra)


def _done_reachable(spec: AgentSpec, mode: str, max_markings: int) -> tuple[bool, bool, list | None]:
    with _quiet():
        frag, cpn, m0, ctx = build(spec, mode)
    ex = explore(cpn, m0, ctx, dict(frag.blame.get(END, [])), max_markings)
    return ex["done_reachable"], ex["complete"], ex["witness"]


def tool_check(spec: AgentSpec, mode: str = "enforce", max_markings: int = 5000) -> dict | None:
    """None when the file declares no tools.  Otherwise:
    {"declared": [...], "mapping": {action: tool | None}, "unmapped": [...], "unused": [...], "shared": {tool: [actions]},
     "feasible": bool, "complete": bool, "witness": [...] | None, "blocked": [{"rule": id, "needs": [actions]}]}"""
    if not spec.tools:
        return None
    mapping = {a.id: a.tool for a in spec.actions}
    unmapped = [a.id for a in spec.actions if a.tool is None]              # needs a tool that no declared tool provides
    free = [a.id for a in spec.actions if a.tool == NO_TOOL]               # needs no tool: always possible
    used = {a.tool for a in spec.actions if a.tool and a.tool != NO_TOOL}
    unused = [t for t in spec.tools if t not in used]
    shared = {t: [a.id for a in spec.actions if a.tool == t] for t in spec.tools}
    shared = {t: ids for t, ids in shared.items() if len(ids) > 1}
    compiled = [r for r in spec.rules if r.modality != "should"]
    feasible, complete, witness = _done_reachable(_with_tools_only(spec, compiled, unmapped), mode, max_markings)
    def needs_of(r: Rule) -> list[str]:
        return sorted({str(v) for v in r.args.values() if isinstance(v, str) and v in unmapped}
                      | {str(x) for v in r.args.values() if isinstance(v, list) for x in v if str(x) in unmapped})

    alone, blocked = [], []
    if not feasible:
        for r in compiled:                                   # rules that cannot be satisfied even on their own
            ok, _, _ = _done_reachable(_with_tools_only(spec, [r], unmapped), mode, max_markings)
            if not ok:
                alone.append({"rule": r.id, "needs": needs_of(r)})
        if not alone and len(compiled) <= 20:                # otherwise: culprits, rules whose removal alone repairs the rest
            for r in compiled:
                ok, _, _ = _done_reachable(_with_tools_only(spec, [x for x in compiled if x is not r], unmapped), mode, max_markings)
                if ok:
                    blocked.append({"rule": r.id, "needs": needs_of(r)})
    out = {"declared": list(spec.tools), "mapping": mapping, "unmapped": unmapped, "free": free, "unused": unused, "shared": shared,
           "feasible": feasible, "complete": complete, "witness": witness, "alone": alone, "blocked": blocked}
    lg = logger.bind(stage="verify")
    lg.info("tools: {} declared ({}); {} of {} actions are tool calls, {} need no tool{}", len(spec.tools), ", ".join(spec.tools),
            sum(1 for a in spec.actions if a.tool and a.tool != NO_TOOL), len(spec.actions), len(free),
            f"; no declared tool for {', '.join(unmapped)}" if unmapped else "")
    if feasible:
        lg.info("with the declared tools only, every rule can be satisfied{}", "" if complete else " (bounded exploration)")
    else:
        if alone:
            lg.warning("with the declared tools only, these rules cannot be satisfied: {}",
                       ", ".join(f"{b['rule']} (needs {', '.join(b['needs'])})" if b["needs"] else b["rule"] for b in alone))
        else:
            who = ", ".join(f"{b['rule']}" + (f" (needs {', '.join(b['needs'])})" if b["needs"] else "") for b in blocked)
            lg.warning("with the declared tools only, the rules cannot all be satisfied{}", f" - removing any one of {who} would repair it" if blocked else "")
    return out


def tool_lines(spec: AgentSpec, t: dict, by_id: dict | None = None) -> list[str]:
    """The tools section as lines of text (REPORT.md and the `tools` command print the same)."""
    by_id = by_id or {r.id: r for r in spec.rules}
    lines = [f"declared tools: {', '.join(t['declared'])}", ""]
    for a in spec.actions:
        if a.tool == NO_TOOL:
            lines.append(f"- {a.id} -> no tool needed (a decision, a message, an event)")
        elif a.tool:
            lines.append(f"- {a.id} -> {a.tool}")
        else:
            lines.append(f"- {a.id} -> NO DECLARED TOOL performs it")
    lines.append("")
    if t["shared"]:
        lines.append("tools shared by several actions (a gate on the tool name alone cannot tell them apart): "
                     + "; ".join(f"{tool}: {', '.join(ids)}" for tool, ids in t["shared"].items()))
    if t["unused"]:
        lines.append(f"declared tools no action uses: {', '.join(t['unused'])}")
    if t["shared"] or t["unused"]:
        lines.append("")
    if t["feasible"]:
        w = t["witness"] or []
        lines.append("With the declared tools only (an action that needs an undeclared tool cannot happen) every rule can be satisfied"
                     + ("" if t["complete"] else " (bounded exploration)") + "; a complete run: "
                     + (" -> ".join(str(x) for x in w) if w else "(the empty run)"))
    else:
        def cite(b):
            r = by_id.get(b["rule"])
            s = r.source if r else None
            return f"{b['rule']}" + (f" (line {s.line_begin}: \"{s.quote.strip()}\")" if s and s.quote else "")
        if t["alone"]:
            lines.append("With the declared tools only (an action that needs an undeclared tool cannot happen) these rules cannot be satisfied:")
            for b in t["alone"]:
                lines.append(f"- {cite(b)} - needs {', '.join(b['needs']) if b['needs'] else 'an action no declared tool performs'}")
        else:
            lines.append("With the declared tools only (an action that needs an undeclared tool cannot happen) the rules cannot all be satisfied"
                         + (f"; removing any ONE of {', '.join(b['rule'] for b in t['blocked'])} would repair it:" if t["blocked"]
                            else "; no single rule is the cause (a combination of rules needs actions that no declared tool performs)"))
            for b in t["blocked"]:
                needs = f" - needs {', '.join(b['needs'])}, which no declared tool performs" if b["needs"] else " - depends on the rules above through actions no declared tool performs"
                lines.append(f"- {cite(b)}{needs}")
    return lines
```
