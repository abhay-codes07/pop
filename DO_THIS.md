# agentmd2cpn — DO THIS, in this order (complete, from where you are now)

Everything you need is in this file: every file to create, every edit, every command, and what
each command must print. Work top to bottom. Do not skip a part — later parts assume the earlier
ones. Every file here was built and verified on a copy of your project; the whole chain ends
with every example re-run and all tests green.

Conventions used below:

- **Type / copy** = create or replace the file with exactly the content shown (copy it from this
  document on GitHub). Then check the hash with the command in A0 — the 8 characters must match
  the ones given. If they do not match, you have a typo; fix it before going on.
- **Copilot** = paste the text block into Copilot chat (Agent mode). Copilot must never edit
  anything under `tests/`.
- **Check** = run the command and compare with the expected output.
- `N` = the number of tests `pytest` prints for you right now (A0 measures it). Each part says
  what it adds.
- All commands are PowerShell, run in `C:\Users\z005cvjb\modelling-petrinets` with the venv active.

---

## Part A — robustness: no input shape may crash the tool

### A0. Measure where you are

```powershell
python -m pytest tests -q
```

Write down the number it prints (`55 passed` or a little more if Copilot added tests with the
package). That is `N`.

The hash command (use it on every typed file; replace `<file>`):

```powershell
python -c "import hashlib,pathlib;print(hashlib.sha256(b'\n'.join(l.rstrip() for l in pathlib.Path('<file>').read_bytes().splitlines() if l.strip())).hexdigest()[:8])"
```

### A1. Add six tests (type / copy; append each block at the END of the file named)

Append to `tests/test_dsl.py`:

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

Append to `tests/test_extract.py`:

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

Append to `tests/test_cli.py`:

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

Append to `tests/test_runtime.py`:

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

Append to `tests/test_templates.py`:

```python
def test_english_never_raises():
    from agentmd2cpn.templates import english
    assert "precedence" in english("precedence", {"first": "a"}) and "limit" in english("limit", {}) and english("phase_sequence", {"phases": ["x"]})
```

**Check:**

```powershell
python -m pytest tests -q
```

→ `6 failed, N passed`. The six new tests fail; nothing else fails. (If something else fails,
you pasted into the wrong place — fix that first.)

### A2. Copilot

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

