# Failure analysis

## What the models actually did

Every trial agent read the contract, joined the two exports, and computed
corrections. None of them failed on arithmetic. They failed on **how many
underlying problems there are.** | Run | Labels reported | Labels in the data |
|---|---|---|
| Probe run 1 | 7 | 4 |
| Probe run 2 | 5 | 4 |

Both runs produced corrected dates and amounts that were largely right, and both
scored zero, because the grading compares the *partition*, which orders group
together, not the names.

The seven-label run split defects by symptom. It saw records whose business date
was one day early, records one day late, records with a whole-dollar amount, and
so on, and gave each surface pattern its own name. Those are four presentations of
two mechanisms.

The five-label run got closer and then split the residual bucket in two, which was
a reasonable reading of an earlier revision of the data and is why that revision
was changed.

## Why this is the interesting failure

The model is not confused. It builds a coherent theory, verifies it against the
data, finds the theory consistent, and stops. **Its own verification passes while
its answer is wrong**, because the thing it got wrong is not checkable from
inside the theory it built.

That matches published analysis of frontier failures on this benchmark family:
verification failures account for roughly half of all failures, where the agent
makes a plausible attempt and does not correctly establish that the goal state is
satisfied before terminating. Terminal-Bench's own analysis says the models are
strong at execution and weak at reading a problem's structure and choosing a
method.

This task is built directly on that. There is no trick, no hidden file, no
adversarial environment. There is only a question whose answer cannot be confirmed
by the reasoning that produced it.

## What makes it hard for a person, and why that is different

A domain expert finds this task tractable but not trivial. The four mechanisms are
individually recognisable: a timezone boundary that moves the business date, a
cents truncation on one channel, soft-deleted orders still carrying entries, and a
residual manual-entry fault that can corrupt either field or both. The work is
recognising that they are four rather than three or seven, which requires forming
a hypothesis about mechanism and then testing it against populations rather than
against individual records.

The 65 records carrying two defects at once are what defeats symptom-based
reasoning. A record with both a shifted date and a wrong amount reads as one
unknown fault unless you already believe the two mechanisms are independent.

## What was ruled out along the way

**Numerical precision.** An earlier candidate failed a correct solution on the
sixteenth significant digit of a float. That is difficulty from arbitrary
precision, which the implementation rubric rejects by name, and it produces false
survivors: the model looks defeated when the verifier is simply wrong. Amounts
here are compared to half a cent, which is the unit the domain actually uses.

**Planted bugs in a static codebase.** Thirteen candidates were built on the
premise that finding several buried defects in unfamiliar code would be hard.
Eleven were solved. Frontier models are now very good at this, and the prediction
that they would struggle was wrong in a specific and useful way.

**Difficulty from environment complexity.** Larger environments did not correlate
with survival. The candidate that beat the models longest had four small Python
files.

## The failure mode this task does *not* have

An earlier candidate in the same series beat all three claude trials and then
failed the rubric's `difficult` criterion, because the environment's own comments
named the mechanism and a two-line change passed the tests. **Repeated model
failure is not evidence that a task is difficult.** It is evidence that one model,
at one configuration, made the same mistake repeatedly. That distinction cost two
days to learn and is the single most useful thing to come out of the exercise.
