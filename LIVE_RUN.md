# agentmd2cpn — a live run on a file the package has never seen, then two changes to it

What this shows, in order: a new agent file with real Copilot tools goes through the whole pipeline
(§ 2); one sentence of it is changed so that a **different action** is required (§ 3); one sentence is
added that gives an action a **parameter** (§ 4); and the first version comes back without a model call
(§ 5). § 6 says what each stage guarantees after this, and what it does not.

Every number below is for the *expected reading* of the file — the rule kinds in the table of § 2.
The model's action names are its own; where the text says `read_manifest`, use the id your `extract`
log printed. Where a sentence is ambiguous (§ 2 names the one), the kind may differ; that is what the
review step is for.

---

## 0. First, one small change: a changed file keeps its action names

When you edit the file, its hash changes and the model reads it again — and may rename the actions,
which would make every command of § 3 and § 4 need new names. With this change, the ids of the earlier
reading are offered to the model (`Known action ids from a previous pass — reuse them where they fit`,
the sentence the extractor already uses between its own samples). `--extractor llm` stays a fresh
reading.

`tests/test_reread.py` (24 lines, hash `20b880dc`; delivered as a file):

```python
"""A file that changed since it was last read is read again with its earlier action ids offered to the model."""
import json

import yaml

from agentmd2cpn.cli import run

MD = '---\nname: "t"\n---\n# T\n\n- Start by reading the request.\n'
ANSWER = json.dumps({"actions": [{"id": "read_request"}],
                     "rules": [{"kind": "init", "args": {"action": "read_request"}, "quote": "Start by reading the request.", "line_begin": 6, "line_end": 6}]})


def test_a_changed_file_is_read_again_with_its_earlier_action_ids(tmp_path):
    md, out = tmp_path / "t.agent.md", tmp_path / "out"
    md.write_text(MD, encoding="utf-8")
    out.mkdir()
    old = {"name": "t", "source_hash": "0000000000000000", "actions": [{"id": "read_request"}, {"id": "answer"}], "rules": []}
    (out / "agent_spec.yaml").write_text(yaml.safe_dump(old), encoding="utf-8")        # a reading of another version of the file
    prompts = []
    run(md, out, complete=lambda p: prompts.append(p) or ANSWER, png=False)
    assert len(prompts) == 3 and all("read_request" in p and "answer" in p for p in prompts)   # offered from the first sample on
    prompts.clear()
    run(md, out, extractor="llm", complete=lambda p: prompts.append(p) or ANSWER, png=False)
    assert "answer" not in prompts[0]                                                 # --extractor llm: a fresh reading, nothing offered
```

```text
Rule: nothing in agentmd2cpn/ may name any agent file's actions, rules, tools or paths; do not edit the original test files (test_cli, test_lock, test_report, test_selfcheck, test_emit, test_extract, test_ingest, test_dsl, test_templates, test_compile, test_compose, test_verify, test_verify_large, test_runtime, test_fastcpn, test_llm_client, test_tools, test_vote, test_reread); the test files you created you may update to the current behaviour; run python -m pytest tests -q at the end and show the count.
Make tests/test_reread.py pass (read it first). In cli.py's extract stage, when the extractor is auto and no spec matches the file's hash, read the action ids of the reviewed spec next to the file (else of the previous extraction in the out folder), whatever version of the file it was made for, log "the file changed since it was last read: its <n> action ids are offered to the model so names stay the same where the actions did", and pass them to extract_spec(..., known=ids); LLMExtractor.extract_samples(doc, known=None) starts its known-ids set from them instead of empty. With --extractor llm nothing is offered.
```

→ previous count + 1, all passing.

---

## 1. The file

`examples\deps_agent\deps_agent.agent.md` (26 lines, hash `040eb5f8`; delivered as a file):

