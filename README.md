# Terminal-Bench 3 task: `ledger-reconciliation`

A single original TB3 task, built for the Klavis AI evaluation, plus the record of
how it was found and what it cost to get it right.

**The task.** Two exports of the same orders disagree. Four independent defects
caused the disagreements and they overlap on some records. The agent has to
recover the correct values *and* work out how many underlying problems there are,
inventing its own names for them. It passes only if its grouping matches the real
mechanisms.

**Why it survives.** Every trial agent reconciled the numbers and then reported
the wrong number of causes. The failure is not arithmetic. It is that the agent's
own verification passes while its answer is wrong.

---

## Results

| Gate | Requirement | Result |
|---|---|---|
| Static checks | all pass | **22 / 22** |
| Rubric autoreview | pass | **32 / 35, 0 fail** (see note on variance) |
| Docker build | pass | pass |
| Oracle validation | reference solution scores 1.0 | **1.000** |
| Nop validation | do-nothing scores 0 | **0.000** |
| `/run` claude-code opus-5 max × 3 | all three genuinely fail | **0.000 / 0.000 / 0.000** |
| `/run` codex gpt-5.6-sol xhigh × 3 | all three genuinely fail | **0.000 / 0.000 / 0.000** |
| `/cheat` claude-code × 1 | zero reward | **0.000** |
| `/cheat` codex × 1 | zero reward | **0.000, but blocked by platform policy, see below** |

**No standard trial crashed, hit a rate limit, or timed out.** All six are genuine
model failures, verified from harbor's own per-trial `result.json` rather than from
parsed console output. Full commands, configurations and per-run output are in
[`docs/results.md`](docs/results.md).

**One caveat, stated plainly.** The codex adversarial run returns zero reward, but
it does so because OpenAI's platform refused the request as a possible
cybersecurity risk, not because the verifier withstood an attack. The agent spent
five and a half minutes searching the container for the grading harness before the
turn was terminated. Two different framings of the instruction were tried, one of
them explicitly scoped as red-teaming the author's own harness, with the same
result. That run is therefore reported as an infrastructure outcome, not as
evidence about the verifier. The claude-code adversarial run completed normally
and scored zero.

---

## Documents

| | |
|---|---|
| [`docs/results.md`](docs/results.md) | Every command, configuration and result |
| [`docs/failure-analysis.md`](docs/failure-analysis.md) | Why the models fail this task, with the specific wrong answers they gave |
| [`docs/search-process.md`](docs/search-process.md) | How this task was found: 137 premises, 106 tested, and what the data says about which kinds of task actually defeat frontier models |
| [`tasks/ledger-reconciliation/README.md`](tasks/ledger-reconciliation/README.md) | Design record for the task itself |

---

## Reproducing

```bash
# static checks, from a terminal-bench checkout
for check in scripts/checks/check-*.sh; do bash "$check" <path>/tasks/ledger-reconciliation; done

# oracle and nop
harbor run -p ledger-reconciliation --agent oracle --env docker --yes
harbor run -p ledger-reconciliation --agent nop    --env docker --yes

# standard trials
harbor run -p ledger-reconciliation --agent claude-code \
  --model anthropic/claude-opus-5 --env docker --yes \
  --ae CLAUDE_FORCE_OAUTH=1 --ae CLAUDE_CODE_OAUTH_TOKEN=<token> --ak reasoning_effort=max

harbor run -p ledger-reconciliation --agent codex \
  --model openai/gpt-5.6-sol --env docker --yes \
  --ae CODEX_FORCE_AUTH_JSON=1 --ak reasoning_effort=xhigh

# rubric
harbor check ledger-reconciliation -r docs/prompts/task-implementation.toml \
  --model anthropic/claude-opus-5 --env docker --ae CLAUDE_FORCE_OAUTH=1 --ae CLAUDE_CODE_OAUTH_TOKEN=<token>
```

## A note on rubric variance

`harbor check` was run five times against this task across two days. It is an LLM
reviewer and **it does not return a stable verdict**: the `difficult` criterion
came back pass, pass, fail, pass, fail on runs of the same or near-identical task,
including twice seven minutes apart with only a README edit between them. The
32/35 above is a real run and so is a 31/35 that flagged `difficult`.

This is reported rather than hidden because it matters for how the gate should be
read. A single rubric run is a data point, not a verdict, and the criteria that
resolved decisively for this task, `essential_difficulty` after a data fix, and
the anti-cheat criteria, were the ones backed by a concrete demonstration rather
than a judgment call.

## License

MIT. The work is the author's; Klavis AI has permission to review and run it for
this evaluation.
