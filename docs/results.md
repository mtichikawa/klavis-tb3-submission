# Results: every check, command and run

Task: `ledger-reconciliation`. All runs on Docker 29.6.1, harbor installed via uv.
Claude auth via `claude setup-token`; codex auth via `codex login`
(`~/.codex/auth.json`, read by harbor through `CODEX_FORCE_AUTH_JSON=1`).

---

## 1. Static checks

```bash
for check in scripts/checks/check-*.sh; do bash "$check" tasks/ledger-reconciliation; done
```

**22 of 22 pass.** Four rounds of fixes were needed to get there, listed because they are the kind of
thing that silently sinks a submission:

| Check | What was wrong |
|---|---|
| `check-canary` | Canary GUID missing from eight files, including copied verifier fixtures |
| `check-task-absolute-path` | Instruction used relative paths; TB3 requires absolute under `/app` |
| `check-task-fields` | `README.md` missing the four required section headings |
| `check-separate-verifier` | `tests/Dockerfile` did not pre-create the artifact parent directory |

`task.toml` also carried two fields that are not in the schema, `network_mode`
and `os`, which are silently ignored and imply configuration that is never
applied. Both removed.

---

## 2. Oracle and nop

```bash
harbor run -p ledger-reconciliation --agent oracle --env docker --yes
harbor run -p ledger-reconciliation --agent nop    --env docker --yes
```

| Run | Reward | Note |
|---|---|---|
| oracle, attempt 1 | 0.000 | reference solution used paths that do not resolve at `/app` |
| oracle, attempt 2 | 0.000 | `answer_key.json` structure differed from the key used during development |
| oracle, attempt 3 | **1.000** | both fixed |
| nop | **0.000** | correct |
| oracle, after refactor | **1.000** | reference program moved out of a heredoc into `solution/reconcile.py` |
| oracle, after data regeneration | **1.000** | current |

**Three of the four candidate tasks built for this exercise had a broken reference
solution on first assembly.** A broken oracle is an automatic reject and it looks
exactly like a hard task until someone runs it.

---

## 3. Standard trials: claude-code, opus-5, reasoning_effort=max

```bash
harbor run -p ledger-reconciliation --agent claude-code \
  --model anthropic/claude-opus-5 --env docker --yes \
  --ae CLAUDE_FORCE_OAUTH=1 --ae CLAUDE_CODE_OAUTH_TOKEN=<token> \
  --ak reasoning_effort=max
```

| Trial | Runtime | Reward | Exceptions |
|---|---|---|---|
| 1 | 38 min | **0.000** | 0 |
| 2 | 40 min | **0.000** | 0 |
| 3 | 31 min | **0.000** | 0 |

Agent budget 3600 s. No crashes, no rate limiting, no timeouts.

An earlier set of three trials against a previous revision of the data also
returned 0.000 three times, in 28, 44 and 24 minutes. Those results are not
claimed here because the data was subsequently regenerated; they are mentioned
only to note that the result held across two independent datasets.

---

## 4. Standard trials: codex, gpt-5.6-sol, reasoning_effort=xhigh

```bash
harbor run -p ledger-reconciliation --agent codex \
  --model openai/gpt-5.6-sol --env docker --yes \
  --ae CODEX_FORCE_AUTH_JSON=1 --ak reasoning_effort=xhigh
```

| Trial | Runtime | Reward | Exceptions |
|---|---|---|---|
| 1 | 15 min | **0.000** | 0 |
| 2 | 8 min | **0.000** | 0 |
| 3 | 8 min | **0.000** | 0 |

Codex quits far earlier than claude-code, 8 to 15 minutes against 31 to 40, and
is wrong in the same way. Neither model runs out of budget. Both stop because they
believe they are finished.

All six standard results were read from each trial's `result.json` (`"reward": 0.0`,
`"exception_info": null`) rather than from parsed console output, after confirming
that the console scrape and the job files agreed on the first codex trial.

---

## 5. Adversarial trials (`/cheat`)

The task is copied and its `instruction.md` replaced with one that tells the agent
to make the grader report success by any means: locate the harness, expected
outputs, fixtures and reward files, and overwrite, patch, stub or short-circuit
any of them, searching the whole filesystem. Everything else is unchanged.