```markdown
---
name: "deps_agent"
description: "Updates the dependencies of one project per run and leaves the result for review"
tools: [read, search, fetch, execute, edit]
---
# Dependency Update Agent

You update the dependencies of one project per run, in the order below.

## Run sequence

1. Start by reading the dependency manifest.
2. Run the test suite before updating any dependency.
3. Read a dependency's changelog before updating it.
4. After every update, run the test suite again.
5. Finish by writing the update summary or by reporting that nothing needs updating.

## Safety

- Never edit the lockfile by hand.
- Never publish the package.
- Do not retry a failed install more than 2 times.

## Style

- Keep the summary short.
```

**The tools come from VS Code, not from me.** Open the file in VS Code and use *Configure Tools…* (the
tools action VS Code shows in the editor title of an agent file); tick Read, Search, Fetch, Execute and
Edit — VS Code rewrites the `tools:` line with its own names. If what it writes differs from the five
above (another version of VS Code names them differently), keep VS Code's line: the pipeline reads the
line as written, and no name in it is known to the package:

```powershell
Get-ChildItem agentmd2cpn -Recurse -Include *.py,*.yaml,*.yml,*.md,*.txt,*.j2 | Select-String -CaseSensitive -Pattern "\b(read_manifest|update_dependency|read_changelog|retry_install|lockfile|changelog|deps_agent|fetch)\b"
```

→ nothing. (`execute` is left out of the pattern because the word is in one comment of dsl.py, *who may
execute*, about roles. Changing the tool names changes the hash, so configure the tools *before* the
first run.)

---

## 2. The run

```powershell
agentmd2cpn examples\deps_agent\deps_agent.agent.md
```

What the stages print, with the expected reading:

```text
ingest   | parsed deps_agent.agent.md: 14 segments, 0 code spans, hash <16 hex>
extract  | R1 init: first action must be read_manifest
extract  | R2 precedence: update_dependency only after run_tests
extract  | R3 precedence: update_dependency only after read_changelog
extract  | R4 response: after update_dependency, run_tests must eventually follow
extract  | R5 end: last action must be write_summary or report_no_updates
extract  | R6 forbid: edit_lockfile is never allowed
extract  | R7 forbid: publish_package is never allowed
extract  | R8 limit: retry_install at most 2 times
extract  | tool: read_manifest -> read (3/3 samples)
extract  | tool: run_tests -> execute (3/3 samples)
extract  | tool: read_changelog -> fetch (3/3 samples)
extract  | tool: update_dependency -> execute (3/3 samples)
extract  | tool: write_summary -> edit (3/3 samples)
extract  | tool: report_no_updates -> none (3/3 samples)
extract  | tool: edit_lockfile -> edit (3/3 samples)
extract  | tool: publish_package -> execute (3/3 samples)
extract  | tool: retry_install -> execute (3/3 samples)
compile  | compiled 8 rules into 17 fragments
compile  | fused 17 fragments -> 8 places, 9 transitions, 53 arcs
compile  | lowered to cpnpy: 10 places, 10 transitions, 87 arcs
verify   | state space: 43 markings, 129 arcs
verify   | dead transitions (can never fire): ['edit_lockfile', 'publish_package']
verify   | shortest trace that satisfies every rule: read_manifest -> report_no_updates
verify   | tools: 5 declared (read, search, fetch, execute, edit); 8 of 9 actions are tool calls, 1 need no tool
verify   | with the declared tools only, every rule can be satisfied
selfcheck| enforcement witnesses: 8 of 8 rules refuse a step on their own within 15 markings
monitor  | ... one BLOCK per rule ...
examples\deps_agent\deps_agent.agent.md: 8 rules, 43 reachable markings, consistent=True, traps=0, dead=['edit_lockfile', 'publish_package']
```

The expected reading, sentence by sentence — check your `extract` lines against it:

| line | sentence | kind | tool of the action |
|---|---|---|---|
| 12 | Start by reading the dependency manifest. | `init` | read |
| 13 | Run the test suite before updating any dependency. | `precedence` run_tests → update_dependency | execute, execute |
| 14 | Read a dependency's changelog before updating it. | `precedence` read_changelog → update_dependency (*ambiguous: `alt_precedence` — a fresh read before each update — is also a fair reading; both are right, the numbers differ*) | fetch (or read) |
| 15 | After every update, run the test suite again. | `response` update_dependency → run_tests | — |
| 16 | Finish by writing the update summary or by reporting that nothing needs updating. | `end` with two alternatives | edit; none (a message) |
| 20 | Never edit the lockfile by hand. | `forbid` | edit |
| 21 | Never publish the package. | `forbid` | execute |
| 22 | Do not retry a failed install more than 2 times. | `limit`, n = 2 | execute |
| 26 | Keep the summary short. | no rule (or `should`) | — |

Three things worth noticing in the log: `search` is declared and no action uses it (the file never
asks the agent to search — the report says so); `execute` is shared by four actions (a gate on the
tool name alone could not tell `run_tests` from `publish_package` — the argument condition of the
later mapping is for exactly that); and the two forbidden actions are *dead* in the net, as they
should be.

Review, then lock (B2/B3 of the playbook): if a kind differs from the table and the table is right,
edit `out\deps_agent\agent_spec.yaml` (kind or args, keep the quote); in either case:

```powershell
agentmd2cpn examples\deps_agent\deps_agent.agent.md --extractor yaml:out\deps_agent\agent_spec.yaml --lock
agentmd2cpn tools examples\deps_agent\deps_agent.agent.md
```

```text
declared tools: read, search, fetch, execute, edit

- read_manifest -> read
- run_tests -> execute
- read_changelog -> fetch
- update_dependency -> execute
- write_summary -> edit
- report_no_updates -> no tool needed (a decision, a message, an event)
- edit_lockfile -> edit
- publish_package -> execute
- retry_install -> execute

tools shared by several actions (a gate on the tool name alone cannot tell them apart): execute: run_tests, update_dependency, publish_package, retry_install; edit: write_summary, edit_lockfile
declared tools no action uses: search

With the declared tools only (an action that needs an undeclared tool cannot happen) every rule can be satisfied; a complete run: read_manifest -> write_summary
```

A run the rules refuse — an update before its changelog and before the tests, and a finish without
the test run that R4 demands:

```powershell
agentmd2cpn drive out\deps_agent read_manifest update_dependency run_tests read_changelog update_dependency write_summary --png --name early_update
```

```text
   - | ALLOW read_manifest
R2, R3 | BLOCK update_dependency
   - | ALLOW run_tests
   - | ALLOW read_changelog
   - | ALLOW update_dependency
   - | ALLOW write_summary
  R4 | BLOCK END
   - | cannot finish: some rule is not satisfied yet
finished: False - R4 not satisfied
```

Two rules refuse the first update at once (no tests yet, no changelog yet); the last line is the net
refusing to *finish*, because the update was never followed by a test run. A run that finishes:

```powershell
agentmd2cpn drive out\deps_agent read_manifest run_tests read_changelog update_dependency run_tests write_summary --name one_update
```

```text
... six ALLOW lines, ALLOW END, DONE: all rules satisfied
finished: True
```

The generated net on its own — the budget rule and a forbidden action:

```powershell
python out\deps_agent\deps_agent_cpn.py read_manifest retry_install retry_install retry_install edit_lockfile report_no_updates
python out\deps_agent\deps_agent_cpn.py
```

```text
ALLOW read_manifest
ALLOW retry_install
ALLOW retry_install
BLOCK retry_install (R8)
BLOCK edit_lockfile (R6)
ALLOW report_no_updates
DONE: all rules satisfied
```

and the re-verification: 43 markings, DONE reachable True, trap states 0, dead `['edit_lockfile',
'publish_package']` — the pipeline's numbers reproduced by a file that imports only cpnpy.

---

## 3. Change 1 — a different action

Replace line 16 with:

```markdown
5. Finish by posting the update summary in the team channel.
```

and run the pipeline again (`agentmd2cpn examples\deps_agent\deps_agent.agent.md`). The hash changed,
so the reviewed spec is not reused; the model reads the file again with the nine ids offered:

```text
extract  | reviewed spec examples\deps_agent\deps_agent.spec.yaml is for another version of deps_agent.agent.md (source_hash ... != ...) - not reused
extract  | the file changed since it was last read: its 9 action ids are offered to the model so names stay the same where the actions did
extract  | R5 end: last action must be post_summary_in_channel          <- the one rule that changed
extract  | tool: ... (no line for post_summary_in_channel: the model found no declared tool for it)
verify   | state space: 43 markings, 129 arcs                            <- the net: still consistent
verify   | tools: 5 declared (read, search, fetch, execute, edit); 7 of 9 actions are tool calls, 1 need no tool; no declared tool for post_summary_in_channel
verify   | with the declared tools only, these rules cannot be satisfied: R5 (needs post_summary_in_channel)
examples\deps_agent\deps_agent.agent.md: 8 rules, 43 reachable markings, consistent=True, traps=0, dead=['edit_lockfile', 'publish_package']
```

```powershell
agentmd2cpn tools examples\deps_agent\deps_agent.agent.md
```

```text
...
- post_summary_in_channel -> NO DECLARED TOOL performs it
...
With the declared tools only (an action that needs an undeclared tool cannot happen) these rules cannot be satisfied:
- R5 (line 16: "Finish by posting the update summary in the team channel.") - needs post_summary_in_channel
```

If the model answered `none` for the new action (*a message*), that is a reading to correct before
judging: posting in a team channel needs a tool the agent does not have — set `tool: null` on it in
`out\deps_agent\agent_spec.yaml` and rerun with `--extractor yaml:out\deps_agent\agent_spec.yaml`.

What this shows: **two different questions.** The net answers *are the rules consistent* — yes, 43
markings, a run that satisfies all eight exists (`read_manifest -> post_summary_in_channel`). The tool
check answers *can this agent, with these tools, do it* — no: posting in a channel is something none
of read/search/fetch/execute/edit does, and R5 requires it in every run. The sentence is a fine policy
and this agent cannot obey it; the fix is in the file (a tool, or a different ending), not in the
pipeline. The monitor shows the same from the other side — a run that does everything it can:

```powershell
agentmd2cpn drive out\deps_agent read_manifest run_tests read_changelog update_dependency run_tests write_summary --name no_channel
```

```text
... ALLOW ... ALLOW write_summary
  R5 | BLOCK END
finished: False - R5 not satisfied
```

---

## 4. Change 2 — a parameter

Put line 16 back as it was, and add one line to *Safety* (after the retry line):

```markdown
- Never update a dependency to a major version.
```

Run the pipeline again. The reading gets a parameter and a ninth rule; the shape of the net changes
and its size does not:

```text
extract  | the file changed since it was last read: its 9 action ids are offered to the model ...
extract  | R9 forbid_when: update_dependency is forbidden when bump == 'major'
compile  | compiled 9 rules into 19 fragments
compile  | fused 19 fragments -> 9 places, 9 transitions, 54 arcs      <- one place more: the parameter's values
compile  | lowered to cpnpy: 11 places, 10 transitions, 89 arcs
verify   | state space: 43 markings, 129 arcs                            <- unchanged
verify   | with the declared tools only, every rule can be satisfied
examples\deps_agent\deps_agent.agent.md: 9 rules, 43 reachable markings, consistent=True, traps=0, dead=['edit_lockfile', 'publish_package']
```

Where the parameter went, stage by stage:

- `agentmd2cpn extract …`: the action `update_dependency` has `params: {bump: <type>}`, `data` has that
  type as an enum `[major, other]` (the two values the rules need to tell apart — nothing else about
  version numbers exists in the net), R9 is `forbid_when` with `where: bump == 'major'`.
