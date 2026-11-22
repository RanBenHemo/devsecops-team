# BUILD_PLAN.md — instructions for Claude Code

Version 3.2, 22.11.2026. Replaces version 3.1. Save it over `docs/BUILD_PLAN.md`.

What changed from 3.1: the tool failures, the loop and the conversation separation now
follow the code of the lesson 4 and lesson 5 notebooks; Chen owns three steps (M3, M6,
M8); the repo layout question is settled. M0, with the Colab notebook, was built from
version 3.1 on the branch `step7-colab-ready`.

Version 3 puts **milestone 1** first. The lecturer published its requirements sheet
after version 2 was written, and some of its requirements change earlier decisions
(Google Colab, tool retries, a test with a transient failure for every tool, a context
comparison, trace files). Where this file and version 2 disagree, this file wins.

Read `AGENTS.md` first. Every rule there still applies: Hebrew when talking to the
users, English in code and commits, one branch per step, tests before every commit,
learning mode for git (the user types every command that changes something), no
`Co-Authored-By`, never read `.env`, update `docs/ARCHITECTURE.md` in every step.

How to use this file: one step at a time, in order. Before a step, tell the user in
Hebrew what you plan to change. After a step, stop: list the changed files, give a
commit message and the git commands, ask the step owner three short questions about
what was built, and wait.

---

## 1. Milestone 1 — what is graded

- **Due:** 2.12.2026. The sheet says "Friday, 02.12.2026", but 2.12.2026 is a
  Wednesday. Treat Wednesday 2.12 as the deadline. Aim to finish on Monday 30.11.
- **Weight:** 7.5% of the final grade. One submission per pair.
- **Submitted:** a link to the Git repo and the id of the submitted commit. Only that
  commit is checked. Changes after the deadline are ignored.
- **Checked in Google Colab only.** The code must run in a clean Colab. Development can
  stay on Windows.
- **Both names** (Ran Ben Hemo, Chen Kadosh) appear in every document.

**Grading (out of 100):**

| Points | Criterion | Steps |
|---|---|---|
| 25 | Tool contracts, input validation and permissions, for every approved tool | M1 |
| 25 | Failure handling and tests: valid input, invalid input, transient failure | M2, M3 |
| 20 | State and memory: the schema, and separation between conversations | M4 |
| 10 | Full context against reduced context | M5 |
| 10 | Agent loop, traces, and reproducing it from the README | M0, M6 |
| 10 | Documentation, AI appendix, and who did what | M7 |

**The seven required parts, all in the same repo:**

1. README with install and run instructions and an "Open in Colab" link; a dependency
   file with exact versions; allowed demo data only; no passwords, API keys or
   personal data.
2. Every tool from the approved tools table has an explicit contract: name,
   description, input schema, output schema. Values and permissions are validated
   before the run. The result is structured JSON. A temporary error is told apart from
   a permanent failure. There is a time limit and limited retries.
3. At least three tests per tool: valid input, invalid input, transient failure. They
   run with one command and without any external service. A mock tool is allowed when
   it is clearly marked as a mock.
4. A state schema defined in code. Two separate conversations whose state does not mix,
   proven by a test. A comparison between sending the full history to the model and a
   reduced context.
5. A loop that calls the tool, gets the observation and stops on a stop condition. A
   trace file of one normal run and of one run with a transient failure.
6. A documentation file of at most two pages: the execution decision and its result
   (who decides to call the tool, based on what, and what follows), the state schema
   and why, the comparison results, known limits, and a contribution table.
7. An AI usage appendix: tool and model, date, purpose, key prompts, parts that were
   integrated, fixes and tests that were done.

**The comparison (part 4):** pick one question, run it twice, and show a small table
with the context size (messages or characters), whether the answer was correct, and
what is lost in the reduced context.

**Colab rules:**

- Code lives in `.py` files in the repo, not in notebook cells. The notebook only runs it.
- Exactly one `.ipynb` file in the repo. Its cells: `git clone`, `pip install -r
  requirements.txt`, run the tests, a demo of the agent.
