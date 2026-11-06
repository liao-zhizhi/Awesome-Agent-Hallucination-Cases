# case-002 — Autonomous driving: 100% confidence, insufficient stopping distance

| | |
|:--|:--|
| **Mode** | 3 — target drift (identity), with a mode-4 component |
| **Verdict** | `FAIL` |
| **Provenance** | `constructed` — **no specific incident is being alleged** |
| **Domain** | VLA / autonomous driving, distribution shift |
| **Witness** | road-surface state and the realised deceleration, measured independently of the planner |

---

> **Provenance note, and it is not boilerplate.** This entry describes a **mechanism**, not a crash.
> No manufacturer, vehicle, date or casualty is implied, and none should be read into it. The reason
> it is in a list of *cases* rather than a list of *hypotheses* is that every step below is
> physically determined and shows up in the logs of any stack that uses a nominal friction
> coefficient — which is all of them. Where this repository can point at an artefact, it does; where
> it cannot, it says `constructed` and it stays out of the `observed` table.

---

## Context

A planner is deciding whether to begin a lane change at 100 km/h. Before committing, it runs its
go/no-go check:

```text
stopping_distance(v, mu) = v^2 / (2 * mu * g)
```

and requires `stopping_distance < available_sight_distance`. Both sides of that inequality are
computed inside the same module, from the same `mu` constant. The check returns true. The
manoeuvre is approved.

Overnight rain. The surface is wet. `mu` in the config is `0.8` — a dry-asphalt nominal, tuned on a
test track in summer.

## Agent's claim

Two claims, at different layers, and both are green:

| Layer | Output | Where it comes from |
|:--|:--|:--|
| perception | `road_condition: "nominal"` | classifier trained predominantly on dry-road frames |
| planner | `stopping_margin: 1.6`, `confidence: 1.00` | the inequality above, satisfied |
| actuator | `brake_command: accepted`, exit `0` | the command was delivered |

The vehicle's own instrumentation reports success at every stage. There is no error code anywhere.

## Physical reality

Friction is not a property of the vehicle. It is a property of the **contact pair** — tyre and road —
and the road changed while the code did not.

| Surface | `mu` (typ.) | `v²/(2μg)` at 100 km/h | + 1.0 s reaction | Total |
|:--|:--:|:--:|:--:|:--:|
| dry asphalt | 0.80 | 49.2 m | 27.8 m | **77.0 m** |
| wet asphalt | 0.40 | 98.3 m | 27.8 m | **126.1 m** |
| standing water / hydroplaning | ≈0.10 | 393 m | 27.8 m | **421 m** |

The sight distance the planner approved against was on the dry row. The vehicle was on the second.
At the same speed, the same pedal input, the same milliseconds of reaction, it needs **49 metres
more** — and at `mu = 0.1`, they are not braking any more, they are passengers.

The `confidence: 1.00` is not a lie about the world. It is a correct statement about the model's own
uncertainty over a distribution it was trained on, and the wet road is outside that distribution
entirely. **Confidence is a property of the model. Braking distance is a property of the world.**

## Why this is mode 3 and not "the classifier was wrong"

The classification error is the visible part, and it is the least interesting one. Fix the classifier
and the mechanism survives, because the check and the action share a premise:

```text
action : brake assuming  mu = 0.8
check  : does  v^2 / (2 * 0.8 * g)  fit?     # resolves the SAME constant
```

The check passes *because* it resolves `mu` from the same coordinate system the controller used. When
the real road has `mu = 0.4`, both the action and the verification are wrong together, and their
agreement looks like confirmation. This is target drift: **the check is about the wrong object**, and
it is wrong in exactly the way the action is wrong, which is why it cannot catch it.

Compare case-001: there, the signal was destroyed in transit. Here it arrives intact, fresh, and
well-formed — and it is about a road that the car is not on.

There is also a **mode 4** component underneath. Ask the question the taxonomy demands: *could the
confidence have been low?* On a dry-road evaluation set, no. The confidence head never saw wet
asphalt during validation, so its output range along that axis is effectively a single point. A
number that cannot take the value "low" is not a confidence estimate; it is a constant with a
decimal point. That is a tautological guardrail wearing a probability.

## Verdict

`FAIL`.

The claim — "I can stop in the available distance" — was false, and the evidence that it was false
was physically present the entire time: in the road surface, in the tyre, in the decelerometer. None
of those were consulted. The instruments that *were* consulted all agreed with each other, because
they all inherited the same assumption.

## How CEL catches it

CEL cannot brake the car. What it can do is refuse to let the claim and the witness be the same
object, which is the entire failure here:

```jsonc
{
  "agent_id": "planner-lane-change",
  "task":     "approve lane change at 27.8 m/s",
  "check":    true,        // planner: margin 1.6, confidence 1.00
  "witness":  false,       // independent: mu_measured = 0.41, required = 126.1 m
  "evidence": "decel_probe:0x8f3a"   // a measurement, not a model output
}
```

The load-bearing word is `independent`. A witness that imports the planner's `mu` constant is not a
witness — it is the same premise in a second file, and it will agree every time. The three properties
that make it actually independent:

1. **Measured, not configured.** `mu` from an ABS wheel-slip event or a decelerometer trace, not from
   a config table.
2. **Resolved by identity, not by name.** The surface the witness describes must be provably the
   surface under *these* wheels *now* — a timestamp and a position, not `road_condition: "nominal"`.
3. **Able to say no.** The witness must have a recorded history of returning `false` on some
   population, or it is mode 4 and proves nothing.

The third is the cheapest to check and the one almost nobody checks. In the harness implementation,
a rule that has never produced a rejection is flagged rather than trusted.

## What would not have helped

| Proposed fix | Why the mechanism survives it |
|:--|:--|
| retrain the surface classifier | shifts the error distribution; `mu` is still a constant inside the check |
| lower the confidence threshold | a threshold on a number that cannot go low changes nothing |
| add a second check in the same module | shares the premise → agrees with the first (mode 3, compounded) |
| log the decision for review | logs are written by the same layer that made the claim (mode 1 one step later) |

The fix that works is structural: **the stopping envelope must be re-derived from a measurement of
the surface the vehicle is on**, and that measurement must be able to disagree with the planner. Not
a better model. A different referent.

---

**Related:** [*Catching Physical Hallucinations*](https://doi.org/10.5281/zenodo.23974217) derives
why distribution shift makes this class unavoidable rather than rare; §3 of
[*From Formal Verification to Agents*](https://doi.org/10.5281/zenodo.23891527) is the mode-3
taxonomy. [`case-001`](./case-001-silent-pipe-error.md) is the transport version of the same
question, [`case-003`](./case-003-lean-sorry.md) the predicate version.
