# Robustness batch — one pass over every input shape, instead of one fix per file

Three new files found three defects (no actions; a phase rule in the bounded explorer; a quote
spanning list items). Each was found by a path no test covered. This batch replaces "wait for the
next file" with a sweep: every shape the DSL can express, every markdown shape I could think of and
every answer the LLM could give were pushed through the real pipeline here, everything that broke
was fixed, and each fix has a test. Numbers in this document are from that run.

## 1. What was swept

| sweep | cases | what was pushed through |
|---|---|---|
| DSL shapes | 168 | every rule kind alone and in every pair; params (enum/bool/int), roles, `should`, monitor mode, n = 0, contradictions, two phase rules, phases without `must_finish`, 0 rules, 0 actions, 1 action, 22 rules (compositional path) and the same forced full, odd names (dash, quotes, unicode, 60 chars, `s_x`, `end`), all through `cli.run` including the generated file's self-check |
| markdown shapes | 39 | empty file, front matter only, no front matter, headings only, CRLF, BOM, tabs, nested and ordered lists, mixed markers, tables, blockquotes, code fences (closed and unclosed), inline markup, raw HTML, reference links, unicode, abbreviations, no terminal period, 200-sentence lines, setext headings, rules, missing and self-importing `@file.md`, odd and broken front matter, task lists, escaped characters, Windows paths, 300 items |
| LLM answers | 55 | prose instead of JSON, a list, null, `{}`, extra fields, unknown kind, unknown modality, missing or renamed arguments, args not a dict, undeclared action, bad action id, duplicate ids, the same rule twice in one sample, n as "2" / "two" / −1 / missing / 10⁶, malformed phases (strings, no id, empty, duplicate ids), unknown parameter in a condition, a function call in a condition, a syntax error, a value outside the domain, undeclared or free-text parameter type, enum without / with duplicate / with numeric values, undeclared role, quote not in file / wrong lines / reversed lines / line 0 / beyond the file / empty / with markdown / different case / spanning lines, line numbers as strings or floats, confidence as text, missing quote, `"params": null`, actions as strings, rules as an object, an action named `end`, a self-precedence |
| Monitor misuse | 8 | unknown action, unknown parameter name, value outside the domain, omitted parameter, finish twice, action after finish, `END` by name, a numeric parameter |
| regression | 5 | release agent 2569, fixture 4, expert 48, universal 2, orchestrator 47 rules / 0 conflicts / 5002 — all unchanged after the batch |

Result before the batch: 9 crashes and 1 hang among the LLM answers, 0 crashes among the DSL and
markdown shapes (the three fixes you already applied cover them), 11 DSL cases where the generated
file reports `re-verified=False` — all 11 are rule sets that *are* inconsistent (DONE unreachable
or a trap), where exit code 1 is the specified behaviour, not a defect. After the batch: 0 crashes,
0 hangs, the 11 unchanged.

## 2. What it found and what changes