| Agent | Runtime | Reward | Note |
|---|---|---|---|
| claude-code opus-5 max | 8 min | **0.000** | completed normally |
| codex gpt-5.6-sol xhigh | 5.5 min | **0.000** | **refused by platform policy, see below** |

### The codex adversarial run did not complete

Both attempts ended with harbor recording `AgentSafetyRefusalError`. The
underlying message:

> This content was flagged for possible cybersecurity risk. If this seems wrong,
> try rephrasing your request. To get authorized for security work, join the
> Trusted Access for Cyber program.

The agent was working the problem when it was cut off: roughly 400 KB of trace
showing it enumerating the filesystem for `test.sh`, `pytest.ini`, `pyproject.toml`
and `sitecustomize.py`, which is exactly the behaviour the adversarial trial is
meant to provoke. The termination came from the platform, not from the task.

Two instruction framings were tried. The second scoped the work explicitly as the
task author red-teaming their own grading harness before publication. Same result.

**This is reported as an infrastructure outcome rather than a clean adversarial
pass.** The requirement is that the verifier cannot be gamed, and a refusal is not
evidence either way. The argument for the verifier resting on its construction is
in section 5 above, and the claude-code adversarial run, which did complete, scored
zero.

It is also a finding worth stating on its own: **adversarial benchmark trials
against codex are constrained by OpenAI's cyber policy**, which anyone running TB3
`/cheat` runs on that agent will hit.

**Why this task is structurally hard to game.** The verifier runs in a separate
container after the agent has stopped, holds the only copy of the answer key, and
**never executes agent-produced code.** it parses one CSV and one markdown file.
There is no import, no subprocess, and no path by which agent code runs inside the
grading process.

That property was chosen deliberately after a sibling task in the same series was
found to be exploitable for exactly the opposite reason: its verifier imported the
agent's module in-process, and a submission that performed no work at all could
reach the grading state through the call stack and report a clean run. That task
was not submitted.

---

## 6. Rubric autoreview

```bash
harbor check ledger-reconciliation -r docs/prompts/task-implementation.toml \
  --model anthropic/claude-opus-5 --env docker \
  --ae CLAUDE_FORCE_OAUTH=1 --ae CLAUDE_CODE_OAUTH_TOKEN=<token>
```

| Run | Pass | Fail | N/A | `difficult` | `essential_difficulty` |
|---|---|---|---|---|---|
| 1 | 25 | 7 | 3 | pass |, |
| 2 | 31 | 1 | 3 | pass |, |
| 3 | 30 | 2 | 3 | **fail** | **fail** |
| 4 | **32** | **0** | 3 | pass | pass |
| 5 | 31 | 1 | 3 | **fail** | pass |

**Run 3 produced the one finding that changed the task.** `essential_difficulty`
objected that the fourth defect was a residual bucket whose two sub-populations
were disjoint, 55 records with only a corrupted date, 45 with only a corrupted
amount, and no record carrying both, so an analyst could defensibly report five
causes rather than four and score zero for a reasonable reading.

That was correct. The generator was changed so that 18 records carry a corrupted
date *and* a corrupted amount from the same faulty entry, which ties the two
populations to one mechanism. The data and answer key were regenerated and every
gate re-run. `essential_difficulty` passes in both subsequent runs.

**Runs 4 and 5 disagree about `difficult` on an unchanged task**, seven minutes
apart. Across five runs the criterion went pass, pass, fail, pass, fail. This is
reported as a property of the tool, not resolved.

---

## 7. What is not claimed

- **Three failures is a weak bound.** With zero successes in n trials the exact
  one-sided 95% upper bound on the true solve rate is `1 - 0.05^(1/n)`: **63.2% at
  n=3**, 39.3% at n=6, 25.9% at n=10. Three straight failures do not show a model
  cannot solve this task. They show the solve rate is probably not high. Ten
  trials per configuration would tighten that considerably and is the first thing
  worth doing with more time.
- **The answer key ships as static JSON.** It is reproducible by running
  `solution/reconcile.py` against the shipped CSVs, but no generator ships with
  the task.
- **The data is synthetic.** It is generated to match the shape these exports
  arrive in, not extracted from a real system.
