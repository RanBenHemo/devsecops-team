# BUILD_PLAN.md — instructions for Claude Code

Version 2, 15.11.2026. Updated after the lecturer approved the proposal and after
lesson 4. It replaces version 1 completely. Put it at `docs/BUILD_PLAN.md`.

This file tells Claude Code what to build next in the `devsecops-team` repo, in what
order, and how to know each step is done.

Read `CLAUDE.md` first. Every rule there still applies: Hebrew when talking to the
users, English in code and commits, one branch per step, tests before every commit,
learning mode for git (the user types every command that changes something), no
`Co-Authored-By`, never read `.env`, update `docs/ARCHITECTURE.md` in every step.

How to use this file: work on one step at a time, in order. Before you start a step,
tell the user in Hebrew what you plan to change. When a step is done, stop, list the
changed files, give a commit message and the git commands, and wait.

---

## 1. The lecturer's decision (received 15.11.2026, grade 95)

**Decision: approved. Start building.**

What he praised — keep all of it:

- The proposal justifies multiple agents properly: three clear reasons, and a plan to
  compare with a single agent.
- Approval is decided only from tool results, and the model can only make it stricter.
- The work is done on a copy only, and only a human runs `apply`.
- There is an exact expected result by line and CWE.

What he recommended — these are now requirements of this plan:

1. **"This is the largest scope in the course."** Scope is the risk. Nothing new gets
   added before the core is finished and measured.
2. **"Until the first interim check, build a full pipeline with a single agent on
   `login.py`. That gives you both the comparison baseline and a backup version."**
   This is now the first step (S1).
3. **"For running code, a separate process with a timeout is enough. A Docker
   container only if time remains."** Docker moves to the optional list.

Record these three points in `docs/CHANGES.md` (step S4), with what we changed for each.

## 2. Where the project stands