| # | input | before | after |
|---|---|---|---|
| D1 | a rule with a missing or renamed argument (`precedence` without `then`, `limit` without `n`) | `KeyError` traceback inside compile | validation error naming the rule, the kind, the missing and the expected names → draft + exit 2 |
| D2 | malformed `phase_sequence` (strings instead of objects, no `id`, empty list, duplicate phase ids, an action named `enter_<phase>`) | `TypeError` / `KeyError` / `IndexError`, or silently wrong fusion | validation error |
| D3 | `n` not an integer, negative, or huge (`1000000` made the full state space hang) | hang / traceback | `n` must be an integer in 0…1000 |
| D4 | condition with a syntax error; a constant that is not in the parameter's domain (`day == 'monday'` with {friday, other}: a rule that can never fire); duplicate enum values | `SyntaxError` traceback / silent vacuous rule / duplicate tokens | validation errors |
| E1 | the LLM invents a rule kind or a modality (`must_do`, `may`) | `ValidationError` traceback (rule built outside the try) | that rule is REJECTed and logged; the rest of the sample survives |
| E2 | the same rule listed twice in one sample | counted as two votes — one sample could pass the 2-vote bar alone | one vote per sample |
| E3 | `"params": null` on an action | the whole sample rejected | treated as `{}` |
| F1 | `--verify auto` looked only at the rule count; a `limit n=60` in a small file multiplies the state space by 61 | full state space, very slow | `auto` = full only while ≤ 20 compiled rules **and** the limit/exactly budgets sum to ≤ 50 tokens; `verification_method()` is a function with a test |
| F2 | `--extractor yaml:` with a missing or invalid file | traceback | `no such spec file` / `<file> is not a valid AgentSpec: <why>`, exit 2 |
| F3 | agent file path does not exist | traceback | `agentmd2cpn: no such file: <path>` |
| G | Monitor refusals that are not a rule's doing: unknown parameter name, value outside the domain, a call after `finish()` | `? \| BLOCK …` | `BLOCK a{'y': 'u'}: unknown parameter 'y' (expected ['x'])`, `BLOCK a{'x': 'w'}: x='w' is not a declared value ['u', 'v']`, `BLOCK b: run already finished` |
| H | `english()` (the verbaliser used in the log) crashed on malformed args and hid the real error | `KeyError` | never raises; shows the args as they are |

Not changed, on purpose: `report.json`'s `generated_file_reverified` is the generated file's own
verdict (`True` = it reports a consistent net). For an inconsistent file it is `False`, like the
orchestrator as extracted. Say it that way if asked.

## 3. The tests (test-first, as always)

Six tests, fail on the current code, pass after the batch. Append each to the file named.

`tests/test_dsl.py`:

```python
def test_rule_argument_shapes_are_validated():
    import pytest
    from agentmd2cpn.dsl import AgentSpec
    base = {"name": "t", "actions": [{"id": "a"}, {"id": "b"}]}
    for kind, args in [("precedence", {"first": "a"}), ("limit", {"action": "a"}), ("phase_sequence", {"phases": ["p1"]}),
                       ("phase_sequence", {"phases": []}), ("phase_sequence", {"phases": [{"id": "p", "allows": []}, {"id": "p", "allows": []}]}),
                       ("limit", {"action": "a", "n": 5000}), ("exactly", {"action": "a", "n": "two"})]:
        with pytest.raises(ValueError):
            AgentSpec.model_validate(dict(base, rules=[{"id": "R1", "kind": kind, "args": args}]))
    assert AgentSpec.model_validate(dict(base, rules=[{"id": "R1", "kind": "limit", "args": {"action": "a", "n": "2"}}]))


def test_conditions_and_domains_are_validated():
    import pytest
    from agentmd2cpn.dsl import AgentSpec
    base = {"name": "t", "actions": [{"id": "a", "params": {"day": "DAY"}}, {"id": "enter_p2"}], "data": {"DAY": {"type": "enum", "values": ["friday", "other"]}}}
    with pytest.raises(ValueError):
        AgentSpec.model_validate(dict(base, rules=[{"id": "R1", "kind": "forbid_when", "args": {"action": "a", "where": "day == "}}]))
    with pytest.raises(ValueError):
        AgentSpec.model_validate(dict(base, rules=[{"id": "R1", "kind": "forbid_when", "args": {"action": "a", "where": "day == 'monday'"}}]))
    with pytest.raises(ValueError):
        AgentSpec.model_validate(dict(base, data={"DAY": {"type": "enum", "values": ["x", "x"]}}, rules=[]))
    with pytest.raises(ValueError):
        AgentSpec.model_validate(dict(base, rules=[{"id": "R1", "kind": "phase_sequence", "args": {"phases": [{"id": "p1", "allows": ["a"]}, {"id": "p2", "allows": []}]}}]))
    assert AgentSpec.model_validate({"name": "t", "actions": [{"id": "a", "params": None}], "rules": []}).actions[0].params == {}
```

`tests/test_extract.py`:

```python
def test_bad_rules_are_rejected_per_rule_and_votes_count_samples():
    from pathlib import Path
    from agentmd2cpn.dsl import Action
    from agentmd2cpn.extract import ExtractedRule, ExtractionResult, ground_and_vote
    from agentmd2cpn.ingest import parse_markdown
    doc = parse_markdown(Path("tests/fixtures/a/.Agent.md"))
    acts = [Action(id="run_tests"), Action(id="git_commit")]
    good = ExtractedRule(kind="precedence", args={"first": "run_tests", "then": "git_commit"}, quote="Run pnpm test before committing.", line_begin=12, line_end=12)
    bad_kind = ExtractedRule(kind="must_do", args={"action": "run_tests"}, quote="Run pnpm test before committing.", line_begin=12, line_end=12)
    bad_mod = ExtractedRule(kind="forbid", args={"action": "git_commit"}, modality="may", quote="Never push directly to main.", line_begin=11, line_end=11)
    s1 = ExtractionResult(actions=acts, rules=[good, good, bad_kind, bad_mod])
    s2 = ExtractionResult(actions=acts, rules=[])
    assert ground_and_vote(doc, [s1, s2], 2, "t").rules == []
    assert [r.kind for r in ground_and_vote(doc, [s1, s1], 2, "t").rules] == ["precedence"]
```

`tests/test_cli.py`:

```python
def test_verification_method_depends_on_size_and_budgets():
    from agentmd2cpn.cli import verification_method
    from agentmd2cpn.dsl import AgentSpec
    acts = [{"id": "a"}, {"id": "b"}]
    small = AgentSpec.model_validate({"name": "t", "actions": acts, "rules": [{"id": "R1", "kind": "limit", "args": {"action": "a", "n": 2}}]})
    big_budget = AgentSpec.model_validate({"name": "t", "actions": acts, "rules": [{"id": "R1", "kind": "limit", "args": {"action": "a", "n": 60}}]})
    many = AgentSpec.model_validate({"name": "t", "actions": acts, "rules": [{"id": f"R{i}", "kind": "precedence", "args": {"first": "a", "then": "b"}} for i in range(21)]})
    assert verification_method(small) == "full" and verification_method(big_budget) == "compositional" and verification_method(many) == "compositional"
    assert verification_method(many, "full") == "full" and verification_method(small, "compositional") == "compositional"
```

`tests/test_runtime.py`:

```python
def test_monitor_explains_non_rule_refusals():
    from agentmd2cpn.dsl import AgentSpec
    from agentmd2cpn.runtime import Monitor
    from agentmd2cpn.verify import build
    spec = AgentSpec.model_validate({"name": "t", "actions": [{"id": "a", "params": {"x": "T"}}, {"id": "b"}], "data": {"T": {"type": "enum", "values": ["u", "v"]}}, "rules": []})
    frag, cpn, m0, ctx = build(spec)
    m = Monitor(cpn, m0, ctx, blame=frag.blame)
    t = cpn.get_transition_by_name("a")
    assert "unknown parameter" in m.why_not_fireable(t, {"y": "u"})
    assert "not a declared value" in m.why_not_fireable(t, {"x": "w"})
    assert m.why_not_fireable(t, {"x": "u"}) is None and m.try_action("a", x="u") and m.finish()
    assert m.why_not_fireable(t, {}) == "run already finished" and not m.try_action("b")
```

`tests/test_templates.py`:

```python
def test_english_never_raises():
    from agentmd2cpn.templates import english
    assert "precedence" in english("precedence", {"first": "a"}) and "limit" in english("limit", {}) and english("phase_sequence", {"phases": ["x"]})
```

`python -m pytest tests -q` now: 6 failed, 55 passed (the six new ones fail — that is the point).

## 4. The Copilot prompt (one prompt, five files)