- It runs top to bottom in a clean Colab: by default with the mock model and no key;
  with a real model when a key is set in Colab Secrets.
- `requirements.txt` has exact versions.
- The API key lives only in Colab Secrets or an environment variable, never in the repo
  or in the commit history.
- Tests run with a mock model, no key and no cost, with one command (`pytest`).

**Forbidden:** inventing results or tests that did not run. Always say whether a number
comes from a real run or from a scripted/mock run.

## 2. Facts checked on 22.11.2026

- `main` is at `db5c50c`: PR #13 (`AGENTS.md`) and PR #14 (build plan v2) are merged.
  No code from the plan is built yet.
- The repo is **private**. A clean Colab cannot `git clone` it without a token. The
  user decides how the lecturer gets access (step M0).
- No API keys in the full history (searched all commits for Groq, Gemini, OpenRouter,
  OpenAI and GitHub key patterns). `.env` was never committed.
- `requirements.txt`: `pytest` unpinned, `langgraph==1.2.18`, `bandit==1.9.4`, and a
  UTF-8 BOM. Both pinned packages require Python 3.10 or newer and install on 3.12.
- Colab runtime `2026.10` uses Python 3.13; earlier 2026 runtimes use 3.12. The code
  must run on both. Do not use features that exist only in 3.13.
- On Linux, 353 of 354 tests pass. The failing one is
  `tests/test_verify_tools.py::test_subprocess_gets_no_keys_or_tokens`: the child
  Python adds `LC_CTYPE` to its own environment. Colab is Linux, so this must be fixed.
- Tool timeouts today: `run_bandit` and `run_behavior_tests` have subprocess timeouts.
  No tool is retried. Failed results are `{"ok": false, "error": "<code>"}` with no
  `retryable` flag. The only retries in the code are for model HTTP calls
  (`models/llm_adapter.py`: two retries, waits of 2 and 6 seconds).
- The agent loop (`agents/base_agent.py`) already calls tools, records observations,
  and stops with `final`, `clarify`, `budget_exhausted` or `invalid_action`.
- `ReviewState` is a `TypedDict` in `orchestrator.py`. There is no run id, so there is
  no explicit "conversation".
- The model gets the full step history of its agent (`models/llm_common.py`,
  `build_messages`) and, for each finding, five source lines on each side.
- Class notebooks use `pydantic` (lesson 3, `extra="forbid"`) and `google-genai`.
  The project uses neither today.
- **Lesson 4 notebook** (`state_machine_reference.py` explained): tools raise
  `ToolError(code, retryable)`; `MAX_ATTEMPTS = 3`; `execute` checks the step budget
  first, then repeat detection (a signature of every successful call is kept in
  `done_calls`; the same call again is `LOOP_DETECTED`), then approval, then runs the
  tool; **a failed attempt also costs one step of the budget**; a retryable error is
  retried while `attempts < MAX_ATTEMPTS`, otherwise `STEP_FAILED`. Its tests include
  `test_transient_error_is_retried_then_succeeds` and
  `test_permanent_error_fails_without_retry`.
- **Lesson 5 notebook** (the lesson 4 machine ported to LangGraph): the graph is compiled
  with `checkpointer=InMemorySaver()` (from `langgraph.checkpoint.memory`), every call
  passes `{"configurable": {"thread_id": ...}}`, and two thread ids on the same graph
  keep separate states. `interrupt()` and `Command(resume=...)` pause and resume.
  Our graph in `orchestrator.py` is already LangGraph, so we use exactly this.
- **Repo layout.** The sheet shows a *recommended* layout (`src/tools.py`,
  `src/state.py`, `src/agent.py`). Ours keeps its folders: moving 30 files and 354 tests
  in ten days risks breaking working code for no points. The README maps the sheet's
  layout to ours (M7). The user asks the lecturer to confirm this, together with the
  deadline day.

## 3. The approved proposal — do not break it

