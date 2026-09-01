# How this task was found

Writing one task that frontier models cannot solve is a search problem. This is
the record of the search, including the parts that did not work, because the
negative results are more informative than the one survivor.

## The numbers

| | |
|---|---|
| Premises written | 237 |
| Environments built | 133 |
| Blind-tested against a frontier agent | 106 |
| Solved by the agent (dead) | ~98 |
| Survived blind testing | 6 |
| Survived scrutiny after that | 1 |

## The method

Rather than build one task carefully, build many cheaply and let an agent kill
them. Each candidate got a throwaway environment, an instruction, a verifier and a
reference solution, then a blind solve run with no hints. A candidate that the
agent solved was discarded immediately.

The economics justify it: building a probe took 4 to 9 minutes and a solve run 30
to 60. Building one properly to TB3 standard takes hours, so it is worth spending
an hour to avoid spending a day on a task that dies.

**Two pieces of infrastructure turned out to matter more than the probes.**

**An error-versus-failure guard.** A blind run that dies on a rate limit produces
no work and looks identical to a task the agent could not solve. Without a guard,
every infrastructure failure is recorded as a survivor. This caught a real
incident: `claude -p` backgrounded inside a script has no stdin, waits three
seconds, warns, and exits. **Seventy-one probe runs died that way in one night.**
The guard recorded them as errors. Without it there would have been seventy-one
fake candidates and no way to tell them from the real ones.

**A status ledger generated from disk, not maintained by hand.** The prose log went
stale twice inside 48 hours — it claimed 70 probes were untested when they had
been, and recorded a candidate as a survivor when it had failed once and passed
twice. Every status is now derived from the artifacts on each run.

## What did not work

**Numerical and precision tasks — 6 candidates, 0 survivors.** They produce false
survivors rather than hard tasks. The failure is in the verifier, not the model.

**Planted defects in a static codebase — 13 candidates, 2 apparent survivors, both
later rejected.** This family was predicted to survive and did not. Reading
unfamiliar code and finding several buried faults is now something frontier models
do well. One probe's agent found all four planted bugs and explained each.

**Security and access control — 2 candidates, both solved.** Including the
highest-ranked premise in the whole set.

**Social science and measurement — 8 candidates, all solved.**

**Resurrections.** Four probes that had been solved were rebuilt around the axes
that seemed to work — a system changing while the agent works, a component with no
readable source, a diagnostic that reports healthy because it shares the defect.
Two were rebuilt and tested. Both were solved.

## What did work, and how narrow it turned out to be

The survivors were all **state reconstruction where the observable state is not
the true state**, and specifically where the agent's own verification passes while
its answer is wrong. That is narrower than any of the hypotheses going in.

It also lines up with published work: verification failures account for roughly
47 to 60% of frontier failures on this benchmark family, and Terminal-Bench's own
analysis attributes failures to judgment rather than execution — models reach for
a generic method over the domain-correct one.

## The most useful negative result

**Complexity does not defeat comprehension.** Large environments, many files and
many interacting defects were all solved. The candidate that survived longest had
four small Python files. What survives is not complexity but a question the model
cannot check its own answer to.

## What I would do differently

**Read the accepted corpus first.** Terminal-Bench's 66 public tasks are the
worked answer to "what does an accepted task look like," and they were not
examined until day six. They show a different strategy from the one used here:
`atrx-vep-crispr`, `coq-block-bound`, `glycan-ms2-elucidation`, `fin-saccr-rwa`,
`intrastat-meldung`. **Most accepted tasks win on domain knowledge, not on
mechanism.** Median expert time estimate is 4 hours with a long tail to 60.

Searching for a mechanism that defeats a model competes against its reasoning,
which is excellent. Setting a task in a field with genuine specialist depth
competes against its knowledge, which has fixed and findable gaps. The second is
the better strategy and the corpus says so plainly.

That analysis arrived too late to redirect this search. It is recorded here
because it is the most actionable thing learned, and because a benchmark is only
as good as the contributors' understanding of what makes one.

## Candidates that got close

| Candidate | Why it failed to make it |
|---|---|
| Change notifications from lagging replicas | Beat all three claude trials and the adversarial run, then failed the rubric's `difficult` criterion: the environment's comments named the mechanism and a two-line change passed |
| Mutual exclusion on a lying network export | Difficult and beat all three trials, but the rubric demonstrated two working exploits, the second unfixable without redesigning the harness |
| Retuning a control loop under sensor latency | Failed two trials, solved on the third |