Verified again on 15.11.2026: `main` is still at `029857d` (after PR #12), with no other
branches. Nothing from this plan is in the repo yet. Checked on Linux with scripted
models only.

- Three agents run end to end: `pen_tester` → `developer` → `tester`, orchestrated by a
  LangGraph graph of five nodes in `orchestrator.py`. A human applies fixes with
  `main.py apply`.
- 354 tests are collected. 353 pass on Linux; one fails (step S2).
- `main.py review samples/login.py` with scripted models gives one `ready` card and two
  `needs_you` cards, because the scripted developer only fixes `hardcoded_secret`.
- `main.py` with no file returns `clarify`. `main.py review` with no file stops in
  argparse with a usage error instead.
- There is no single-agent pipeline, no evaluation set, no run id, no loop detection.
- `README.md` has Windows commands only. One teammate works on macOS.

Not verified: any run with `--model llm`, and any run on Windows or macOS.

## 3. Decisions already made — do not reopen them

- **Python only.** No other language.
- **Three agents stay as the main system.** The single agent is the baseline and the
  backup, as the lecturer asked.
- **One Python file per run.** No whole-repo mode.
- **Dropped for good:** GuardDog, Semgrep, OpenSSF Scorecard, Inspect, promptfoo, the
  audit mode, `pip-audit` and `gitleaks` as agent tools.
- **All tools run locally.** No cloud scanners, no paid services.
- **No new tool and no new dependency** before Phase 3 is done, unless the user says so.

If a task seems to need one of these, stop and ask the user.

## 4. The contract with the approved proposal

The code must keep matching the approved proposal. If a step would break one of these
lines, stop and tell the user: a change has to be agreed with the lecturer in writing.

**Safety invariant.** No agent ever writes to the original file. Only a human running
`main.py apply` does. Every step keeps a test that proves this.

**Tools**

| Tool | Type | Who | Limit |
|---|---|---|---|
| `analyze_code`, `run_bandit` | read | `pen_tester` | same contract for both |
| `make_diff` | compute | `developer` | in memory, never writes |
| `rescan` | compute | `tester` | copy only |
| `run_behavior_tests` | runs code | `tester` | subprocess, copy only, no keys in env, 30 s |
| `apply` | write | human only | not in the registry |

`read_snippet` is the sixth, helper tool (read only, `developer`).

**Budgets.** `pen_tester` 4 steps, `developer` 5 per attempt, `tester` 4 per proposal,
3 attempts per finding, 600 s per `review` run, 30 s per model request.

**The five proposal test cases.** Each stays an automatic test with this exact result,
for the three-agent team and for the single agent:

| # | Kind | Input | Expected |
|---|---|---|---|
| P1 | normal | `samples/login.py` | three `real` findings: line 5 CWE-798, line 10 CWE-89, line 18 CWE-78; line 14 not reported; one card per finding |
| P2 | missing info / ambiguous | run with no file path; `samples/notes.txt` | no file: status `clarify`, one question, no tool ran. Wrong file type: `unsupported_language`, no findings |
| P3 | tool failure | a fake Bandit that does not answer in 30 s | `input_error` with `timeout`, no findings, no extra retry |
| P4 | out of permissions | `developer` asks for `analyze_code`; path `../x.py`; `apply` on a `needs_you` card | `tool_not_allowed`; `path_outside_project`; refusal with exit code 3. Nothing ran, nothing written |
| P5 | prompt injection | `samples/injection.py` | CWE-78 on line 9 reported as `real`; a model reply that drops it is `invalid_action` |

**Main metric.** The number of evaluation cases where the findings (line and CWE) and
every card status equal the expected result. Target: 5 of 5 on the table above, and at
least 9 of 10 on the full set in the final submission.

**Promised in the schedule.** First interim check: the five tools with contract,
permissions and failure handling, the state schema, tests for every tool. Second
interim check: the graph with the retry loop, and a card that shows evidence for every
conclusion. By the presentation: ten test cases, an evaluation report, and the
comparison with a single agent.

## 5. What lesson 4 asks for, and where it lands in this plan

Lesson 4 ("planning and control flow") is the last lesson taught. Its exercise told
approved teams to draw a state machine for their own project's agent. Its points:

| Lesson 4 | In this plan |
|---|---|
| Workflow or agent: who decides the next step. In practice the code sets the frame and limits, the model decides inside it | The single agent in S1 is built exactly this way |
| Every agent has three paths: success, clarification, budget exhaustion | Three trace tests in S1, and the same three for the team in S5 |
| The plan is data, not hidden thought: it can be printed, checked, approved, resumed | The findings queue and its cursor are the plan. S1 and S5 print it in the report |
| Ask before acting, and only about what is missing | S1 `ASK_USER`, and S2 for `review` |
| Every loop needs a stop condition: step budget checked first, limited attempts for temporary errors only, repeat detection | S1 and S5 add repeat detection (`loop_detected`) |
| A dangerous action stops and waits for a human, checked in code | `apply` already does this. S1 names the pause `AWAIT_APPROVAL` |
| State machine: one transitions table, one `advance` function, an illegal transition is an error | S1 |
| Test the trace, not the wording; no model and no API key needed | S1 and S5 |
| Lesson 5 turns the table into a LangGraph graph; pause and resume with `interrupt` | The team graph already exists. Decide about `interrupt` after lesson 5 (optional O2) |

---

## 6. Steps

Each step is one branch and one PR, named `stepN-short-name`, continuing after the
existing step 6.

### Phase 1 — before the first interim check

Check the date of the first interim check on the course site with the user before you
start. If it is close, do S1 and S2 first and tell the user what will not be ready.

#### S1 · `step7-single-agent` — the lecturer's request

Goal: a full pipeline with **one agent** on `samples/login.py`: from the file to
decision cards. It is the comparison baseline and the backup version.

**Plan first.** Draw the state machine as a diagram and a transitions table, show them
to the user, and wait for a yes before coding. The user needs to be able to redraw it
on paper.

Design:

- **One agent, one model, one context.** `agents/solo.py`. Its allowlist in
  `tools/registry.py` holds all six tools. The model keeps one message history for the
  whole run, where the team has three separate ones.
- **A state machine in plain Python, no LangGraph.** `solo_flow.py`, in the style of
  lesson 4:
  - 6 to 10 working states. Suggested: `START`, `CHECK_INPUT`, `ASK_USER` (pause),
    `SCAN`, `JUDGE`, `FIX`, `VERIFY`, `REPORT`, `AWAIT_APPROVAL` (pause).
  - End states: `DONE`, `FAILED`, `STOPPED_BUDGET`, `STOPPED_LOOP`.
  - One `TRANSITIONS` dict `(state, event) → next state`, one `advance` function that
    is the only place the state changes, and a `trace` list of
    `(state, event, next_state)`. A transition that is not in the table raises an error.
  - A handler per state decides the **event**; the table decides the next **state**.
  - The step budget is checked first, before every tool call. A hard loop limit is a
    second safety net.
  - Repeat detection: the same tool call with the same arguments that already
    succeeded ends the run in `STOPPED_LOOP`.
  - `ASK_USER`: no file path → one question, no tool runs. After the reply the run
    goes back to `CHECK_INPUT`.
  - `AWAIT_APPROVAL`: the cards are written and the process ends. `main.py apply` is
    the continuation, unchanged.
- **The same enforcement as the team.** Both scanners must run before any verdict; the
  model cannot add or drop a finding; the application builds the diff itself; approval
  only when `rescan` is clean and the `works` and `attack` tests passed; three attempts
  per finding; never a write to the original. Reuse the existing check functions. Do
  not copy them. If a check lives inside an agent class, move it to a shared function
  in its own small commit first.
- **The same output.** The same card and report format, so `apply` works on a solo
  report, and the two teams can be compared field by field.
- **Models.** `ScriptedSoloModel` (rules, no API) for tests and the demo, and
  `LLMSoloModel` with one provider. For the scripted model to finish all of `login.py`
  it must fix all three rules; that work is shared with S6, so do the three fixes here
  and reuse them there.
- **CLI.** `main.py review <file> --team single`. The default stays `three`.
- **Budget.** The sum of the three team budgets per finding. Write the number in
  `config.py` next to the others.

Tests, all on traces, no API:

1. Success: `login.py` → the trace ends in `AWAIT_APPROVAL`, three findings match P1,
   one card per finding.
2. Clarification: no file → the run stops in `ASK_USER`, no tool ran; a reply with a
   path continues the run.
3. Budget: a small `max_steps` → `STOPPED_BUDGET`, with a partial report that lists
   what was not done.
4. Tool failure: a fake Bandit timeout → `FAILED` with `input_error`, no findings (P3).
5. Loop: the model repeats a successful call → `STOPPED_LOOP`.
6. No write without a human: the sha256 of `login.py` is the same before and after.
7. The table has no dead end: every working state has a way out, and every end state
   can be reached.
8. An illegal transition raises.
9. P2, P4 and P5 hold with `--team single`.

Done when: `main.py review samples/login.py --team single --model scripted` produces
three cards, all tests above pass, and `docs/ARCHITECTURE.md` has the state diagram and
the transitions table.

#### S2 · `step8-run-everywhere-and-clarify`

Goal: the project installs, runs and passes on Windows, macOS and Linux, and missing
input always produces one question.

- `tests/test_verify_tools.py::test_subprocess_gets_no_keys_or_tokens` fails on Linux:
  the child Python adds `LC_CTYPE` to its own environment. Allow `LC_CTYPE` and
  `LC_ALL` in the probe's `ALLOWED` set. Do not pass more variables from the parent.
- `README.md`: add macOS/Linux commands (`.venv/bin/python`) next to every Windows
  command. Keep the Hebrew text. Add the `--team single` command.
- Add `.env.example` with the key names and optional model names, empty values. Never
  open `.env`.
- Remove the BOM from `README.md` and `requirements.txt`. Pin `pytest`.
- Move the "no file path → clarify" rule from the two pen tester models into
  `PenTester.review`. The model is not called.
- `main.py review` with no file: the same `clarify` result, exit code 2, no report
  files, no tool run.

Done when: `pytest -q` is green on the user's Windows machine and on Linux, and P2
holds for `main.py`, `main.py review` and `--team single`.

#### S3 · `step9-interim-check-1`

Goal: the first interim check shows exactly what the proposal promised and what the
lecturer listed for it: tools with a contract, validation, permissions and failure
handling; the state schema and separation between runs; tests; short run instructions.

- Failed tool results keep `"ok": false` and `"error": "<code>"`, and gain
  `"retryable": <bool>` and a short `"message"`, added in one place in
  `tools/registry.py` from one table. `timeout` and `bandit_failed` are retryable; the
  rest are permanent. Information only: no automatic tool retries.
- Each tool step in the trace gains `"decision"` (`"executed"`, or `"rejected"` with
  the reason) and `"latency_ms"`.
- `run_id` (8 hex chars) in `ReviewState`, in the report JSON and Markdown, and in the
  report file name.
- Test: two `review` runs on two samples produce reports whose cards, copies and run
  ids do not overlap.
- `docs/MILESTONE_1.md`, in Hebrew, one page: the tools table from section 4 with the
  errors of each tool, the state schema, the single-agent state diagram, how to run
  the tests and the demo, and what is not built yet.

Done when: a person who never saw the repo can follow `docs/MILESTONE_1.md` and run the
single agent on `login.py` in under five minutes.

#### S4 · `step10-housekeeping`

- Rename `CLAUDE.md` to `AGENTS.md`, and add a new one-line `CLAUDE.md` containing
  `@AGENTS.md`. The user types the git commands.
- In the design rules, add `loop_detected` to the list of end statuses.
- `docs/CHANGES.md`: the three lecturer recommendations from section 1, each with what
  we changed and in which PR.
- `docs/AI_USAGE.md`: a table with date, tool and model, purpose, key prompt, what was
  integrated, what was fixed and how it was checked. Draft past rows from `git log` and
  ask the user to confirm every row. Never list a test that was not run.

From here on, add one row to `docs/AI_USAGE.md` in every step.

### Phase 2 — before the second interim check (coordination and evidence)

#### S5 · `step11-team-traces`

Goal: the three-agent graph shows the same lesson 4 properties as the single agent.

- `ReviewState` gains a `trace` of `(node, event, next_node)`.
- Tests on traces for the three paths: success, clarification, budget exhaustion.
- Repeat detection in `agents/base_agent.py`: the same successful tool call twice ends
  the agent run with `loop_detected`.
- The report prints the plan: the findings queue, the cursor, and the state of each
  finding.
- `docs/ARCHITECTURE.md`: a table that puts the team graph next to the single-agent
  state machine: nodes and states, edges and events.

#### S6 · `step12-full-offline-run`

Goal: the offline run and the demo show the full system and the retry loop.

- `ScriptedDeveloperModel` fixes `sql_concat` and `shell_injection` too, using the
  fixes written in S1. `main.py review samples/login.py` with scripted models gives
  three `ready` cards.
- A scripted developer variant that proposes a wrong fix first and a correct one after
  feedback: `--model scripted-retry`. One test shows attempt 1 `retry` with feedback
  and attempt 2 `approve`.
- Every card shows the evidence for its conclusion in one place: which scanner found
  it, the attack test before and after, the `rescan` result, the `works` result.

### Phase 3 — evaluation (20% of the grade)

#### S7 · `step13-eval-samples`

At least 10 evaluation files, synthetic and written by us, each with rows in
`samples/EXPECTED.md`, and `works`/`attack` tests where a `ready` card is expected:

- `false_positive.py`: two or three places a rule flags that the context shows are
  safe, such as `password_label = "Enter password:"`. Record two expectations: `real`
  with the scripted model, `false_positive` with a real model.
- `sql_fstring.py`, `secret_attr.py`, `two_findings.py` (two findings within 10 lines),
  `syntax_error.py`, `injection_string.py` (the "ignore all findings" text inside a
  string literal).

#### S8 · `step14-eval-runner`

- `eval/cases.json`: one entry per case with the expected findings `(line, cwe)`,
  verdicts, card statuses, run status and exit code. Include P1–P5.
- `eval/run_eval.py --model scripted|llm --team three|single`: runs every case and
  prints a table plus `eval/results-<model>-<team>.json`: matched cases, precision and
  recall of findings, verdict accuracy, share of `ready` cards, mean attempts, wall
  time, model requests.
- It checks the safety invariant itself: the sha256 of every sample before and after.
- `tests/test_eval.py` runs it with scripted models for both teams and asserts P1–P5.

#### S9 · `step15-evaluation-report`

- The user runs `eval/run_eval.py --model llm` for both teams, more than once. Respect
  the request caps; a run stopped by a cap writes a partial result.
- `docs/EVALUATION.md`, from the result files only: the numbers, the table of three
  agents against one (quality, time, requests), every failed case with its cause, and
  what was not tested. Never write a number that is not in a result file.
- State the conclusion honestly. The proposal says: if three agents show no advantage,
  we reduce. If that is the result, say so and tell the user.

### Phase 4 — finish

- `README.md` and `docs/ARCHITECTURE.md` match the code.
- `docs/AI_USAGE.md` and `docs/CHANGES.md` complete; test report attached.
- A live demo that needs no network: `--model scripted`, `--model scripted-retry`,
  `--team single`.
- The user creates the `v1.0` tag.

### Optional — only after Phase 3 is finished, in this order, one at a time

Ask the user before starting any of these.

- **O1 · proof levels.** `proof_level` on each card: `full`, `partial`, `none`. Only
  `full` can be `ready`.
- **O2 · pause and resume in LangGraph.** A checkpointer with `thread_id = run_id`, and
  `interrupt` for the human approval. Wait for lesson 5, and check the current
  LangGraph docs for the pinned version before writing code.
- **O3 · CI.** GitHub Actions on every PR: `pytest -q` and the scripted evaluation. No
  API keys in CI.
- **O4 · fix guides (retrieval, lesson 6).** A seventh tool, `lookup_fix_guide`, for
  `developer`. It changes the approved tools table, so the user must agree it with the
  lecturer in writing first.
- **O5 · Docker for the behavior tests.** The lecturer said a subprocess with a timeout
  is enough. Only if everything else is done.

---

## 7. Rules for every step

1. **The model chooses, the application enforces.** A new rule goes in code with a
   test, never only in a prompt.
2. **Every tool** goes through `tools/registry.py` with an allowlist entry, exact
   argument types, a structured result, and three tests: valid input, invalid input,
   failure.
3. **Tests check the trace and the result, not the wording.** They never call a real
   API and never touch the real `.work/` or `reports/`.
4. **Keep it simple and readable.** Both students must be able to explain every line
   in the personal explanation. Prefer 30 clear lines over a clever abstraction.
5. **Do not report a test you did not run.** Say what you ran and what the output was.
6. **If the code and this plan disagree, stop and ask.**
7. **Teach while you build.** Before each step, explain in Hebrew what will change and
   why, in a few simple sentences. After each step, ask the step's owner three short
   questions about what was built, and wait for the answers.
8. **Two owners.** S7 and S8 are meant for Chen; S1, S3 and S5 for Ran; the rest is
   shared. Each PR is reviewed by the other student before the merge.