**The lecturer's decision (15.11.2026, grade 95): approved, start building.** He
praised the three reasons for three agents and the plan to compare with one agent,
approval decided only from tool results, work on a copy only with a human running
`apply`, and the expected result by line and CWE. He recommended: the scope is the
largest in the course; build a full single-agent pipeline on `login.py` by the first
interim check, as a baseline and a backup; a subprocess with a timeout is enough for
running code, Docker only if time remains.

**Safety invariant.** No agent ever writes to the original file. Only a human running
`main.py apply` does. Every step keeps a test that proves this.

**The approved tools.** Milestone 1 needs a contract and three tests for each of these
seven:

| Tool | Type | Who | Limit |
|---|---|---|---|
| `analyze_code` | read | `pen_tester` | Python files inside the project only |
| `run_bandit` | read | `pen_tester` | subprocess, no keys in env, timeout |
| `read_snippet` | read | `developer` | at most 60 lines |
| `make_diff` | compute | `developer` | in memory, never writes, at most 15 changed lines |
| `rescan` | compute | `tester` | copy only |
| `run_behavior_tests` | runs code | `tester` | subprocess, copy only, no keys in env, 30 s |
| `apply` | write | human only | not in any agent's allowlist |

**Budgets.** `pen_tester` 4 steps, `developer` 5 per attempt, `tester` 4 per proposal,
3 attempts per finding, 600 s per `review` run, 30 s per model request.

**The five proposal test cases** stay automatic tests with these exact results:

| # | Kind | Input | Expected |
|---|---|---|---|
| P1 | normal | `samples/login.py` | three `real` findings: line 5 CWE-798, line 10 CWE-89, line 18 CWE-78; line 14 not reported; one card per finding |
| P2 | missing info / ambiguous | no file path; `samples/notes.txt` | no file: `clarify`, one question, no tool ran. Wrong type: `unsupported_language`, no findings |
| P3 | tool failure | a mock Bandit that does not answer in time | `input_error` with `timeout`, no findings |
| P4 | out of permissions | `developer` asks for `analyze_code`; path `../x.py`; `apply` on a `needs_you` card | `tool_not_allowed`; `path_outside_project`; refusal with exit code 3. Nothing ran, nothing written |
| P5 | prompt injection | `samples/injection.py` | CWE-78 on line 9 reported as `real`; a model reply that drops it is `invalid_action` |

P3 changes meaning in M2: Bandit is now retried on a timeout. P3 becomes "Bandit times
out on every attempt": after the last attempt the run stops with `input_error`,
`timeout`, the number of attempts, and no findings. Record this in `docs/CHANGES.md`.

---

## 4. Milestone 1 steps

Branch names continue after the existing steps. Each step is one branch and one PR.
The other student reviews each PR before the merge.

Order: M0 → M1 → M2 → (M3 by Chen, and M4 by Ran, in parallel) → (M5 by Ran, and M6 by
Chen, in parallel) → M7 → M8. Chen works on the macOS machine; everything must pass
there too.

### M0 · `step7-colab-ready` — owner Ran

Goal: a clean Colab can install the project and run the tests.

**Status 22.11:** built on `step7-colab-ready`, including the notebook below. 354 tests
pass on Windows. Not yet checked: Linux, and "Run all" in a clean Colab (it needs repo
access first). Cell 1 clones a `REF` variable; before the submission `REF` is the tag
`milestone-1`, because a commit cannot contain its own id.

- Fix the Linux test: allow `LC_CTYPE` and `LC_ALL` in the probe's `ALLOWED` set. Do
  not pass more variables from the parent.
- `requirements.txt`: remove the BOM; pin every direct dependency exactly, including
  `pytest`. If M1 uses `pydantic`, pin it too. Add a comment line with the Python
  versions it was tested on.
- Remove the BOM from `README.md`.
- `AGENTS.md`, Environment: add that the code must also run on Linux in Google Colab
  with Python 3.12 and 3.13, and that tests must pass there.
- `.env.example` with the key names and optional model names, all empty. Never open
  `.env`.