Then run python -m pytest tests -q and paste the full output; it must say N+6 passed, where N is the count before the six new tests. Then run python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png and paste the last line; it must still be: 16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True.
```

Copilot must end with `N+6 passed`. If one test still fails after one correction round, do not
let Copilot edit the test — paste me the failing test's output.

### A3. Check

```powershell
python -m pytest tests -q
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png
python -m agentmd2cpn examples/nope.md -o out/x
python -m agentmd2cpn examples/universal/universal.agent.md -o out/x --extractor yaml:examples/nope.yaml
```

→ `N+6 passed`; the release line unchanged:
`examples\release_agent\.Agent.md: 16 rules, 2569 reachable markings, consistent=True, traps=0, dead=['force_push'], generated file re-verified=True`;
the two wrong paths end cleanly with `agentmd2cpn: no such file: examples\nope.md` and
`ERROR | extract | no such spec file: examples\nope.yaml` — no traceback.

Part A done. Click **Keep** on Copilot's changes.

---

## Part B — the completed pipeline: REPORT.md, lock, enforcement check, fresh logs

No Copilot in this part. Tests first (B1–B4), then two new modules (B5–B6), three replaced
files (B7–B9), four small edits (B10–B13), then the check (B14).

### B1. Create `tests/test_report.py` (type / copy; 55 lines, hash `acdc2aed`)

```python
def test_report_names_conflicts_dead_ends_and_dead_actions(tmp_path):
    import yaml
    from agentmd2cpn.cli import run
    md = tmp_path / "x.agent.md"
    md.write_text("# X\n\n- Start with a.\n- Do a only after b.\n- Never do c.\n", encoding="utf-8")
    spec = {"name": "x", "actions": [{"id": "a"}, {"id": "b"}, {"id": "c"}], "rules": [
        {"id": "R1", "kind": "init", "args": {"action": "a"}, "source": {"line_begin": 3, "line_end": 3, "quote": "Start with a."}},
        {"id": "R2", "kind": "precedence", "args": {"first": "b", "then": "a"}, "source": {"line_begin": 4, "line_end": 4, "quote": "Do a only after b."}},
        {"id": "R3", "kind": "forbid", "args": {"action": "c"}, "source": {"line_begin": 5, "line_end": 5, "quote": "Never do c."}}]}
    y = tmp_path / "spec.yaml"
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    run(md, tmp_path / "out", extractor=f"yaml:{y}", png=False)
    rep = (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    assert "INCONSISTENT" in rep and 'line 3: "Start with a."' in rep and 'line 4: "Do a only after b."' in rep
    assert "removing any ONE of R1, R2" in rep and "BLOCKED by R2" in rep and "forbidden by R3" in rep and "suspicious" in rep
    del spec["rules"][0]
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    run(md, tmp_path / "out2", extractor=f"yaml:{y}", png=False)
    rep = (tmp_path / "out2" / "REPORT.md").read_text(encoding="utf-8")
    assert "Verdict: CONSISTENT" in rep and "forbidden by R3" in rep and "Never do c." in rep and "suspicious" not in rep


def test_report_zero_rules_and_livelock(tmp_path):
    import yaml
    from agentmd2cpn.cli import run
    md = tmp_path / "y.md"
    md.write_text("# Y\n\n- Finish with stop.\n- After stop, always summarize.\n", encoding="utf-8")
    y = tmp_path / "spec.yaml"
    y.write_text(yaml.safe_dump({"name": "y", "actions": [], "rules": []}), encoding="utf-8")
    run(md, tmp_path / "out", extractor=f"yaml:{y}", png=False)
    assert "NOTHING TO ENFORCE" in (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    spec = {"name": "y", "actions": [{"id": "stop"}, {"id": "summarize"}], "rules": [
        {"id": "R1", "kind": "end", "args": {"action": "stop"}, "source": {"line_begin": 3, "line_end": 3, "quote": "Finish with stop."}},
        {"id": "R2", "kind": "response", "args": {"trigger": "stop", "then": "summarize"}, "source": {"line_begin": 4, "line_end": 4, "quote": "After stop, always summarize."}}]}
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    run(md, tmp_path / "out2", extractor=f"yaml:{y}", png=False)
    rep = (tmp_path / "out2" / "REPORT.md").read_text(encoding="utf-8")
    assert "INCONSISTENT" in rep and "removing any ONE of R1, R2" in rep and "livelock" in rep and 'line 4: "After stop, always summarize."' in rep


def test_report_review_first_lists_shaky_rules_and_silent_sentences(tmp_path):
    import yaml
    from agentmd2cpn.cli import run
    md = tmp_path / "r.md"
    md.write_text("# R\n\n- Start with a.\n- Never do c.\n- Always do b after a.\n", encoding="utf-8")
    spec = {"name": "r", "actions": [{"id": "a"}, {"id": "b"}, {"id": "c"}], "rules": [
        {"id": "R1", "kind": "init", "args": {"action": "a"}, "votes": 3, "source": {"line_begin": 3, "line_end": 3, "quote": "Start with a."}},
        {"id": "R2", "kind": "forbid", "args": {"action": "c"}, "votes": 2, "source": {"line_begin": 4, "line_end": 4, "quote": "Never do c."}}]}
    y = tmp_path / "spec.yaml"
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    run(md, tmp_path / "out", extractor=f"yaml:{y}", png=False)
    rep = (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    assert "## Review first" in rep and "[2/3 votes]" in rep and 'line 4: "Never do c."' in rep
    silent = rep.split("Sentences with rule words")[1].split("## Verdict")[0]
    assert 'line 5: "Always do b after a."' in silent and "line 3" not in silent
```

### B2. Create `tests/test_lock.py` (48 lines, hash `cc3e9678`)

```python
import json

import yaml

from agentmd2cpn.cli import run

ACTS = [{"id": "a"}, {"id": "b"}]
RULE = {"kind": "precedence", "args": {"first": "a", "then": "b"}, "quote": "Do b only after a.", "line_begin": 3, "line_end": 3}


def _fake(calls):
    def complete(prompt):
        calls.append(prompt)
        return json.dumps({"actions": ACTS, "rules": [RULE]})
    return complete


def test_same_file_is_extracted_once_then_reused_and_locked(tmp_path):
    md = tmp_path / "t.agent.md"
    md.write_text("# T\n\n- Do b only after a.\n", encoding="utf-8")
    calls = []
    rep1 = run(md, tmp_path / "out", complete=_fake(calls), png=False)          # auto: nothing to reuse -> the LLM, 3 samples
    assert len(calls) == 3 and len(rep1["rules"]) == 1
    spec = yaml.safe_load((tmp_path / "out" / "agent_spec.yaml").read_text(encoding="utf-8"))
    assert len(spec["source_hash"]) == 16 and spec["provenance"].startswith("llm ")
    rep2 = run(md, tmp_path / "out", complete=_fake(calls), png=False)          # auto: the previous extraction matches -> no LLM
    assert len(calls) == 3 and rep2["rules"] == rep1["rules"]
    run(md, tmp_path / "out", complete=_fake(calls), png=False, lock=True)       # --lock: t.spec.yaml next to the file
    lock = tmp_path / "t.spec.yaml"
    assert lock.is_file() and yaml.safe_load(lock.read_text(encoding="utf-8"))["provenance"].startswith("reviewed ")
    rep3 = run(md, tmp_path / "out_fresh", complete=_fake(calls), png=False)    # another out dir: the lock is what is reused
    assert len(calls) == 3 and rep3["rules"] == rep1["rules"]
    assert "reused from the reviewed spec" in (tmp_path / "out_fresh" / "REPORT.md").read_text(encoding="utf-8")


def test_changed_file_and_forced_llm_extract_again(tmp_path):
    md = tmp_path / "t.agent.md"
    md.write_text("# T\n\n- Do b only after a.\n", encoding="utf-8")
    calls = []
    run(md, tmp_path / "out", complete=_fake(calls), png=False, lock=True)
    assert len(calls) == 3
    run(md, tmp_path / "out", extractor="llm", complete=_fake(calls), png=False)   # llm = always extract
    assert len(calls) == 6
    md.write_text("# T\n\n- Do b only after a.\n- Be brief.\n", encoding="utf-8")   # the file changed: its hash differs
    calls.clear()
    run(md, tmp_path / "out", complete=_fake(calls), png=False)                 # auto: lock and previous extraction are stale
    assert len(calls) == 3
    assert "is for another version of t.agent.md" in (tmp_path / "out" / "pipeline.jsonl").read_text(encoding="utf-8")
```

### B3. Create `tests/test_selfcheck.py` (47 lines, hash `6148c1b4`)

```python
import yaml

from agentmd2cpn.cli import run


def _spec(tmp_path):
    md = tmp_path / "s.agent.md"
    md.write_text("# S\n\n- Start with a.\n- Do b only after d.\n- Never do c.\n- Do d exactly once.\n- Finish with b.\n", encoding="utf-8")
    spec = {"name": "s", "actions": [{"id": "a"}, {"id": "b"}, {"id": "c"}, {"id": "d"}], "rules": [
        {"id": "R1", "kind": "init", "args": {"action": "a"}, "source": {"line_begin": 3, "line_end": 3, "quote": "Start with a."}},
        {"id": "R2", "kind": "precedence", "args": {"first": "d", "then": "b"}, "source": {"line_begin": 4, "line_end": 4, "quote": "Do b only after d."}},
        {"id": "R3", "kind": "forbid", "args": {"action": "c"}, "source": {"line_begin": 5, "line_end": 5, "quote": "Never do c."}},
        {"id": "R4", "kind": "exactly", "args": {"action": "d", "n": 1}, "source": {"line_begin": 6, "line_end": 6, "quote": "Do d exactly once."}},
        {"id": "R5", "kind": "end", "args": {"action": "b"}, "source": {"line_begin": 7, "line_end": 7, "quote": "Finish with b."}}]}
    y = tmp_path / "spec.yaml"
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    return md, y


def test_every_rule_gets_an_enforcement_witness_in_report_and_monitor_log(tmp_path):
    md, y = _spec(tmp_path)
    rep = run(md, tmp_path / "out", extractor=f"yaml:{y}", png=False)
    enf = rep["state_space"]["enforcement"]
    assert sorted(enf["witnesses"]) == ["R1", "R2", "R3", "R4", "R5"] and enf["unwitnessed"] == []
    assert enf["witnesses"]["R3"]["step"] == "c" and enf["witnesses"]["R4"]["step"] in ("d", "END") and enf["witnesses"]["R5"]["step"] == "END"
    text = (tmp_path / "out" / "REPORT.md").read_text(encoding="utf-8")
    assert "## Enforcement check" in text and "c -> BLOCKED by R3" in text and "finishing -> BLOCKED by R5" in text
    assert 'line 5: "Never do c."' in text
    mon = (tmp_path / "out" / "monitor.log").read_text(encoding="utf-8")
    assert "DONE: all rules satisfied" in mon                      # the verifier's complete run, replayed
    assert "| R3 | BLOCK c" in mon and "| R5 | BLOCK END" in mon and 'probe R3 (line 5: "Never do c.")' in mon


def test_enforcement_check_is_skipped_for_an_inconsistent_or_empty_spec(tmp_path):
    md, y = _spec(tmp_path)
    spec = yaml.safe_load(y.read_text(encoding="utf-8"))
    spec["rules"][4]["args"]["action"] = "c"                       # end with c, which is forbidden: no run can finish
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    rep = run(md, tmp_path / "out", extractor=f"yaml:{y}", png=False)
    assert rep["state_space"]["done_reachable"] is False
    assert "R3" in rep["state_space"]["enforcement"]["witnesses"]        # probes still run where they can
    assert "complete run from the verifier" not in (tmp_path / "out" / "monitor.log").read_text(encoding="utf-8")
    spec["rules"], spec["actions"] = [], []
    y.write_text(yaml.safe_dump(spec), encoding="utf-8")
    rep = run(md, tmp_path / "out2", extractor=f"yaml:{y}", png=False)
    assert rep["state_space"]["enforcement"]["witnesses"] == {}
    assert "## Enforcement check" not in (tmp_path / "out2" / "REPORT.md").read_text(encoding="utf-8")
```

### B4. Append three tests

Append to `tests/test_ingest.py`:

```python
def test_document_sha_identifies_the_text(tmp_path):
    from agentmd2cpn.ingest import parse_markdown
    a, b, c = tmp_path / "a.md", tmp_path / "b.md", tmp_path / "c.md"
    a.write_text("# A\n\n- Never push to main.\n", encoding="utf-8")
    b.write_bytes(b"# A\r\n\r\n- Never push to main.\r\n")                       # same text, Windows line endings
    c.write_text("# A\n\n- Never push to main!\n", encoding="utf-8")             # one character differs
    sa, sb, sc = (parse_markdown(p).sha for p in (a, b, c))
    assert len(sa) == 16 and sa == sb and sa != sc
```

Append to `tests/test_cli.py` (`FIX` is already defined at the top of that file):

```python
def test_each_pipeline_run_starts_its_logs_fresh_and_dates_monitor_log(tmp_path):
    from agentmd2cpn.cli import run
    for _ in range(2):
        run(FIX / ".Agent.md", tmp_path, extractor=f"yaml:{FIX / 'spec.yaml'}", png=False)
    jl = (tmp_path / "pipeline.jsonl").read_text(encoding="utf-8")
    assert jl.count('"parsed ') == 1                                   # not two runs appended
    mon = (tmp_path / "monitor.log").read_text(encoding="utf-8").splitlines()
    assert sum("end of pipeline run" in l for l in mon) == 1 and "examples/drive.py" in mon[-1]   # one run, dated
```

Append to `tests/test_extract.py` (`FIX`, `ACTS`, `GOOD`, `MINOR`, `fake_llm` are already
defined at the top of that file):

```python
def test_rule_ids_follow_document_order_not_vote_order():
    doc = parse_markdown(FIX)
    both = json.dumps({"actions": ACTS, "rules": [GOOD, MINOR]})            # GOOD cites line 12, MINOR line 11
    spec = extract_spec(doc, "demo", LLMExtractor(fake_llm([both, both, json.dumps({"actions": ACTS, "rules": [GOOD]})]), samples=3), min_votes=2)
    assert [(r.id, r.source.line_begin, r.votes) for r in spec.rules] == [("R1", 11, 2), ("R2", 12, 3)]
```

**Check:** `python -m pytest tests -q` → the 10 new tests fail (missing `REPORT.md`, `run()` has no
`lock` argument, no `enforcement` key, no `sha`, …); nothing else fails. That is expected; go on.

### B5. Create `agentmd2cpn/report.py` (205 lines, hash `9f738b9b`)

```python
"""REPORT.md - the verification result in the words of the agent file.

Rule ids are mapped back to their line numbers and verbatim quotes, conflicts are explained as
the pairs of sentences that cannot hold together, every dead end is shown the way the Monitor
would see it (which action is BLOCKED by which rule at that point), and dead actions are
labelled "forbidden by ..." or "suspicious".  Nothing in here depends on a particular file.
"""
from __future__ import annotations

import re
from contextlib import contextmanager
from pathlib import Path

import networkx as nx
from loguru import logger

from .compose import DONE, END
from .dsl import AgentSpec, Rule
from .runtime import Monitor
from .templates import english
from .verify import build, state_space_check
from .verify_large import conflicts

RULE_WORDS = re.compile(r"\b(must|never|always|only|before|after|at most|at least|exactly|do not|don't|should|first|last|"
                        r"finish|start|immediately|once|until|unless|forbidden|required|prohibited|ensure|not)\b", re.I)


def review_items(doc, spec: AgentSpec) -> tuple[list[Rule], list]:
    """What a reviewer should look at before trusting the rules: rules not seen in every sample, and sentences that
    use rule words (must, never, before, ...) but produced no rule - where the extraction may have missed something."""
    top = max((r.votes for r in spec.rules), default=0)
    shaky = [r for r in spec.rules if r.votes < top]
    cited = set()
    for r in spec.rules:
        if r.source and r.source.line_begin:
            cited.update(range(r.source.line_begin, max(r.source.line_begin, r.source.line_end) + 1))
    silent = [s for s in doc.segments if s.kind in ("list_item", "paragraph") and RULE_WORDS.search(s.text)
              and not set(range(s.line_begin, s.line_end + 1)) & cited]
    return shaky, silent


@contextmanager
def _quiet():
    """Build the sub-nets of this stage without the per-build compile/compose log lines."""
    for m in ("agentmd2cpn.compile", "agentmd2cpn.compose"):
        logger.disable(m)
    try:
        yield
    finally:
        for m in ("agentmd2cpn.compile", "agentmd2cpn.compose"):
            logger.enable(m)


def _sub(spec: AgentSpec, rules: list[Rule]) -> AgentSpec:
    """The same spec restricted to `rules` (actions, data and roles unchanged)."""
    return AgentSpec(name=spec.name, actions=spec.actions, data=spec.data, roles=spec.roles, rules=list(rules))


def _cite(r: Rule) -> str:
    s = r.source
    where = f"line {s.line_begin}" if s and s.line_begin else "no source"
    quote = f': "{s.quote.strip()}"' if s and s.quote else ""
    return f"{r.id}  {r.kind}: {english(r.kind, r.args)}   ({where}{quote})"


def dead_end(spec: AgentSpec, rules: list[Rule] | None = None, mode: str = "enforce") -> dict | None:
    """The first trap state of the net of `rules` (default: all), as the Monitor would see it:
    the shortest path there and, at that point, every action with the rules that block it."""
    sub = spec if rules is None else _sub(spec, rules)
    with _quiet():
        frag, cpn, m0, ctx = build(sub, mode)
    res = state_space_check(cpn, m0, ctx, quiet=True)
    a = res["analyzer"]
    done = {n for n in a.RG.nodes if a.RG.nodes[n]["marking"].get_multiset(DONE).tokens}
    traps = [n for n in a.list_dead_markings() if n not in done]
    if not traps:
        return None
    src = list(a.RG.nodes)[0]
    best = None
    for t in traps:
        try:
            p = nx.shortest_path(a.RG, src, t)
        except nx.NetworkXNoPath:
            continue
        if best is None or len(p) < len(best):
            best = p
    if best is None:
        return None
    path = [a.RG.edges[u, v]["transition"] for u, v in zip(best, best[1:])]
    m = Monitor(cpn, a.RG.nodes[best[-1]]["marking"], ctx, blame=frag.blame)
    blocked = {}
    for t in cpn.transitions:
        if t.name == END:
            continue
        blocked[t.name] = m.explain(t.name, {}) or ["(no binding)"]
    blocked[END] = m.explain(END, {}) or ["(no binding)"]
    return {"path": path, "blocked": blocked}


def _grouped(blocked: dict, indent: str) -> list[str]:
    """One line per set of blocking rules, smallest sets first; long action lists are abbreviated."""
    groups: dict[tuple, list[str]] = {}
    for act, rs in blocked.items():
        groups.setdefault(tuple(rs), []).append(act)
    out = []
    for rs, acts in sorted(groups.items(), key=lambda kv: (len(kv[0]), kv[0])):
        names = ", ".join(acts[:4]) + (f" and {len(acts) - 4} more" if len(acts) > 4 else "")
        out.append(f"{indent}BLOCKED by {', '.join(rs):<22} {names}")
    return out


def write_report(doc, spec: AgentSpec, res: dict, out: Path, mode: str = "enforce", note: str = "") -> Path:
    by_id = {r.id: r for r in spec.rules}
    compiled = [r for r in spec.rules if r.modality != "should"]
    consistent = bool(res.get("done_reachable")) and not res.get("trap_states")
    lines = [f"# agentmd2cpn report - {doc.file}", "",
             f"{len(doc.segments)} segments read; {len(spec.actions)} actions; {len(spec.rules)} rules ({len(compiled)} enforced, "
             f"{len(spec.rules) - len(compiled)} marked should); verification: {res.get('method', 'full')}"
             + ("" if res.get("complete", True) else " (bounded)"), ""]
    if note:
        lines += [note, ""]
    shaky, silent = review_items(doc, spec)
    if shaky or silent:
        lines += ["## Review notes (reviewed spec)" if spec.provenance.startswith("reviewed") else "## Review first", ""]
        if shaky:
            top = max(r.votes for r in spec.rules)
            lines += [f"Rules not seen in every sample (the samples read these sentences differently; {top} = every sample):", ""]
            lines += [f"- {_cite(r)}  [{r.votes}/{top} votes]" for r in shaky] + [""]
        if silent:
            lines += ["Sentences with rule words that produced no rule (decide: no rule, or a missed one to add by hand):", ""]
            lines += [f"- line {s.line_begin}: \"{s.text.strip()[:140]}\"" for s in silent] + [""]
    if not compiled:
        lines += ["## Verdict: NOTHING TO ENFORCE - this file yields no rules", "",
                  "Either the text contains no constraints on the order, count or arguments of actions, or every candidate rule was",
                  "rejected or voted out at extraction (see the REJECT / DROP lines in pipeline.jsonl).", ""]
    elif consistent:
        w = res.get("witness") or []
        lines += ["## Verdict: CONSISTENT - every rule can be satisfied in one run", "",
                  "shortest complete run: " + (" -> ".join(str(x) for x in w) if w else "(the empty run)"), ""]
        if not res.get("complete", True):
            lines += [f"dead ends: none found in {res.get('states')} explored markings (bounded exploration, so their absence is not proven; "
                      "the conflict search - the init/end/chain_response rules together, then every other rule against them - is exact)", ""]
    else:
        why = "no run can satisfy all rules (DONE unreachable)" if not res.get("done_reachable") else f"{res['trap_states']} dead end(s): runs that can neither continue nor finish"
        lines += [f"## Verdict: INCONSISTENT - {why}", ""]
    # ---- conflicts, in sentences
    found = res.get("conflicts")
    if found is None and not consistent:                         # full method: name the culprits now
        with _quiet():
            found = conflicts(spec, mode)
    if found:
        lines += ["## Conflicts - sentences that cannot hold together", ""]
        for i, c in enumerate(found, 1):
            culprits = [by_id[x] for x in c["culprits"] if x in by_id]
            lines += [f"{i}. {c['why']} among {', '.join(c['rules'])} - removing any ONE of {', '.join(c['culprits'])} repairs it:"]
            lines += [f"   - {_cite(r)}" for r in culprits]
            involved = [r for r in spec.rules if r.id in set(c["rules"]) | set(c["culprits"])]
            de = dead_end(spec, involved, mode)
            if de:
                lines += [f"   the dead end, as the Monitor sees it - after: {' -> '.join(de['path']) or '(start)'}"] + _grouped(de["blocked"], "       ")
            else:
                lines += ["   no dead end: runs can go on forever but none of them can finish (a livelock between these sentences)"]
            lines.append("")
    elif not consistent and res.get("trap_states"):
        de = dead_end(spec, None, mode)
        if de:
            lines += ["## The dead end, as the Monitor sees it", "", f"after: {' -> '.join(de['path']) or '(start)'}"] + _grouped(de["blocked"], "    ") + [""]
    # ---- dead actions
    dead = [d for d in res.get("dead_transitions", []) if d != END]
    if dead:
        lines += ["## Actions that can never happen", ""]
        for d in dead:
            forbids = [r for r in compiled if r.kind == "forbid" and r.args.get("action") == d]
            if forbids:
                lines += [f"- {d} - forbidden by {', '.join(_cite(r) for r in forbids)}  [expected]"]
            else:
                mentions = [r.id for r in compiled if d in [v for v in r.args.values() if isinstance(v, str)]
                            or any(d in (p.get("allows") or []) for p in (r.args.get("phases") or []) if isinstance(p, dict))]
                why = "a consequence of the conflicts above" if found else "the other rules leave it no reachable state"
                lines += [f"- {d} - no rule forbids it: {why} (mentioned by {', '.join(mentions) or 'no rule'})  [suspicious]"]
        lines.append("")
    # ---- enforcement: one refused step per rule, from the net
    enf = res.get("enforcement")
    if enf and compiled:
        from .selfcheck import fmt_path
        lines += ["## Enforcement check - one step the net refuses because of that rule alone", ""]
        for r in compiled:
            w = enf["witnesses"].get(r.id)
            if w:
                step = w["step"] + (f"{w['params']}" if w["params"] else "")
                what = "finishing" if w["step"] == END else step
                lines.append(f"- {_cite(r)}  --  after {fmt_path(w['path'])}: {what} -> BLOCKED by {r.id}")
            else:
                lines.append(f"- {_cite(r)}  --  no step refused by this rule alone within {enf['states']} markings: other rules refuse the same steps first (the rule is implied by them), or its trigger lies deeper than the search went")
        lines += ["", "(also replayed on the Monitor: see monitor.log)", ""]
    # ---- the rules
    lines += ["## Rules", "", "| id | kind | meaning | modality | votes | line | quote |", "|---|---|---|---|---|---|---|"]
    for r in spec.rules:
        s = r.source
        lines.append(f"| {r.id} | {r.kind} | {english(r.kind, r.args)} | {r.modality} | {r.votes} | {s.line_begin if s else ''} | {(s.quote.strip() if s and s.quote else '').replace('|', '/')} |")
    lines += ["", "## Actions", "", ", ".join(a.id + (f" {dict(a.params)}" if a.params else "") for a in spec.actions) or "(none)", ""]
    path = out / "REPORT.md"
    path.write_text("\n".join(lines), encoding="utf-8")
    logger.bind(stage="report").info("wrote {}", path)
    return path
```

### B6. Create `agentmd2cpn/selfcheck.py` (194 lines, hash `2347c327`)

```python
"""Enforcement check: for every rule, one concrete step the compiled net refuses because of that rule alone.

The reachable markings are searched breadth-first (bounded) for a state in which some action - or finishing - is
blocked by exactly one rule.  Path + refused step is that rule's enforcement witness.  The witnesses and the
verifier's complete run are then replayed on the Monitor, so monitor.log shows every rule enforcing its sentence on a
real trace.  Nothing here knows any particular file: the probes come from the net and the blame map."""
from __future__ import annotations

import ast
import copy
import heapq
import itertools

from loguru import logger

from .compose import DONE, END
from .dsl import AgentSpec
from .runtime import Monitor


def _key_tokens(ms):
    return tuple(sorted(repr(t.value) for t in ms.tokens))


def _key(mk):
    return tuple((p, _key_tokens(ms)) for p, ms in sorted(mk._marking.items()))


def _enabled(cpn, mk, ctx, action: str, params: dict) -> bool:
    """Is there a binding of this action, with these parameter values, in this marking?"""
    t = cpn.get_transition_by_name(action)
    return t is not None and any(all(b.get(k) == v for k, v in params.items()) for b in cpn._find_all_bindings(t, mk, ctx))


def _fire(cpn, m0, ctx, path) -> "Marking":
    """A copy of m0 after firing `path` = [(action, params), ...]; stops at the first step that cannot fire."""
    mk = copy.deepcopy(m0)
    for a, prm in path:
        t = cpn.get_transition_by_name(a)
        bs = [b for b in cpn._find_all_bindings(t, mk, ctx) if all(b.get(k) == v for k, v in prm.items())] if t else []
        if not bs:
            break
        cpn.fire_transition(t, mk, ctx, binding=bs[0])
    return mk


def _param_choices(cpn, m0, action: str) -> list[dict]:
    """Every combination of declared parameter values of an action (from its ENV_<action>_<param> places), max 16."""
    t = cpn.get_transition_by_name(action)
    names = [v for v in t.variables if not v.startswith(("s_", "ph_", "c_"))]
    domains = []
    for v in names:
        env = f"ENV_{action}_{v}"
        if env not in m0._marking:
            return [{}]
        domains.append([tok.value for tok in m0.get_multiset(env).tokens])
    return [dict(zip(names, vals)) for vals in itertools.product(*domains)][:16] if names else [{}]


def enforcement_witnesses(spec: AgentSpec, frag, cpn, m0, ctx, witness=None, max_markings: int = 400, bfs: int = 120) -> dict:
    """{rule id: {"path": [(action, params)...], "step": action | END, "params": {...}}} for every compiled rule that
    refuses some step on its own.  Phase 1 expands the first `bfs` reachable markings breadth-first (short, readable
    witnesses for most rules).  Phase 2, for the rules still without one, continues best-first up to `max_markings`
    expansions: markings where fewer rules block finishing come first (that is where one rule alone blocks it), among
    them those in which a wanted rule is "armed" (its state place has left its initial state), then shallower ones.  A marking in which exactly one rule blocks
    finishing is that rule's witness and is recorded as soon as it is generated.  The verifier's complete run, when
    given, seeds phase 2: from an accepting marking a single pending obligation is one or two steps away."""
    want = {r.id for r in spec.rules if r.modality != "should"}
    found: dict[str, dict] = {}
    actions = [t.name for t in cpn.transitions if t.name != END]
    choices = {a: _param_choices(cpn, m0, a) for a in actions}
    accepting = dict(frag.blame.get(END, []))
    initial = {p: _key_tokens(ms) for p, ms in m0._marking.items() if p.startswith(("S_", "PHASE_", "B_"))}

    def failing(mk) -> list[str]:                       # rules whose accepting predicate is false = who blocks END
        binding = {}
        for p, ms in mk._marking.items():
            if ms.tokens and p.startswith("S_"):
                binding["s_" + p[2:]] = ms.tokens[0].value
            elif ms.tokens and p.startswith("PHASE_"):
                binding["ph_" + p[6:]] = ms.tokens[0].value
        return [rid for rid, pred in accepting.items() if not ctx.evaluate_guard(pred, binding)]

    def armed(mk) -> int:                               # 1 if some wanted rule's state place has moved, else 0
        return int(any(p.split("_", 1)[1] in want and _key_tokens(mk.get_multiset(p)) != tok for p, tok in initial.items()))

    def key2(mk, path, seed=False):                     # phase 2 priority: few rules blocking END, an armed wanted rule, shallow
        return (-1 if seed else len(failing(mk)), -armed(mk), len(path), next(tick))

    def record_end(mk, path) -> None:
        who = failing(mk)
        if len(who) == 1 and who[0] in want and not mk.get_multiset(DONE).tokens:
            found[who[0]] = {"path": list(path), "step": END, "params": {}}
            want.discard(who[0])

    def probe_actions(mk, path) -> None:
        m = Monitor(cpn, mk, ctx, blame=frag.blame)
        for a in actions:
            for params in choices[a]:
                if not _enabled(cpn, mk, ctx, a, params):
                    who = m.explain(a, params)
                    if len(who) == 1 and who[0] in want:
                        found[who[0]] = {"path": list(path), "step": a, "params": params}
                        want.discard(who[0])

    def successors(mk, path):
        for t in cpn.transitions:
            for b in cpn._find_all_bindings(t, mk, ctx):
                nxt = copy.deepcopy(mk)
                cpn.fire_transition(t, nxt, ctx, binding=b)
                k = _key(nxt)
                if k not in seen:
                    seen.add(k)
                    params = {x: v for x, v in b.items() if not x.startswith(("s_", "ph_", "c_"))}
                    yield nxt, path + [(t.name, params)]

    start = copy.deepcopy(m0)
    seen, states = {_key(start)}, 0
    record_end(start, [])
    tick = itertools.count()
    heap = [(0, next(tick), start, [])]                  # phase 1: breadth-first (depth, tick)
    while heap and want and states < bfs:
        _, _, mk, path = heapq.heappop(heap)
        states += 1
        if mk.get_multiset(DONE).tokens:
            continue
        probe_actions(mk, path)
        for nxt, npath in successors(mk, path):
            record_end(nxt, npath)
            heapq.heappush(heap, (len(npath), next(tick), nxt, npath))
    rest = [key2(mk, path) + (mk, path) for _, _, mk, path in heap]                 # phase 2: re-key
    if witness is not None and want:                     # the verifier's complete run: finishing probes are 1-2 steps from here
        path = _steps(witness)
        mk = _fire(cpn, m0, ctx, path)
        if _key(mk) not in seen:
            seen.add(_key(mk))
            rest.append(key2(mk, path, seed=True) + (mk, path))                  # expanded first
    heapq.heapify(rest)
    heap = rest
    while heap and want and states < max_markings:
        f_stored, a_stored, _, _, mk, path = heapq.heappop(heap)
        if f_stored >= 0 and a_stored != -armed(mk):     # a rule was witnessed meanwhile: lazy re-prioritisation
            heapq.heappush(heap, key2(mk, path) + (mk, path))
            continue
        states += 1
        if mk.get_multiset(DONE).tokens:
            continue
        probe_actions(mk, path)
        for nxt, npath in successors(mk, path):
            record_end(nxt, npath)
            heapq.heappush(heap, key2(nxt, npath) + (nxt, npath))
    return {"witnesses": found, "unwitnessed": sorted(want), "states": states}


def _steps(witness) -> list[tuple[str, dict]]:
    """Normalise a witness from either verifier: 'act', "act{'p': 'v'}" or ('act', {...}) -> (act, params)."""
    out = []
    for s in witness or []:
        if isinstance(s, tuple):
            out.append((s[0], dict(s[1])))
        elif "{" in s:
            name, _, rest = s.partition("{")
            out.append((name, ast.literal_eval("{" + rest)))
        else:
            out.append((s, {}))
    return out


def fmt_path(path) -> str:
    return " -> ".join(a + (f"{p}" if p else "") for a, p in path) or "(start)"


def replay(spec: AgentSpec, frag, cpn, m0, ctx, witness, enforcement: dict) -> None:
    """Write the complete run and one refused step per rule to monitor.log, as real Monitor decisions."""
    log = logger.bind(stage="monitor")
    steps = _steps(witness)
    if witness is not None:
        log.info("complete run from the verifier ({} steps):", len(steps))
        m = Monitor(cpn, m0, ctx, blame=frag.blame)
        for a, p in steps:
            m.try_action(a, **p)
        m.finish()
    by_id = {r.id: r for r in spec.rules}
    for rid, w in enforcement["witnesses"].items():
        r = by_id[rid]
        s = r.source
        log.info("probe {} (line {}: \"{}\") after {}:", rid, s.line_begin if s else "?", (s.quote or "").strip() if s else "", fmt_path(w["path"]))
        m = Monitor(cpn, _fire(cpn, m0, ctx, w["path"]), ctx, blame=frag.blame)   # the prefix, replayed silently
        if w["step"] == END:
            m.finish()
        else:
            m.try_action(w["step"], **w["params"])
    if enforcement["unwitnessed"]:
        log.info("no single-rule refusal found within {} markings for: {}", enforcement["states"], ", ".join(enforcement["unwitnessed"]))
```

### B7. Replace `agentmd2cpn/logging_setup.py` entirely (28 lines, hash `4f1cf67e`)

```python
"""Loguru configuration: one logger, context bound per stage / rule, three sinks."""
from __future__ import annotations

import sys
from pathlib import Path

from loguru import logger

FMT = "<green>{time:HH:mm:ss}</green> | <level>{level:<7}</level> | <cyan>{extra[stage]}</cyan> | {message}"


def setup_logging(out_dir: Path | None = None, level: str = "INFO", fresh: bool = True) -> None:
    """fresh=True (a pipeline run) starts pipeline.jsonl and monitor.log empty, so a folder never mixes two runs;
    fresh=False (drive.py, a Monitor session on an existing folder) appends to them."""
    logger.remove()
    logger.configure(extra={"stage": "-", "rule": "-"})
    logger.add(sys.stderr, level=level, format=FMT, colorize=True)
    if out_dir is not None:
        out_dir.mkdir(parents=True, exist_ok=True)
        mode = "w" if fresh else "a"
        logger.add(out_dir / "pipeline.jsonl", level="DEBUG", serialize=True, mode=mode)
        logger.add(out_dir / "monitor.log", level="INFO", mode=mode,
                   filter=lambda r: r["extra"].get("stage") == "monitor",
                   format="{time:YYYY-MM-DD HH:mm:ss} | {extra[rule]} | {message}")


def stage(name: str):
    return logger.contextualize(stage=name)
```

### B8. Replace `agentmd2cpn/cli.py` entirely (196 lines, hash `ab92c959`)

Your current `cli.py` was written by Copilot; this one contains everything it has (the batch's
`verification_method`, the yaml and missing-file errors) plus the new stages. Keep nothing of
the old file.

```python
"""python -m agentmd2cpn <any .md> -o out/ [--extractor auto|llm|yaml:<spec.yaml>] [--lock] [--samples 3] [--min-votes 2]"""
from __future__ import annotations

import argparse
import datetime
import json
import os
from pathlib import Path

os.environ.setdefault("PM4PY_SHOW_PROGRESS_BAR", "False")

import yaml
from loguru import logger

from .dsl import AgentSpec
from .emit import emit_dot, emit_json, emit_python, run_generated, write_spec
from .extract import ExtractionError, LLMExtractor, extract_spec
from .ingest import parse_markdown
from .logging_setup import setup_logging, stage
from .report import write_report
from .selfcheck import enforcement_witnesses, replay
from .verify import build, composition_check, state_space_check
from .verify_large import verify_large


MAX_FULL_RULES, MAX_FULL_BUDGET = 20, 50


def verification_method(spec: AgentSpec, verify: str = "auto") -> str:
    """full = exhaustive state space; compositional = conflicts + bounded exploration (verify_large).
    auto: full while the rule set is small (<= 20 compiled rules) and the budgets of limit/exactly rules
    add up to <= 50 tokens (each token multiplies the state space)."""
    if verify in ("full", "compositional"):
        return verify
    compiled = [r for r in spec.rules if r.modality != "should"]
    budget = sum(int(r.args["n"]) for r in compiled if r.kind in ("limit", "exactly"))
    return "full" if len(compiled) <= MAX_FULL_RULES and budget <= MAX_FULL_BUDGET else "compositional"


def agent_name(path: Path) -> str:
    """orchestrator.agent.md -> orchestrator, team.skills.md -> team, AGENTS.md -> AGENTS; a dot-file such as
    .Agent.md takes the name of its folder."""
    if path.name.startswith("."):
        return path.parent.name or "agent"
    return path.name.split(".")[0] or "agent"


def lock_path(path: Path) -> Path:
    """Where a reviewed spec for this file lives: <dir>/<agent_name>.spec.yaml, or the older <dir>/spec.yaml if present."""
    named, plain = path.parent / f"{agent_name(path)}.spec.yaml", path.parent / "spec.yaml"
    return plain if plain.is_file() and not named.is_file() else named


def load_spec(src: Path) -> AgentSpec:
    try:
        return AgentSpec.model_validate(yaml.safe_load(src.read_text(encoding="utf-8")))
    except FileNotFoundError:
        logger.error("no such spec file: {}", src)
        raise SystemExit(2)
    except Exception as e:                                           # pydantic ValidationError, yaml errors
        msg = "; ".join(x["msg"] for x in e.errors()) if hasattr(e, "errors") else str(e)
        logger.error("{} is not a valid AgentSpec: {}", src, msg)
        raise SystemExit(2)


def find_spec(path: Path, out: Path, sha: str) -> tuple[Path, str] | None:
    """The spec to reuse for exactly this version of the file: the reviewed spec next to it, else the previous
    extraction in `out`. A spec whose source_hash is missing or differs is named and skipped (the file changed)."""
    for cand, what in ((lock_path(path), "reviewed spec"), (out / "agent_spec.yaml", "previous extraction")):
        if not cand.is_file():
            continue
        try:
            got = (yaml.safe_load(cand.read_text(encoding="utf-8")) or {}).get("source_hash", "")
        except Exception:
            got = ""
        if got == sha:
            return cand, what
        logger.warning("{} {} is for another version of {} (source_hash {} != {}) - not reused", what, cand, path.name, got or "missing", sha)
    return None


@logger.catch(reraise=True)
def run(path: Path, out: Path, extractor: str = "auto", mode: str = "enforce", min_votes: int = 2, samples: int = 3,
        complete=None, verify: str = "auto", png: bool = True, lock: bool = False) -> dict:
    """extractor: auto = the reviewed spec next to the file or the previous extraction in `out` when either matches the
    file's hash, else the LLM; llm = always the LLM; yaml:<file> = that spec. lock = write the spec used in this run,
    stamped with the file's hash, next to the file so every later run reuses it without the LLM."""
    path, out = Path(path), Path(out)
    setup_logging(out)
    with stage("ingest"):
        doc = parse_markdown(path)
        logger.info("parsed {}: {} segments, {} code spans, hash {}", path.name, len(doc.segments), len(doc.code_spans), doc.sha)
    with stage("extract"):
        reused = find_spec(path, out, doc.sha) if extractor == "auto" else None
        if extractor.startswith("yaml:"):
            src = Path(extractor[5:])
            spec = load_spec(src)
            if spec.source_hash and spec.source_hash != doc.sha:
                logger.warning("{} was made for another version of {} (source_hash {} != {}): line numbers and quotes may be off", src, path.name, spec.source_hash, doc.sha)
            note = f"rules from {src} ({spec.provenance or 'no provenance recorded'})"
            logger.info("loaded DSL from {} ({} actions, {} rules)", src, len(spec.actions), len(spec.rules))
        elif reused:
            src, what = reused
            try:
                spec = load_spec(src)
            except SystemExit:
                logger.error("{} matches this file but does not load (an unfinished draft?): fix it and pass --extractor yaml:{}, or delete it to extract again", src, src)
                raise
            note = f"rules reused from the {what} {src} ({spec.provenance or 'no provenance recorded'}) - same file content, the LLM was not called"
            logger.info("{} {} matches this file (hash {}) - reused, LLM not called ({} actions, {} rules)", what, src, doc.sha, len(spec.actions), len(spec.rules))
        else:
            if complete is None:
                from .llm_client import complete
            try:
                spec = extract_spec(doc, agent_name(path), LLMExtractor(complete, samples=samples), min_votes=min_votes)
            except ExtractionError as e:
                draft = out / "agent_spec.yaml"
                draft.write_text(yaml.safe_dump(e.raw, sort_keys=False), encoding="utf-8")
                logger.error("extracted DSL is invalid: {}", "; ".join(x["msg"] for x in e.error.errors()))
                logger.error("draft written to {} - fix it, then rerun with --extractor yaml:{}", draft, draft)
                raise SystemExit(2)
            spec.provenance = (f"llm {os.environ.get('AZURE_OPENAI_DEPLOYMENT_NAME', '?')} samples={samples} min_votes={min_votes} "
                               f"{datetime.date.today().isoformat()}")
            note = f"rules extracted now ({spec.provenance}) - a draft until reviewed and locked"
        spec.source_hash = doc.sha
        write_spec(spec, out / "agent_spec.yaml")
        if lock:
            dst = lock_path(path)
            if not spec.provenance.startswith("reviewed"):
                spec.provenance = f"reviewed {datetime.date.today().isoformat()}" + (f"; {spec.provenance}" if spec.provenance else "")
            write_spec(spec, dst)
            logger.info("locked: {} will be reused for this version of {} until the file changes", dst, path.name)
    with stage("compile"):
        frag, cpn, m0, ctx = build(spec, mode)
    with stage("verify"):
        compiled = sum(1 for r in spec.rules if r.modality != "should")
        if verification_method(spec, verify) == "full":
            res = state_space_check(cpn, m0, ctx)
            res["method"] = "full"
        else:
            logger.info("{} rules: full state space not enumerable, using compositional verification", compiled)
            res = verify_large(spec, mode)
        n = composition_check(spec, samples=200)
    with stage("selfcheck"):
        res["enforcement"] = enforcement_witnesses(spec, frag, cpn, m0, ctx, res.get("witness"))
        logger.info("enforcement witnesses: {} of {} rules refuse a step on their own within {} markings{}",
                    len(res["enforcement"]["witnesses"]), compiled, res["enforcement"]["states"],
                    f"; none found for {', '.join(res['enforcement']['unwitnessed'])}" if res["enforcement"]["unwitnessed"] else "")
        replay(spec, frag, cpn, m0, ctx, res.get("witness"), res["enforcement"])
    with stage("report"):
        write_report(doc, spec, res, out, mode, note)
    with stage("emit"):
        py = emit_python(spec, frag, cpn, m0, out / f"{spec.name}_cpn.py")
        ok = run_generated(py)
        emit_json(cpn, m0, ctx, out / f"{spec.name}_cpn.json")
        if png:
            emit_dot(cpn, m0, out / f"{spec.name}_cpn.png")
        report = {"source": str(path), "actions": [a.id for a in spec.actions],
                  "rules": [json.loads(r.model_dump_json()) for r in spec.rules],
                  "state_space": {k: v for k, v in res.items() if k != "analyzer"},
                  "composition_traces_checked": n, "generated_file_reverified": ok}
        (out / "report.json").write_text(json.dumps(report, indent=2, default=str), encoding="utf-8")
        logger.info("report -> {}", out / "report.json")
    logger.bind(stage="monitor").info("end of pipeline run of {} (hash {}); drive the net yourself with: python examples/drive.py {} <action> [<action>:<param>=<value> ...]", path.name, doc.sha, out)
    return report


def main(argv=None):
    ap = argparse.ArgumentParser(prog="agentmd2cpn")
    ap.add_argument("agent_md", type=Path)
    ap.add_argument("-o", "--out", type=Path, default=Path("out"))
    ap.add_argument("--extractor", default="auto",
                    help="auto (default) = reuse the reviewed spec next to the file or the previous extraction when its hash matches the file, else the LLM | llm = always the LLM | yaml:<spec.yaml>")
    ap.add_argument("--lock", action="store_true", help="write the spec used in this run, stamped with the file's hash, next to the file (<name>.spec.yaml) for reuse")
    ap.add_argument("--mode", default="enforce", choices=["enforce", "monitor"])
    ap.add_argument("--min-votes", type=int, default=2)
    ap.add_argument("--samples", type=int, default=3)
    ap.add_argument("--verify", default="auto", choices=["auto", "full", "compositional"],
                    help="full = exhaustive state space (default up to 20 rules); compositional = conflicts + witness search")
    ap.add_argument("--no-png", action="store_true", help="skip the Graphviz drawing (slow and unreadable for large nets)")
    a = ap.parse_args(argv)
    if not a.agent_md.is_file():
        raise SystemExit(f"agentmd2cpn: no such file: {a.agent_md}")
    rep = run(a.agent_md, a.out, a.extractor, a.mode, a.min_votes, a.samples, verify=a.verify, png=not a.no_png, lock=a.lock)
    s = rep["state_space"]
    if s.get("method") == "compositional":
        print(f"{a.agent_md}: {len(rep['rules'])} rules, compositional check: conflicts={len(s['conflicts'])}, "
              f"consistent={s['done_reachable']}, explored={s['states']} markings{'' if s['complete'] else ' (bounded)'}, "
              f"traps={s['trap_states']}, dead={s['dead_transitions']}, generated file re-verified={rep['generated_file_reverified']}")
    else:
        print(f"{a.agent_md}: {len(rep['rules'])} rules, {s['states']} reachable markings, consistent={s['done_reachable']}, "
              f"traps={s['trap_states']}, dead={s['dead_transitions']}, generated file re-verified={rep['generated_file_reverified']}")


if __name__ == "__main__":
    main()
```

### B9. Replace `examples/drive.py` entirely (32 lines, hash `85d810aa`)

```python
"""Drive any generated net with a trace typed on the command line, e.g.
    python examples/drive.py out/release_agent run_build read_changelog deploy_to_production:day=friday
Each argument is an action id, optionally followed by :param=value pairs separated by commas.
The decisions are appended to <out>/monitor.log after the pipeline's own replay, and printed."""
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parents[1]))
import yaml
from loguru import logger

from agentmd2cpn.dsl import AgentSpec
from agentmd2cpn.logging_setup import setup_logging
from agentmd2cpn.runtime import Monitor
from agentmd2cpn.verify import build

out = Path(sys.argv[1])
setup_logging(out, level="ERROR", fresh=False)                # console quiet; decisions appended to monitor.log; pipeline.jsonl kept
log = out / "monitor.log"
before = log.stat().st_size if log.exists() else 0
logger.bind(stage="monitor").info("drive: {}", " ".join(sys.argv[2:]) or "(finish only)")
spec = AgentSpec.model_validate(yaml.safe_load((out / "agent_spec.yaml").read_text(encoding="utf-8")))
frag, cpn, m0, ctx = build(spec)
m = Monitor(cpn, m0, ctx, blame=frag.blame)
for arg in sys.argv[2:]:
    action, _, rest = arg.partition(":")
    params = dict(kv.split("=", 1) for kv in rest.split(",") if kv)
    m.try_action(action, **params)
print("finished:", m.finish())
with log.open("r", encoding="utf-8") as f:                    # print only this drive's lines
    f.seek(before)
    print(f.read())
```

### B10. Edit `agentmd2cpn/dsl.py`

Find `class AgentSpec(BaseModel):`. Directly under its line `source_file: str = ""` add these
two lines (same indentation):

```python
    source_hash: str = ""        # Document.sha of the file this spec was extracted from or reviewed against
    provenance: str = ""         # who produced the rules: "llm <deployment> samples=3 min_votes=2 <date>" or "reviewed"
```

### B11. Edit `agentmd2cpn/ingest.py` (five small insertions)

1. With the imports at the top, add a line `import hashlib`.
2. In `class Document`, after the line `imports: list[str] = field(default_factory=list)` add:
   ```python
       sha: str = ""                                   # of the text (and of every imported file): identifies this version of the file
   ```
3. In `parse_markdown`, directly after the line `doc = Document(file=str(path))` add:
   ```python
       digest = hashlib.sha256(text.encode("utf-8"))
   ```
4. In the loop over imports (`for imp in list(doc.imports):`), directly after the line
   `doc.code_spans.extend(sub.code_spans)` add (same indentation as that line):
   ```python
               digest.update(sub.sha.encode())
   ```
5. Directly before the final `return doc` of `parse_markdown` add:
   ```python
       doc.sha = digest.hexdigest()[:16]
   ```

### B12. Edit `agentmd2cpn/extract.py` (two edits in `ground_and_vote`)

1. Find the loop that assigns the ids R1, R2, … — it starts with
   `for key, n in counts.most_common():` followed by `r = keep[key]`. Replace those two lines
   with:
   ```python
       for key in sorted(counts, key=lambda k: (keep[k].line_begin, keep[k].line_end, k)):   # document order: stable ids across runs
           n, r = counts[key], keep[key]
   ```
   (Rules are now numbered top-to-bottom of the file instead of by vote count, so the same rule
   set always gets the same ids.)
2. Find the dictionary `raw = {"name": name, "source_file": doc.file, ...`. After
   `"source_file": doc.file,` insert `"source_hash": doc.sha,`.

### B13. Edit `agentmd2cpn/emit.py` (two edits)

1. The template starts with
   `TEMPLATE = _ENV.from_string('''"""{{ name }} - coloured Petri net generated by agentmd2cpn from {{ source }}`
   followed by an empty line. Replace those first two lines with these three:
   ```python
   TEMPLATE = _ENV.from_string('''"""{{ name }} - coloured Petri net generated by agentmd2cpn from {{ source }}{% if source_hash %} (hash {{ source_hash }}){% endif %}
   {% if provenance %}rules: {{ provenance }}
   {% endif %}
   ```
   (the next line, `Every rule below was compiled from one sentence …`, stays).
2. In the `TEMPLATE.render(...)` call, replace `name=spec.name, source=spec.source_file, filename=out.name,`
   with `name=spec.name, source=spec.source_file, source_hash=spec.source_hash, provenance=spec.provenance, filename=out.name,`.

### B14. Check

```powershell
python -m pytest tests -q
```

→ `N+16 passed` (the 6 from Part A, 3 + 2 + 2 new files, +1 in ingest, cli, extract). Then:

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png
type out\release_agent\REPORT.md
type out\release_agent\monitor.log
```

Expected last line of the run: `16 rules, 2569 reachable markings, consistent=True, traps=0,
dead=['force_push'], generated file re-verified=True`. In the log, a new stage line:
`selfcheck | enforcement witnesses: 16 of 16 rules refuse a step on their own within … markings`.

`REPORT.md` has these sections, in this order: the counts line, `rules from
examples/release_agent/spec.yaml (no provenance recorded)`, `## Verdict: CONSISTENT - every rule
can be satisfied in one run` with `shortest complete run: read_changelog -> run_build -> run_tests
-> tag_release -> post_release_summary`, `## Actions that can never happen` (`force_push - forbidden
by R11 … [expected]`), `## Enforcement check - one step the net refuses because of that rule alone`
with sixteen lines like

```text
- R1  init: first action must be read_changelog   (line 13: "Start by reading the changelog.")  --  after (start): run_build -> BLOCKED by R1
- R9  limit: retry_deployment at most 2 times   (line 25: "Do not retry a failed deployment more than 2 times.")  --  after read_changelog -> retry_deployment -> retry_deployment: retry_deployment -> BLOCKED by R9
- R11  forbid: force_push is never allowed   (line 27: "Never force-push to any branch.")  --  after read_changelog: force_push -> BLOCKED by R11
- R13  responded_existence: if make_hotfix happens, ensure_ticket_exists must happen too   (line 29: ...)  --  after read_changelog -> run_build -> run_tests -> tag_release -> make_hotfix -> post_release_summary: finishing -> BLOCKED by R13
```

then `## Rules` and `## Actions`. `monitor.log` was written by the pipeline run itself:

```text
2026-11-04 … | - | complete run from the verifier (5 steps):
2026-11-04 … | - | ALLOW read_changelog
2026-11-04 … | - | ALLOW run_build
2026-11-04 … | - | ALLOW run_tests
2026-11-04 … | - | ALLOW tag_release
2026-11-04 … | - | ALLOW post_release_summary
2026-11-04 … | - | ALLOW END
2026-11-04 … | - | DONE: all rules satisfied
2026-11-04 … | - | probe R1 (line 13: "Start by reading the changelog.") after (start):
2026-11-04 … | R1 | BLOCK run_build
2026-11-04 … | - | probe R2 (line 14: "Run the build before running the tests.") after read_changelog:
2026-11-04 … | R2 | BLOCK run_tests
…  (one probe + BLOCK pair per rule, R1 … R16)
2026-11-04 … | - | end of pipeline run of .Agent.md (hash …); drive the net yourself with: python examples/drive.py out/release_agent <action> …
```

Every line carries today's time; the file is rewritten on every run. Then drive it yourself —
the decisions are appended after the pipeline's lines and printed:

```powershell
python examples/drive.py out/release_agent run_build read_changelog run_build run_tests tag_release post_release_summary
```

→ `R1 | BLOCK run_build`, `ALLOW read_changelog`, `ALLOW run_build`, `ALLOW run_tests`,
`ALLOW tag_release`, `ALLOW post_release_summary`, `ALLOW END`, `DONE: all rules satisfied`.

Part B done.

---

## Part C — "finish by A or B", the prompt, the review agent

### C1. Two tests (append)

Append to `tests/test_templates.py`:

```python
def test_init_and_end_accept_a_list_of_alternatives():
    from agentmd2cpn.templates import build_dfa, english
    e = build_dfa("end", {"action": ["approve", "reject"]}).complete(["read", "approve", "reject"])
    assert e.run(["read", "approve"]) and e.run(["read", "reject"]) and not e.run(["approve", "read"]) and not e.run([])
    i = build_dfa("init", {"action": ["a", "b"]}).complete(["a", "b", "c"])
    assert i.run(["b", "c"]) and i.run(["a"]) and not i.run(["c", "a"]) and not i.run([])
    assert english("end", {"action": ["approve", "reject"]}) == "last action must be approve or reject"
    assert english("end", {"action": "approve"}) == "last action must be approve"           # unchanged for one action
```

Append to `tests/test_dsl.py`:

```python
def test_action_lists_only_for_init_and_end():
    import pytest
    from agentmd2cpn.dsl import AgentSpec
    base = {"name": "t", "actions": [{"id": "a"}, {"id": "b"}]}
    AgentSpec.model_validate({**base, "rules": [{"id": "R1", "kind": "end", "args": {"action": ["a", "b"]}}]})
    with pytest.raises(ValueError, match="only allowed for init and end"):
        AgentSpec.model_validate({**base, "rules": [{"id": "R1", "kind": "forbid", "args": {"action": ["a", "b"]}}]})
    with pytest.raises(ValueError, match="unknown action 'zzz'"):
        AgentSpec.model_validate({**base, "rules": [{"id": "R1", "kind": "end", "args": {"action": ["a", "zzz"]}}]})
```

### C2. Edit `agentmd2cpn/templates.py`

Find the two functions `def init(a: str) -> DFA:` and `def end(a: str) -> DFA:` (they are next to
each other). Delete both and put these three functions in their place:

```python
def _one_of(a) -> list[str]:
    """init / end take one action or a list of alternatives ("finish by approving or requesting changes")."""
    return [str(x) for x in a] if isinstance(a, (list, tuple)) else [str(a)]


def init(a) -> DFA:                                     # init(a): first action is a (or one of a list)
    acts = _one_of(a)
    delta = {("RUN", "*"): "RUN"}
    delta.update({("START", x): "RUN" for x in acts})
    return DFA(["START", "RUN"], "START", {"RUN"}, delta, acts, universal=True,
               note=f"first action must be {' or '.join(acts)}")


def end(a) -> DFA:                                      # end(a): last action is a (or one of a list)
    acts = _one_of(a)
    delta = {("NOT_A", "*"): "NOT_A", ("LAST_A", "*"): "NOT_A"}
    delta.update({("NOT_A", x): "LAST_A" for x in acts})
    delta.update({("LAST_A", x): "LAST_A" for x in acts})
    return DFA(["NOT_A", "LAST_A"], "NOT_A", {"LAST_A"}, delta, acts, universal=True,
               note=f"last action must be {' or '.join(acts)}")
```

### C3. Edit `agentmd2cpn/dsl.py`

In the `AgentSpec` validator find the loop that checks that every action named in a rule's args
is declared. It begins with `for key in ("action", "first", "then", "trigger", "approver", "a", "b"):`
and raises `unknown action …`. Replace the whole loop with:

```python
            for key in ("action", "first", "then", "trigger", "approver", "a", "b"):
                if key not in r.args:
                    continue
                vals = r.args[key] if isinstance(r.args[key], list) else [r.args[key]]
                if isinstance(r.args[key], list) and (r.kind not in ("init", "end") or not vals):
                    raise ValueError(f"rule {r.id} ({r.kind}): a list of actions is only allowed for init and end (\"A or B\")")
                for v in vals:
                    if v not in acts:
                        raise ValueError(f"rule {r.id}: unknown action {v!r} in '{key}'")
```

(keep the indentation level of the loop you replaced).

### C4. Replace `agentmd2cpn/prompt/extraction.yaml` entirely (22 lines)

Same prompt as yours, with two new cues: "finish by A or B" is ONE `end` rule with a list, and
a conditional sentence ("if you find X, Y must …") makes the event an action and uses `response`.
The `{schema}`, `{kinds}`, `{known}`, `{segments}` placeholders and the doubled braces stay
exactly as they are — the text still goes through `.format()`.

```yaml
prompt: |
  You convert an agent instruction file into a strict JSON object that matches this JSON schema:
  {schema}

  Rules:
  - `actions` is the closed list of things the agent can do: one snake_case verb_object id per activity, reused in every rule that mentions it. People, documents and places are not actions.
  - Every value in `actions[*].params` must be a key of `data`. Declare each such type in `data` as {{"type": "enum", "values": [...]}} with 2-4 lowercase abstract values that the rules need to distinguish (e.g. ["deploy", "other"], ["friday", "other"]). Never use free text as a type.
  - Every rule must be ONE of these kinds: {kinds}.
    args by kind: forbid/require/limit/exactly -> {{"action": id[, "n": int]}};  init/end -> {{"action": id}} or, for "A or B", {{"action": [idA, idB]}} (ONE rule, never two end rules);
    forbid_when/scope -> {{"action": id, "where": "<param> == '<value>'"}};
    precedence/alt_precedence -> {{"first": id, "then": id}};  approval -> {{"approver": id, "action": id}};
    response/chain_response -> {{"trigger": id, "then": id}};  responded_existence/not_coexistence/not_succession -> {{"a": id, "b": id}};
    phase_sequence -> {{"phases": [{{"id": name, "allows": [ids]}}]}}.
  - `quote` must be copied VERBATIM from the numbered segments below and `line_begin/line_end` must be that segment's lines.
  - Cues: "immediately after A, B" -> chain_response; "after A, B" -> response; "A before B" / "B only after A" -> precedence; "a fresh/new A before every B" -> alt_precedence or approval; "exactly N times" -> exactly; "at most N" / "not more than N" -> limit; "after A never B" -> not_succession; "never both A and B" -> not_coexistence; "if A then B (anywhere)" -> responded_existence; "start by A" -> init; "finish by A" / "finish by A or B" -> end (one rule); "if you find/see/detect X, Y must ..." -> declare the event X as an action and use response (X then Y), never an unconditional require; "never A" -> forbid; "never A when/on <value>" -> forbid_when with a param; "only A when/under <value>" -> scope with a param.
  - A sentence that merely describes the agent, and style advice, produce no rule (or modality "should"); never invent an action for them.
  - If the file has no constraints, return an empty `rules` list. Never invent rules.

  {known}Segments:
  {segments}

  Return only JSON.
```

**Check:**

```powershell
python -m pytest tests -q
```

→ `N+18 passed`.

### C5. Create `examples/review/spec.yaml` (152 lines, hash `f410de19`)

This is your gpt-4o extraction of `review.agent.md` after the review: line 16 ("approving or
requesting changes") is one `end` rule with both actions instead of two contradicting ones (R5
removed); the redundant phase rule R9 removed; line 22 made conditional (new action
`find_security_vulnerability`, R10 = `response`); the unused `issue_type` removed. Ids kept.

```yaml
name: review
source_file: examples/review/review.agent.md
actions:
- id: read_diff
  description: Read the pull request diff
- id: run_tests
  description: Run the automated test suite
- id: post_review_summary
  description: Post the review summary
- id: approve_pr
  description: Approve the pull request
  params:
    test_status: test_status
- id: request_changes
  description: Request changes on the pull request
  params:
    test_status: test_status
- id: merge_pr
  description: Merge the pull request
- id: find_security_vulnerability
  description: Find a security vulnerability while reviewing
- id: create_ticket
  description: Create a ticket for a security vulnerability
- id: edit_files_pr_branch
  description: Edit files in the PR branch
  params:
    branch_scope: branch_scope
data:
  test_status:
    type: enum
    values:
    - passing
    - failing
    - other
  branch_scope:
    type: enum
    values:
    - pr_branch
    - other
roles: []
rules:
- id: R1
  kind: init
  args:
    action: read_diff
  modality: must
  confidence: 0.9
  votes: 3
  source:
    file: examples/review/review.agent.md
    line_begin: 13
    line_end: 13
    heading: ''
    quote: Start by reading the pull request diff.
- id: R2
  kind: precedence
  args:
    first: run_tests
    then: post_review_summary
  modality: must
  confidence: 0.9
  votes: 3
  source:
    file: examples/review/review.agent.md
    line_begin: 14
    line_end: 14
    heading: ''
    quote: Run the automated test suite before posting any comments.
- id: R3
  kind: exactly
  args:
    action: post_review_summary
    n: 1
  modality: must
  confidence: 0.9
  votes: 3
  source:
    file: examples/review/review.agent.md
    line_begin: 15
    line_end: 15
    heading: ''
    quote: Post the review summary exactly once per run.
- id: R4
  kind: end
  args:
    action:
    - approve_pr
    - request_changes
  modality: must
  confidence: 0.8
  votes: 3
  source:
    file: examples/review/review.agent.md
    line_begin: 16
    line_end: 16
    heading: ''
    quote: Finish by approving or requesting changes.
- id: R6
  kind: forbid_when
  args:
    action: approve_pr
    where: test_status == 'failing'
  modality: must
  confidence: 0.9
  votes: 3
  source:
    file: examples/review/review.agent.md
    line_begin: 20
    line_end: 20
    heading: ''
    quote: Never approve a pull request with failing tests.
- id: R7
  kind: forbid
  args:
    action: merge_pr
  modality: must
  confidence: 0.9
  votes: 3
  source:
    file: examples/review/review.agent.md
    line_begin: 21
    line_end: 21
    heading: ''
    quote: Never merge a pull request directly.
- id: R8
  kind: scope
  args:
    action: edit_files_pr_branch
    where: branch_scope == 'pr_branch'
  modality: must
  confidence: 0.9
  votes: 2
  source:
    file: examples/review/review.agent.md
    line_begin: 27
    line_end: 27
    heading: ''
    quote: Only edit files in the PR branch.
- id: R10
  kind: response
  args:
    trigger: find_security_vulnerability
    then: create_ticket
  modality: must
  confidence: 0.86
  votes: 2
  source:
    file: examples/review/review.agent.md
    line_begin: 22
    line_end: 22
    heading: ''
    quote: If you find a security vulnerability, a ticket must be created for it.
```

### C6. Check

```powershell
python -m agentmd2cpn examples/review/review.agent.md -o out/review --extractor yaml:examples/review/spec.yaml --no-png --lock
type out\review\monitor.log
```

→ `examples\review\review.agent.md: 8 rules, 14 reachable markings, consistent=True, traps=0,
dead=['merge_pr'], generated file re-verified=True`; log line `selfcheck | enforcement
witnesses: 8 of 8 rules …`; `locked: examples\review\spec.yaml will be reused for this version of
review.agent.md until the file changes`. `monitor.log`:

```text
… | - | complete run from the verifier (4 steps):
… | - | ALLOW read_diff
… | - | ALLOW run_tests
… | - | ALLOW post_review_summary
… | - | ALLOW request_changes{'test_status': 'other'}
… | - | ALLOW END
… | - | DONE: all rules satisfied
… | - | probe R1 (line 13: "Start by reading the pull request diff.") after (start):
… | R1 | BLOCK run_tests
… | - | probe R2 (line 14: "Run the automated test suite before posting any comments.") after read_diff:
… | R2 | BLOCK post_review_summary
… | - | probe R6 (line 20: "Never approve a pull request with failing tests.") after read_diff:
… | R6 | BLOCK approve_pr{'test_status': 'failing'}
… | - | probe R7 (line 21: "Never merge a pull request directly.") after read_diff:
… | R7 | BLOCK merge_pr
… | - | probe R8 (line 27: "Only edit files in the PR branch.") after read_diff:
… | R8 | BLOCK edit_files_pr_branch{'branch_scope': 'other'}
… | - | probe R3 (line 15: "Post the review summary exactly once per run.") after read_diff -> approve_pr{'test_status': 'passing'}:
… | R3 | BLOCK END
… | - | cannot finish: some rule is not satisfied yet
… | - | probe R4 (line 16: "Finish by approving or requesting changes.") after read_diff -> run_tests -> post_review_summary:
… | R4 | BLOCK END
… | - | cannot finish: some rule is not satisfied yet
… | - | probe R10 (line 22: "If you find a security vulnerability, a ticket must be created for it.") after … find_security_vulnerability -> approve_pr{'test_status': 'passing'}:
… | R10 | BLOCK END
… | - | cannot finish: some rule is not satisfied yet
… | - | end of pipeline run of review.agent.md (hash …); …
```

Part C done.

---

## Part D — the client: no sampling

### D1. Edit `agentmd2cpn/llm_client.py`

In the `client.chat.completions.create(...)` call change `temperature=0.2` to `temperature=0`
and add the argument `seed=7` (anywhere in the same call). gpt-4o accepts both.

### D2. Check

```powershell
python -m agentmd2cpn.llm_client
```

→ `{"ok": true}` (the smoke test that is already in the file).

Part D done.

---

## Part E — lock the examples and prove the goal

### E1. Lock the reviewed specs (one run each; `--lock` writes the file's hash into the spec)

```powershell
python -m agentmd2cpn examples/release_agent/.Agent.md -o out/release_agent --extractor yaml:examples/release_agent/spec.yaml --no-png --lock
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/orchestrator --extractor yaml:examples/orchestrator/spec.yaml --no-png --lock
```

For expert and universal use the reviewed spec you have for them — `examples/expert/spec.yaml`
and `examples/universal/spec.yaml` if those files exist, otherwise the `agent_spec.yaml` of your
last good run of each (for example `out\expert_run_04\agent_spec.yaml`); `--lock` then creates
`examples/expert/expert.spec.yaml` / `examples/universal/universal.spec.yaml` next to the file:

```powershell
python -m agentmd2cpn examples/expert/expert.agent.md -o out/expert --extractor yaml:<your reviewed expert spec> --no-png --lock
python -m agentmd2cpn examples/universal/universal.agent.md -o out/universal --extractor yaml:<your reviewed universal spec> --no-png --lock
```

Expected summary lines: `16 rules, 2569 …`; `47 rules, compositional check: conflicts=0,
consistent=True, explored=5002 markings (bounded), traps=0, dead=[…five…]` with
`enforcement witnesses: about 27 of 38 rules …; none found for …` (the rest are listed in
`REPORT.md` as *implied by other rules or deeper than the search went* — for instance
`invoke_impact_analysis only after confirm_configuration` can never refuse alone because
`invoke_trigger_detection only after confirm_configuration` always refuses first; that is a true
statement about the file); `4 rules, 48 …` with `4 of 4`; `0 rules, 2 …` with
`REPORT.md` saying `## Verdict: NOTHING TO ENFORCE`. Each locked `spec.yaml` now starts with
`source_hash: …` and `provenance: reviewed 2026-11-04`.

### E2. Three runs, no model, identical

```powershell
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/o1 --no-png
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/o2 --no-png
python -m agentmd2cpn examples/orchestrator/orchestrator.agent.md -o out/o3 --no-png
fc out\o1\orchestrator_cpn.py out\o2\orchestrator_cpn.py
fc out\o2\orchestrator_cpn.py out\o3\orchestrator_cpn.py
fc out\o1\agent_spec.yaml out\o3\agent_spec.yaml
```

Each run logs `extract | reviewed spec examples\orchestrator\spec.yaml matches this file (hash …)
- reused, LLM not called (44 actions, 47 rules)`; every `fc` prints `FC: no differences
encountered`. This is the determinism: same file → same spec → same net → same verdict, and no
model was asked.

### E3. A changed file is detected

```powershell
copy examples\review\review.agent.md examples\review\review2.agent.md
```

Open `examples\review\review2.agent.md`, change one word anywhere, save, then:

```powershell
python -m agentmd2cpn examples/review/review2.agent.md -o out/review2 --no-png
```

→ no lock matches (`review2.spec.yaml` does not exist; `spec.yaml` is for another hash:
`… is for another version of review2.agent.md (source_hash … != …) - not reused`), the model
is called once, `out\review2\REPORT.md` opens with `## Review first`. Delete
`examples\review\review2.agent.md` and `out\review2` afterwards.

Part E done — the goal is reached. What is left is paperwork.

---

## Part F — documentation, hygiene, commit, push

### F1. `.gitignore`

Add three lines (keep the ones that are there: `out/`, `.env`, `.venv/`, `__pycache__/`, …):

```text
*.egg-info/
dist/
build/
```

### F2. README and ARCHITECTURE (Copilot)

```text
Update README.md and docs/ARCHITECTURE.md only (no code, no tests). Keep their structure; change these points:
1. Input: any markdown file - .Agent.md, *.agent.md, *.skills.md, AGENTS.md or any .md - not only .Agent.md. The output name is the file name up to its first dot (a dot-file such as .Agent.md takes its folder's name).
2. Usage: `agentmd2cpn file.md -o out/x` (default --extractor auto: the reviewed spec next to the file, else the previous extraction in the output folder, when its hash matches the file, else the model); `--extractor llm` (always the model); `--extractor yaml:<spec.yaml>`; `--lock` (write the spec used in this run, stamped with the file's hash, next to the file so later runs reuse it); `--verify auto|full|compositional`; `--no-png`. Runtime: `python examples/drive.py out/x <action> [<action>:<param>=<value> ...]` appends its decisions to monitor.log.
3. Outputs in out/<name>/: agent_spec.yaml (the DSL; carries source_hash and provenance), REPORT.md (Review first - rules the samples disagreed on and rule-like sentences without a rule; Verdict; Conflicts as sentences with line numbers and the dead end as the Monitor sees it; Actions that can never happen; Enforcement check - one step the net refuses because of each rule alone; Rules table; Actions), monitor.log (the verifier's complete run and one refused step per rule, replayed on the Monitor, plus drive.py decisions), <name>_cpn.py (standalone, re-verifies itself; header names the source file, its hash and the provenance of the rules), <name>_cpn.json, <name>_cpn.png, pipeline.jsonl, report.json.
4. A section "Reproducibility" (one paragraph): the model reads a file once per version; every rule quotes the sentence it came from and must be found at the cited lines; the reviewed spec is locked to the file's hash and reused; a changed file is detected, read again and reported; everything after the YAML (compile, verify, report, emit, monitor) is deterministic.
5. A section "Workflow for a new file": run; read Review first and Conflicts in REPORT.md; fix agent_spec.yaml if a sentence was misread; rerun with --extractor yaml:<that file> --lock; from then on every run is identical and calls no model.
6. Layout: add agentmd2cpn/report.py (REPORT.md), agentmd2cpn/selfcheck.py (enforcement witnesses and the Monitor replay), agentmd2cpn/prompt/extraction.yaml (the extraction prompt), examples/review/ (the code-review agent example), and the new test files.
7. Tests: the count is what `python -m pytest tests -q` prints now (run it and use that number).
8. Limitations: add - a rule for which no step refused by it alone is found within the search bound is reported as implied by other rules or deeper than the search went, not as proven; phase_sequence allows a phase with no action of its own (seen in the review agent); conditions are checked on the parameters the caller passes, not on what the called tool actually returned; init/end accept a list of alternatives, other kinds do not.
9. docs/ARCHITECTURE.md: in the pipeline/stage list add `selfcheck` (enforcement witnesses, Monitor replay) and `report` (REPORT.md) between verify and emit, and the auto/lock logic of the extract stage (lock_path, find_spec, source_hash).
Do not change any number you cannot read from the current outputs. Then show me the diff.
```

Read the diff. The numbers it writes must be the ones you saw in Parts B–E.

### F3. Commit and push

Ask your mentor first whether `examples/orchestrator/` (company-internal) may be committed; if
not, leave it out of the `git add` below.

```powershell
git status
git add .gitignore pyproject.toml README.md .env.example requirements.txt agentmd2cpn tests docs examples/release_agent examples/expert examples/universal examples/review examples/drive.py
git add examples/orchestrator        # only if the mentor said yes
git status
```

The second `git status` must show **no** `.env`, no `out/`, no `agentmd2cpn.egg-info`, no
`dist/`, no `build/`. If any of them appears, stop: `git reset` and fix `.gitignore`. Then:

```powershell
git commit -m "agentmd2cpn: robustness, REPORT.md, lock (one extraction per file version), enforcement check, init/end alternatives, package"
git push -u origin feat/text2dsl
```

Open the branch on GitHub and confirm `.env` is not there and `agentmd2cpn/`, `tests/`,
`examples/`, `docs/`, `README.md` are.

Done.

---

## What to say to the mentor (three sentences)

The extractor is a reader: it proposes rules, each tied to the sentence it came from, and is
asked once per version of a file. The verifier is exact: it names the sentences that cannot hold
together, the dead ends, the dead actions, and for every rule a concrete step the net refuses
because of it, replayed in the runtime log. The reviewed reading is locked to the file's hash, so
the result is reproducible by construction; a changed file is detected and read again.

Source for the architecture: [nl2spec (Cosler, Hahn, Mendoza, Schmitt, Trippel; arXiv 2303.04864, CAV 2023)](https://www.arxiv.org/abs/2303.04864)