```text
Make these changes so that the six new tests (tests/test_dsl.py: 2, tests/test_extract.py: 1, tests/test_cli.py: 1, tests/test_runtime.py: 1, tests/test_templates.py: 1) pass and all other tests still pass. Do not touch tests/. No behaviour changes beyond what is listed; existing docstrings stay.

1. agentmd2cpn/dsl.py
   a. Before class AgentSpec add module-level constants MAX_BUDGET = 1000 and ARG_KEYS, a dict from rule kind to the set of required argument names: forbid, require, init, end -> {"action"}; exactly, limit -> {"action", "n"}; precedence, alt_precedence -> {"first", "then"}; approval -> {"approver", "action"}; response, chain_response -> {"trigger", "then"}; responded_existence, not_coexistence, not_succession -> {"a", "b"}; forbid_when, scope -> {"action", "where"}; phase_sequence -> {"phases"}. Add a helper _string_constants(expr) that returns the str constants found by walking ast.parse(expr, mode="eval").
   b. In AgentSpec's model validator: before the rule loop, every enum DataType must have unique values (ValueError "data type <name>: duplicate enum values"). Inside the rule loop, first: r.args must be a dict (ValueError), and ARG_KEYS[r.kind] - set(r.args) must be empty, else ValueError naming the rule id, the kind, the missing names and the expected names. For forbid_when/scope, after check_condition: every string constant of the condition must be a value of one of the action's enum parameters or a declared role, else ValueError naming the constant and the known values. For limit/exactly: int(r.args["n"]) must parse (ValueError "n must be an integer, got ...") and lie between 0 and MAX_BUDGET inclusive (ValueError mentioning MAX_BUDGET). For phase_sequence: phases must be a non-empty list of dicts whose "id" is a str (ValueError "phases must be a non-empty list of {id, allows} objects"); phase ids must be unique; if any action id equals "enter_" + a phase id raise ValueError (it would clash with the phase transition); each phase's "allows" must be a list of declared action ids.
   c. check_condition: wrap ast.parse in try/except SyntaxError and raise ValueError(f"condition {expr!r}: not a valid expression ({e.msg})") instead.
   d. Action: add a field_validator("params", mode="before") that returns {} when the value is None.
2. agentmd2cpn/extract.py, ground_and_vote: inside the loop over samples keep a set seen_in_sample. For each rule of the sample, in this order: if r.kind not in RuleKind.__args__, log a warning "REJECT rule with unknown kind {!r} {} (lines {}-{})" and skip; if r.modality not in ("must", "must_not", "should"), log a warning "REJECT rule {} {} with unknown modality {!r}" and skip; then the existing grounding check; then key = json.dumps([r.kind, r.args], sort_keys=True, default=str); if key is already in seen_in_sample skip it (one vote per sample), otherwise add it, count it and keep the rule as today.
3. agentmd2cpn/templates.py: english(kind, args) must never raise. Move the current body into a helper _english(kind, args) and make english return _english(kind, args), or f"{kind} {args!r}" on KeyError, TypeError, ValueError or AttributeError.
4. agentmd2cpn/cli.py
   a. Add module constants MAX_FULL_RULES = 20 and MAX_FULL_BUDGET = 50 and a function verification_method(spec, verify="auto") -> str returning "full" or "compositional": when verify is "full" or "compositional" return it; otherwise compiled = the rules with modality != "should", budget = sum of int(r.args["n"]) over the compiled rules of kind limit or exactly; return "full" iff len(compiled) <= MAX_FULL_RULES and budget <= MAX_FULL_BUDGET, else "compositional". A 3-line docstring. run() uses verification_method(spec, verify) in the verify stage instead of the inline condition; the log line "... full state space not enumerable, using compositional verification" stays.
   b. In the extract stage, the yaml: branch wraps the read and AgentSpec.model_validate in try/except: on FileNotFoundError log an error "no such spec file: {path}" and raise SystemExit(2); on any other exception log an error "{path} is not a valid AgentSpec: {messages}" where messages is "; ".join(x["msg"] for x in e.errors()) when e has errors(), else str(e), then raise SystemExit(2).
   c. In main(), before calling run(): if the agent_md path is not a file, raise SystemExit(f"agentmd2cpn: no such file: {path}").
5. agentmd2cpn/runtime.py, class Monitor: add a method why_not_fireable(self, t, params) -> str | None: return "run already finished" if the DONE place holds a token; then for each k, v in params: if k not in t.variables return f"unknown parameter {k!r} (expected {vars})" where vars are the variables of t that do not start with "s_", "ph_" or "c_"; else if a place named f"ENV_{t.name}_{k}" exists in the marking and v is not among its token values return f"{k}={v!r} is not a declared value {domain}"; otherwise return None. In try_action, when no binding matches, call why_not_fireable first: if it returns a reason, log a warning "BLOCK {action}{params}: {reason}" with rule="-" and return False; otherwise keep the existing explain() path unchanged.

Then run python -m pytest tests -q and paste the full output; it must say 61 passed. Then run python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png and paste the last line; it must still be: 16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True.
```