- `agentmd2cpn compile …`: R9 has no automaton — it is one guard part, `not (bump == 'major')`.
- `agentmd2cpn compose …`: a place `ENV_update_dependency_bump` with the tokens `['major', 'other']`;
  the transition `update_dependency` now has the variable `bump` and that guard part ANDed into its
  guard; the blame line `update_dependency: R1, R2, R3, R4, R5, R9`.
- `verify`: still 43 markings — a parameter place is read, never consumed, so it adds no states; the
  parameter only decides which firings are allowed.
- `check`: the R9 witness — *after read_manifest -> run_tests -> read_changelog:
  update_dependency{'bump': 'major'} → BLOCKED by R9 alone*.

The monitor, with the value given on the command line:

```powershell
agentmd2cpn drive out\deps_agent read_manifest run_tests read_changelog update_dependency:bump=major update_dependency:bump=other run_tests write_summary --png --name major_bump
```

```text
   - | ALLOW read_manifest
   - | ALLOW run_tests
   - | ALLOW read_changelog
  R9 | BLOCK update_dependency{'bump': 'major'}
   - | ALLOW update_dependency{'bump': 'other'}
   - | ALLOW run_tests
   - | ALLOW write_summary
   - | ALLOW END
   - | DONE: all rules satisfied
finished: True
```

Same step, two values, two decisions, one rule named. The generated file does the same on its own
(`python out\deps_agent\deps_agent_cpn.py read_manifest run_tests read_changelog update_dependency:bump=major` →
`BLOCK update_dependency{'bump': 'major'} (R9)`), and its `BLAME` table lists R9 under `update_dependency`.
Review and lock this version too if you keep it.

---

## 5. Back to the first version

Remove the line you added. The file is byte-for-byte the first version again, its hash is the one the
reviewed spec of § 2 was stamped with, and the pipeline does not call the model:

```text
extract  | reviewed spec examples\deps_agent\deps_agent.spec.yaml matches this file (hash ...) - reused, LLM not called (9 actions, 8 rules)
```

(The lock of § 2 was made before § 4, so it still carries the eight rules; if you locked § 4's
version, that lock is for the other hash and this run says so.)

---

## 6. Is the pipeline correct now — what each stage guarantees

- **ingest** reads exactly one file: its segments with line numbers, its hash, its declared tools.
  Nothing else enters the pipeline.
- **extract** turns sentences into typed rules, three samples and a majority; every rule carries its
  verbatim quote and line, and a rule whose quote is not in the file is rejected. Tools: only the
  file's own list is offered, a name outside it is ignored, a tool needs a majority, a file without
  `tools:` is never asked. Parameters follow the same majority rule. What the model proposes is a
  draft until you review and lock it; a changed file is read again with the old names offered.
- **compile** builds one small net per rule from the rule's kind only (the automaton tables, the
  budget places, the guard parts); **compose** fuses them on the shared action names. No agent file
  is known to either — § 1's grep.
- **verify** proves the rules consistent (DONE reachable), finds trap states and dead actions, and
  checks the fused net against the AND of the single-rule nets on 200 random traces; the **tool
  check** asks the same reachability question with every unmapped action forbidden.
- **selfcheck** shows, for every rule, one step the net refuses because of that rule alone — the
  rule is not just present, it bites.
- **monitor / generated file** make the same decisions as the verifier on a given run, step by step,
  naming the rule; the generated file reproduces the verification with nothing but cpnpy.

What the pipeline still does **not** do, and says so: map a *real* tool call — the `toolName` and
`toolArgs` a hook sees — onto a transition. `execute` with `npm test` is `run_tests` and `execute`
with `npm publish` is `publish_package`; the per-action `tool` field names the tool, not the argument
condition that tells those apart. That mapping and the hook that uses it are the next step
(TOOLS_AND_HOOKS.md), and the research on how to express it belongs with the policy files from the
other teams.