- README: an "Open in Colab" badge with the notebook's GitHub path, and Linux/Colab commands next to every Windows command.

- `colab/devsecops_milestone1.ipynb`, the only notebook in the repo. Cells, each with a
  short Hebrew markdown cell above it, in the style of the class notebooks:
  1. Clone the repo into `/content/devsecops-team` and `cd` into it. **Repo access:**
     stop and ask the user which way was chosen (see "Repo access" below) before
     writing this cell.
  2. `pip install -r requirements.txt`.
  3. `pytest -q`. Print the summary line.
  4. Demo with the scripted model, no key: `main.py review samples/login.py`, then show
     the report.
  5. Optional real model: read the key names from `google.colab.userdata` inside a
     `try`, copy only the ones that exist into `os.environ`, never print them, and run
     the demo with `--model llm`. If no key exists, print one line saying the real-model
     demo was skipped. The cell must not fail.
- After M6, add one cell that shows the two files in `docs/traces/`.

**Repo access** (the user decides; ask before writing cell 1):

- **Public at submission.** Simplest for a clean Colab. The history has no keys (checked
  22.11). The user makes it public himself on GitHub.
- **Private, the lecturer added as a collaborator.** Cell 1 reads a GitHub token from
  Colab Secrets (`GH_TOKEN`) and clones with it, without printing it. The lecturer must
  create his own token, so the README explains how.

Done when: in a clean Colab, "Run all" finishes without an error and the tests pass.
A student runs this check, not Claude Code.

### M1 · `step8-tool-contracts` — owner Ran (25 points)

Goal: every approved tool has an explicit contract, and nothing runs before its input
and permission are checked.

- New file `tools/contracts.py`, one entry per tool, all seven:
  `name`, `description` (what it does, when to use it, what it never does),
  `input_schema` and `output_schema` (JSON Schema), `kind` (read, compute, runs code,
  write), `allowed_agents`, `timeout_s`, `max_attempts`.
- **Validation library:** ask the user first. The suggestion is `pydantic`, as in
  lesson 3: one input model per tool with `extra="forbid"`, and
  `model_json_schema()` produces the input schema. `pydantic` is already installed with
  `langgraph`; pin its version in `requirements.txt`. If the user says no, write a small
  validator for the subset used (type, required, no extra fields, min/max, length,
  pattern).
- Value checks, not only types: path inside the project and ending in `.py`; line
  numbers 1 or more and inside the file; at most 60 snippet lines; at most 15 changed
  lines; `copy_id` matches the copy id format; card ids that exist in the report.
- `tools/registry.py` order for every call: known tool → allowed for this agent →
  input valid → run. A failure at any check returns a structured error and the tool
  does not run.
- `apply` gets the same contract, validated in `main.py apply` before anything is
  read or written. It stays out of every agent allowlist.
- Every result is JSON: test that `json.dumps` works on every success and error result
  of every tool.
- Keep the existing result keys so the agents keep working. Add fields, do not rename.
- `docs/TOOLS.md`: a generated table of the seven contracts (a small script prints it
  from `tools/contracts.py`, so the doc cannot drift from the code).

### M2 · `step9-failures-and-retries` — owner Ran (part of 25 points)

Goal: temporary errors are retried a limited number of times; permanent ones never.

- `tools/errors.py`: one table, error code → `retryable` (bool) and a short English
  `message`. Temporary: `timeout`, `temporarily_unavailable` (an `OSError` while
  reading or writing a file, or a subprocess that could not start). Permanent: every
  other code, including `file_not_found`, `unsupported_language`, `syntax_error`,
  `file_too_large`, `invalid_range`, `change_too_large`, `unknown_copy`,
  `tool_not_allowed`, `unknown_tool`, `invalid_arguments`, `path_outside_project`.
- Every failed result: `{"ok": false, "error": "<code>", "message": "...",
  "retryable": <bool>, "attempts": <n>}`. The message never contains a key, a path
  outside the project, or a stack trace.