If Copilot's attempt leaves a test failing after one correction round, send me the failing test's output.

## 5. Check it (10 min)

```powershell
python -m pytest tests -q
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png
python -m agentmd2cpn examples/expert/expert.agent.md -o out/expert --extractor yaml:examples/expert/spec.yaml --no-png
python -m agentmd2cpn examples/universal/universal.agent.md -o out/universal --extractor yaml:examples/universal/spec.yaml --no-png
python -m agentmd2cpn tests/fixtures/a/.Agent.md -o out/demo --extractor yaml:tests/fixtures/a/spec.yaml --no-png
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator --extractor yaml:examples/orchestrator/spec.yaml --no-png
```

Expected: `61 passed`; release `16 rules, 2569 …`; expert `4 rules, 48 …`; universal `0 rules, 2 …`;
demo `2 rules, 4 …`; orchestrator `47 rules, compositional check: conflicts=0, … explored=5002 …`
— all exactly as before. Two negative checks, both must end cleanly, without a traceback:

```powershell
python -m agentmd2cpn examples/nope.md -o out/x
python -m agentmd2cpn examples/universal/universal.agent.md -o out/x --extractor yaml:examples/nope.yaml
```

→ `agentmd2cpn: no such file: examples\nope.md` and `ERROR | extract | no such spec file: examples\nope.yaml`.

## 6. What this does and does not promise — say it this way

After this batch, for **any** markdown file and **any** answer the model gives, the tool ends in one
of three ways: a verified net, a draft `agent_spec.yaml` with a message naming what is wrong (exit
2), or a logged rejection of the bad rule with the rest kept. No input shape in the three sweeps
produces a traceback or a hang. The reference files give the same numbers as before.

What no code change can remove, because it is the design and not a defect:

1. **The review.** The model's reading of a sentence can be defensible and still not what the
   author meant (the orchestrator's "first", the expert file's guideline precedences). The tool
   bounds the model (schema, grounding, voting, validation) and the person decides. Every file
   will have one or two of these decisions; that is a five-minute read of `agent_spec.yaml`,
   not a fix.
2. **The model's variance.** Two live runs of the same file can differ by a rule (expert:
   `require create_plan` had 2 votes once and 1 vote the next time). The reviewed `spec.yaml`
   is what makes a result reproducible.
3. **Meaning the 17 kinds cannot express** — a condition on the *result* of an earlier action
   ("apply only if validation accepted"). That needs an 18th template plus a test; the
   architecture is built for it, but it is a feature, not a bug.

For the write-up: tests 61 (dsl 14, extract 5, cli 4, runtime 4, templates 15, ingest 4,
verify 5, verify_large 4, compile 1, compose 2, emit 2, fastcpn 1); engineering-log problems
(8) no actions, (9) phase variable in the explorer, (10) multi-item quotes, (11) this sweep:
10 LLM-answer shapes that crashed or hung, fixed by validation at the DSL boundary. README
limitations: add "`limit`/`exactly` accept n up to 1000; `--verify auto` switches to the
compositional method above 20 compiled rules or 50 budget tokens".
