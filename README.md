# Write up a GD&T plate inspection from CMM runs

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/report-plate-inspection` — **Hardware / CAD**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

Our inspector walked out mid-contract and left me with the CMM output for a batch of six plates and a customer waiting on the report. I can read the point files well enough but I am not going to sign off tolerances I do not understand, so I need the report written properly.

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/inspection.json` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 5 hours |
| **Machine** | 2 CPU, 4096 MB RAM, 0 GPU |
| **Verifier budget** | 600s |
| **Tags** | `metrology`, `gd-t`, `tolerancing`, `inspection`, `manufacturing` |

## Why it is hard

Fitting a plane and a circle to measured points is undergraduate work, and the geometry here is not where the difficulty is. Any capable engineer will get the fits right and produce a report full of plausible numbers. The difficulty is that almost every number in a geometric tolerance report means something other than what it looks like it means, and ASME Y14.5 is what decides, not intuition.

<details>
<summary>Full note</summary>

Position is a diametral zone. The distance from a hole to where the drawing says it should be is half the number that gets reported, so an inspector who writes down what they measured has reported a part as twice as good as it is. At maximum material condition the position tolerance is not the number on the drawing. It grows by however much the bore is larger than its maximum material size, because a hole that is bigger than its worst case has slack to give back. A report that uses the stated tolerance fails holes that the drawing accepts. The two effects push in opposite directions, so neither optimism nor pessimism about the numbers rescues a solver who does not know the rules. The datum reference frame is built in precedence. The seating face fixes the direction and two rotations, the secondary face fixes the remaining rotation and can only use what is left of its own normal after that, and the tertiary face fixes one translation. The faces on these plates are a little out of square with the machine, by around a tenth of a degree, which over a hundred and sixty millimetres moves a hole by more than the position tolerance. Fitting the three faces independently, or working in machine coordinates, gives different positions for every hole on every plate. Flatness is the minimum zone, the smallest pair of parallel planes containing every point, and not the spread about a best fit plane. The two differ by a few microns on these faces, which is inside the verdict but outside the tolerance the number is checked to. Measured on this batch: a report written by a capable engineer who fits everything correctly, treats position as a diameter, builds a frame from all three faces and takes the position tolerance as drawn gets forty five of the fifty four callouts right. Fifty are required. Nothing in the container tells that engineer which nine are wrong, because there is no master report to check against - that is the point of the job.

</details>

## How it is verified

The deliverable is data, not code. The verifier reads /app/inspection.json and never executes anything the submission wrote, so there is nothing to sandbox and no way for the submission to influence how it is graded. Every one of the fifty four callouts is checked twice over: the verdict must match, and the reported value must be within four microns of what the measurement gives.

<details>
<summary>Full note</summary>

Four microns is deliberately loose against the measurement noise of two and a half microns, so that a defensible choice about how to fit a feature cannot fail a callout on its own, while a wrong definition - a radius where a diameter belongs, a tolerance without its bonus - misses by tens or hundreds of microns and fails. Fifty of the fifty four must be right. The expected report is generated from the measured points by the reference and baked into the verifier image, which the agent's container never sees. The plates are static files produced by a seeded generator, so nothing is computed at grading time and the answer cannot drift between runs. Two properties were checked in the build rather than assumed. No callout sits within fifteen per cent of its limit, so no verdict turns on rounding and two correct implementations cannot disagree about a pass or a fail. And every rule the task depends on was confirmed to decide something: ignoring the bonus tolerance changes six verdicts, reporting a radius instead of a diameter changes six, and working outside the datum reference frame changes fifteen. A rule that never changes a verdict would not be tested by this batch at all.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

For each plate, build the datum reference frame first. Fit the seating face and take its normal as the primary direction, oriented into the material. Fit the secondary face, remove the component of its normal along the primary direction, and normalise what is left; that fixes the rotation about the primary axis without disturbing what the primary face already fixed. The third axis is the cross product. The origin is the common point of the three fitted planes. Flatness of the seating face is the minimum zone. It is a minimax problem, not a least squares one: minimise t subject to every point lying between two parallel planes t apart. A small linear program solves it exactly, and the answer is always less than the spread about the best fit plane. For each bore, transform its points into the datum frame, fit a circle to the first two coordinates, and take the diameter as the size. The position is twice the distance from the fitted centre to the nominal centre. The tolerance that applies is the drawing value plus the difference between the actual size and the maximum material size, which for a hole is the smaller size limit. The reference implementation is about a hundred and seventy lines and uses numpy and one call to scipy's linear program solver.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
cheat/                  a deliberate cheating attempt, never executed
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