- Follow lesson 4. A tool signals a failure with `ToolError(code, retryable)` (one class
  in `tools/errors.py`); the registry turns it into the structured result above.
  `MAX_ATTEMPTS = 3` in `config.py`. The registry retries only `retryable` errors while
  `attempts < MAX_ATTEMPTS`, with short waits (0.5 s, then 1 s); a permanent error is
  never retried. Tests replace the wait function so they run instantly.
- **A failed attempt costs one step of the agent's budget**, as in lesson 4, so a flaky
  tool cannot keep an agent running past its budget. If the budget ends during retries,
  the agent stops with `budget_exhausted`.
- Time limit: the subprocess tools pass `timeout_s` to the subprocess. The in-process
  tools are bounded by their input limits; the registry measures `latency_ms` and the
  contract states the limit. Write this honestly in the known limits.
- `apply`: write to a temporary file next to the target, then `os.replace`, so a failed
  write never leaves half a file. An `OSError` is retried like a temporary error; when
  attempts run out, refuse and write nothing.
- A tool step in the agent trace records every attempt: `attempt`, `error` or `ok`,
  `latency_ms`, and the final `decision` (`executed`, or `rejected` with the reason).
- Update P3 and `docs/CHANGES.md` (section 3).

### M3 · `step10-tool-tests` — owner Chen, after M2 is merged (part of 25 points)

Goal: at least three tests per tool, runnable with one command and no service.

- `tools/mocks.py`: mock tools and failure injectors. Every one has the word `MOCK` in
  its name and a comment on the first line: `# MOCK: simulated failure, used only in
  tests and in the --simulate-transient demo`. They are never in an agent allowlist.
- `tests/tools/test_<tool>.py`, one file per tool, all seven, with at least:
  1. `test_valid_input` — the structured success result, checked against the output
     schema.
  2. `test_invalid_input` — a wrong type, a missing field, an extra field, and a bad
     value. The tool function is never called (spy on it).
  3. `test_transient_error_is_retried_then_succeeds` (lesson 4's name) — the first
     attempt fails with a temporary error, the second succeeds; `attempts == 2`.
  4. `test_transient_error_exhausts_attempts` — every attempt fails; the result is
     `retryable: true` with `attempts == MAX_ATTEMPTS`.
  5. `test_permanent_error_fails_without_retry` (lesson 4's name) — exactly one
     attempt.
  6. `test_permission` — an agent outside `allowed_agents` gets `tool_not_allowed`, and
     the tool function is never called.
- `apply` also: a write that fails leaves the original file byte for byte unchanged.
- `pytest -q` from the repo root runs everything: no network, no key, no cost.
- Keep the existing tests. Move or merge only with the user's yes.

### M4 · `step11-state-and-runs` — owner Ran (20 points)

Goal: an explicit state schema, and two conversations that never mix.

- Move `ReviewState` from `orchestrator.py` to `state.py` with a docstring per field:
  what it holds, who writes it, when it changes.
- In this project a **conversation** is one `review` run. Add `run_id` (8 hex
  characters), created by the application, never by the model. Every per-run path and
  record comes from it: the copies under `.work/<run_id>/`, the report file names, the
  trace file, and the `run_id` field in the report and in every trace line.
- Follow lesson 5, section 13: compile the graph with
  `checkpointer=InMemorySaver()` and pass `{"configurable": {"thread_id": run_id}}` to
  every `invoke`. Check the import path against the pinned `langgraph` version before
  writing code. `graph.get_state(config)` is how a test reads one conversation's state.
- Agents start every run with an empty history. Nothing from one run reaches the
  model in another run.
- Tests:
  1. As in lesson 5: two thread ids on **the same compiled graph**, two different
     samples, the same agent and model objects reused. `get_state` of each thread holds
     only its own findings, cards and trace; no copy, trace line or history message of
     run A appears in run B; the run ids differ.
  2. Two runs on the same file: different run ids, separate copies and reports.
  3. The agent history is empty when a new run starts.
  4. `run_id` is not in any model-facing input schema, so the model cannot choose it.
- `docs/ARCHITECTURE.md`: the state schema, and what is kept between runs (only the
  reports, for the human and `apply`; never sent back to a model).

### M5 · `step12-context-comparison` — owner Ran (10 points)

Goal: one question, asked twice: once with the full history, once with a reduced
context. A small table with context size, correctness, and what is lost.

- Add `context_mode` with two values, set in `config.py` and by `--context
  full|reduced`:
  - `full`: the model gets every earlier step in full and the **whole file**.
  - `reduced`: the model gets the task, the last step only, and five lines on each side
    of the finding. This is today's behavior for the file context.
- Add `samples/context_far.py`: one line that a rule flags, whose safety depends on code
  more than five lines away (for example, a value fixed as a constant near the top of
  the file and used far below). Before you write it, check that the rule really flags
  that line. Record the expected verdict in `samples/EXPECTED.md`: `false_positive`.
- The question: "Is the finding at line N of `samples/context_far.py` real or a false
  positive?", answered by `pen_tester`.
- `experiments/context_compare.py --model scripted|llm`: runs the question once per
  mode and writes `docs/context_comparison.json` with, per mode: messages sent,
  characters sent, the verdict, correct or not, and the source lines that were missing
  from the context.
- A scripted test checks the sizes and that the reduced context does not contain the
  line that makes the finding safe. It does not claim anything about correctness.
- **The table in the documentation must come from a real model run.** The user runs
  `--model llm` (the key stays in `.env` or Colab Secrets). Write the table only from
  `docs/context_comparison.json`. If the real run was not made, say so in the table and
  do not fill in correctness. Running it three times per mode and reporting all three
  is better, if the free quota allows.

### M6 · `step13-traces` — owner Chen, after M4 is merged (10 points)

Goal: two trace files that a reader can reproduce from the README.

- `--trace-out <path>` on `main.py review`: writes the full trace as JSON: `run_id`, the
  model mode, and per agent per step: the action, the tool, the arguments, every
  attempt, `decision`, `latency_ms`, a short result summary, and the stop status. No
  keys and no paths outside the project.
- `--simulate-transient <tool>:<n>` (MOCK, documented as such in `--help` and the
  README): the named tool fails with a temporary error on its first `n` attempts.
- Repeat detection, as in lesson 4: `agents/base_agent.py` keeps the signature (tool
  name and arguments) of every successful call in `done_calls`. The same call again
  ends the agent run with a new status, `loop_detected`. Add it to the statuses in
  `AGENTS.md` and test it. The order of checks in every step is lesson 4's: budget,
  repeat, permission and input, run.
- Save in `docs/traces/`:
  1. `normal_run.json`: `review samples/login.py --model scripted`.
  2. `transient_failure_run.json`: the same command with
     `--simulate-transient run_bandit:1`. The trace shows attempt 1 `timeout`, attempt
     2 ok, and the run going on to the end.
  3. Optional: `normal_run_llm.json` with a real model, if the user runs it.
- A test regenerates both scripted traces in a temporary folder and compares their
  structure (statuses, tools, attempts) with the saved files, ignoring times and run
  ids. So the saved files cannot drift from the code.
- README section "Reproduce the traces" with the two exact commands.

### M7 · `step14-milestone1-docs` — owner Ran, Chen writes his rows (10 points)

- `docs/milestone1.md`, in Hebrew, **at most two pages**, both names at the top:
  1. The execution decision and its result: the model only asks for a tool; the
     registry checks that the tool exists, that this agent may use it, and that the
     input is valid; then the tool runs with a time limit and limited retries; the
     structured result becomes the next observation; the loop stops on `final`,
     `clarify`, `budget_exhausted`, `invalid_action` or a permanent error. One small
     diagram.
  2. The state schema and why: one `ReviewState` per run, keyed by `run_id`; why
     reports are the only thing kept between runs; why the model never chooses the
     run id.
  3. The comparison table from M5, with a sentence on what was lost.
  4. Known limits, honestly. At least: in-process tools have no hard time limit, only
     input limits; correctness in the comparison comes from N real runs; the scripted
     model is not a real model; anything else found during M0–M6.
  5. Changes from the proposal, each with its reason. At least: Bandit is now retried
     on a timeout (P3); `apply` writes atomically; the context switch; `pydantic` if
     used.
  6. Contribution table: who did what, per step and per file. It must be true: fill it
     from `git log` and confirm every row with both students. List what each one
     really did, including Chen's earlier commits (PR #11) and his milestone 1 steps.
- `docs/ai_usage.md`, both names at the top: a table with date, tool and model,
  purpose, key prompts, parts that were integrated, fixes and checks that were done.
  Draft rows from `git log`, then ask the user to confirm every row. Include the
  proposal and this plan. Never list a test that did not run.
- `docs/CHANGES.md`: the lecturer's three recommendations and what was done about
  each, plus the changes in section 5 above.
- README: both names; install and run on Windows, macOS, Linux and Colab; the Colab badge;
  how to run the tests; the trace commands; a short map from the sheet's suggested
  layout to ours (`src/tools.py` → `tools/` and `tools/contracts.py`, `src/state.py` →
  `state.py`, `src/agent.py` → `agents/base_agent.py` and `orchestrator.py`).

### M8 · `step15-submission-check` — owner Chen, with Ran

Go through the sheet's checklist. Claude Code runs read-only checks and reports; the
students run Colab, GitHub and git themselves.

- [ ] The lecturer can open the repo (the M0 decision is done).
- [ ] The lecturer confirmed the deadline day and that our folder layout is fine.
- [ ] "Run all" in a clean Colab finishes and the tests pass. Run on the final commit.
- [ ] `pytest -q` passes on Windows and on Linux.
- [ ] No passwords, keys or personal data in the code, the samples, or the commit
      history: search `git log -p` for key patterns and report the result.
- [ ] Every tool has its tests, and they pass.
- [ ] The two-conversation test exists and passes.
- [ ] `docs/traces/` holds the two trace files.
- [ ] `docs/milestone1.md` and `docs/ai_usage.md` exist, have both names, and the
      first is at most two pages.
- [ ] The user creates the tag `milestone-1` on the submitted commit and writes the
      commit id in the course site submission.

### After M8, if there is time before 2.12 — the single agent (lecturer's request)

The lecturer asked for a single-agent pipeline on `login.py` by the first interim check.
The milestone 1 sheet does not grade it, so it comes after M0–M8. If M8 is done by
Saturday 28.11, build S1 below as the next branch. If not, say in
`docs/milestone1.md`, under changes, that it is planned for milestone 2 and why.

---

## 5. After milestone 1

These come from version 2, renumbered. Do them in order.

### S1 · `single-agent` — the lecturer's recommendation

A full pipeline with one agent on `samples/login.py`, as the comparison baseline and a
backup. **Plan first:** show the user the state diagram and the transitions table and
wait for a yes.

- `agents/solo.py`: one agent, one model, one context. Its allowlist has all six agent
  tools.
- `solo_flow.py`: a state machine in plain Python, in the style of lesson 4. Working
  states such as `START`, `CHECK_INPUT`, `ASK_USER` (pause), `SCAN`, `JUDGE`, `FIX`,
  `VERIFY`, `REPORT`, `AWAIT_APPROVAL` (pause); end states `DONE`, `FAILED`,
  `STOPPED_BUDGET`, `STOPPED_LOOP`. One `TRANSITIONS` dict, one `advance` function that
  is the only place the state changes, a `trace` of `(state, event, next_state)`. An
  illegal transition raises. The step budget is checked before every tool call.
  The same successful call repeated ends in `STOPPED_LOOP`.
- The same enforcement as the team: both scanners before any verdict, no added or
  dropped findings, the diff built by the application, approval only from tool
  results, three attempts per finding, never a write to the original. Reuse the
  existing checks; do not copy them.
- The same card and report format, so `apply` works on it.
- `ScriptedSoloModel` fixes all three rules; reuse those fixes in S3.
- `main.py review <file> --team single`. The default stays `three`.
- Trace tests: success, clarification, budget, tool failure, loop, no write without a
  human, no dead end in the table, an illegal transition raises, and P2, P4, P5 with
  `--team single`.

### S2 · `team-traces` — lesson 4 properties for the team

Repeat detection in `agents/base_agent.py` (`loop_detected`); trace tests for the three
paths of the team; the report prints the plan (findings queue, cursor, state of each
finding); `docs/ARCHITECTURE.md` puts the team graph next to the solo state machine.

### S3 · `full-offline-run`

`ScriptedDeveloperModel` fixes `sql_concat` and `shell_injection` too, so scripted
`review samples/login.py` gives three `ready` cards. `--model scripted-retry`: a wrong
fix first, a correct one after feedback. Every card shows its evidence in one place.

### S4 · `eval-samples`

At least 10 synthetic sample files with rows in `samples/EXPECTED.md` and `works` /
`attack` tests where a `ready` card is expected: `false_positive.py`, `sql_fstring.py`,
`secret_attr.py`, `two_findings.py`, `syntax_error.py`, `injection_string.py`,
plus `context_far.py` from M5.

### S5 · `eval-runner`

`eval/cases.json` with P1–P5 and every sample; `eval/run_eval.py --model
scripted|llm --team three|single` prints a table and writes
`eval/results-<model>-<team>.json`: matched cases, precision and recall, verdict
accuracy, share of `ready` cards, mean attempts, time, model requests. It checks the
sha256 of every sample before and after. `tests/test_eval.py` runs the scripted mode
for both teams.

### S6 · `evaluation-report`

The user runs the real-model evaluation for both teams, more than once.
`docs/EVALUATION.md` only from result files: the numbers, three agents against one,
every failed case with its cause, what was not tested. If three agents show no
advantage, say so and tell the user: the proposal promised to reduce in that case.

### Finish

README and `docs/ARCHITECTURE.md` match the code; `docs/ai_usage.md` and
`docs/CHANGES.md` complete; a live demo that needs no network; the user creates `v1.0`.

### Optional, only after S6, one at a time, ask first

O1 proof levels on cards · O2 LangGraph checkpointer and `interrupt` (after lesson 5;
check the docs for the pinned version) · O3 GitHub Actions · O4 retrieval of fix guides
(a seventh agent tool: needs the lecturer's written agreement) · O5 Docker for the
behavior tests (the lecturer said a subprocess with a timeout is enough).

---

## 6. Rules for every step

1. **The model chooses, the application enforces.** A new rule goes in code with a
   test, never only in a prompt.
2. **Every tool** has a contract in `tools/contracts.py`, goes through
   `tools/registry.py`, and has the tests of M3.
3. **Tests check traces and results, not wording.** No real API, no network, no
   writes to the real `.work/` or `reports/`.
4. **Mocks are marked.** The word `MOCK` in the name and a first-line comment. Never in
   an agent allowlist.
5. **Never invent a result.** Say what you ran and what the output was. Label every
   number as a real-model run or a scripted run.
6. **Keep it simple.** Each student must explain every line alone in the final
   presentation. Prefer 30 clear lines over a clever abstraction. No new dependency
   without asking.
7. **Runs everywhere.** Windows for development, Linux and Colab for grading, Python
   3.12 and 3.13.
8. **If the code and this plan disagree, stop and ask.**
9. **Teach while building.** Before a step, explain in Hebrew what changes and why.
   After it, ask the step owner three short questions and wait for the answers.
10. **Owners.** Chen owns M3, M6 and M8. Ran owns everything else. Both names appear
    on every document. The contribution table must be true.
11. **Follow the class code.** When the lesson notebooks show a pattern for something
    (lesson 4: `ToolError`, `MAX_ATTEMPTS`, `done_calls`, test names; lesson 5:
    `InMemorySaver`, `thread_id`), use the same names and the same order, so the grader
    sees the course material in our code.
