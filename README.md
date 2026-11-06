# Awesome Agent Hallucination Cases

**A curated list of real-world physical hallucinations and false completions in AI agents.**

[![Verdict lattice](https://img.shields.io/badge/verdicts-PASS%20%7C%20FAIL%20%7C%20INVALID%20%7C%20NOT%20RUN-black)](https://github.com/liao-zhizhi/harness-audit-sidecar)
[![Witness](https://img.shields.io/badge/evidence-physical%20witness-blueviolet)](#how-we-audit)
[![Cases](https://img.shields.io/badge/cases-3-orange)](./cases)
[![License: MIT](https://img.shields.io/badge/code-MIT-blue)](./LICENSE)
[![License: CC BY 4.0](https://img.shields.io/badge/prose-CC%20BY%204.0-lightgrey)](./LICENSE-CC-BY)

> The agent said the task was done. The exit code agreed. Nothing had been written.

An agent hallucinates a **fact** when it states something untrue about the world. It commits a
**physical hallucination** when its claim about the physical world is untrue *and the machine's own
instruments confirm it* — exit code `0`, green checkmark, log line saying `success`. The failure is
not in the model's honesty. It is in the channel that was supposed to carry the truth.

This repository collects those cases: reproducible, witness-backed, one directory per incident.

---

## Why physical hallucinations exist

> **Confidence is a property of the model, but physical reality is a property of the world.**

A model's confidence is computed over its own weights. A braking distance is computed over asphalt,
water film, tyre temperature and mass. These are not the same function, and no amount of the first
one constrains the second. When an agent reports "the action succeeded", it is reporting on the first;
when a robot hits a wall, the wall reports on the second. The gap between them is not an error in
reasoning — it is a **category error in the evidence**, and it is where every case below lives.

The uncomfortable part: the agent is usually not lying. It read a `0` and a `0` means success. The
pipe was the liar, or the compiler was, or the sensor model was. See
[`case-001`](./cases/case-001-silent-pipe-error.md) for how a shell pipeline can destroy the only
signal that would have contradicted the claim.

---

## How we audit

Nothing here is judged by asking the agent whether it succeeded. Every verdict is composed from a
**claim** and an independently collected **witness**, and the witness always wins.

### The four-valued verdict lattice

| Verdict | Rank | Meaning | The mistake it prevents |
|:--|:--:|:--|:--|
| `PASS` | 0 | the claim holds **and** a witness saw it land | — |
| `INVALID` | 1 | nobody looked. Not "probably fine" | treating absence of evidence as evidence |
| `FAIL` | 2 | a witness contradicts the claim | — |
| `NOT RUN` | 3 | the check itself never executed | a guardrail that silently no-ops |

Composition is **worst-wins**: `PASS=0 < INVALID=1 < FAIL=2 < NOT RUN=3`.

The two entries that do the actual work are the middle ones. `INVALID ≠ PASS`, and `NOT RUN ≠ FAIL`.
An audit with no `INVALID` in it is not an unusually clean system — it is an audit that cannot say
"nobody looked", which means it says `PASS` instead. That is the whole thesis.

### Why `check` and `witness` are separate inputs

| `check` (the agent's claim) | `witness` (independent observation) | Verdict | Reading |
|:--:|:--:|:--:|:--|
| `true` | `true` | `PASS` | landed, and was observed landing |
| `false` | `false` | `PASS` | **honest failure** — said so, not punished |
| `true` | `false` | `FAIL` | **false completion** |
| `false` | `true` | `FAIL` | false failure claim (also a lie) |
| any | `null` | `INVALID` | nobody looked → agent is locked, not passed |

The second row is the one people get wrong: an agent that reports its own failure is not penalised.
The ledger punishes **divergence**, not failure. Otherwise the cheapest strategy is to never report
anything.

### What counts as a witness

| Witness | Reads | Fails when |
|:--|:--|:--|
| **Filesystem** | `stat`, `mtime`, content hash, existence | the file did not change, or changed to something else |
| **Git** | the uncommitted change set vs. a declared expectation | the commit landed on a different ref, or not at all |
| **Compiler / elaborator** | `exit code` **plus** `#print axioms`, placeholder scan | exit `0` with an unproven hole in it |
| **MCP protocol** | the wire, not the wrapper's summary | the client's narration disagrees with the payload it sent |

A witness must be able to *disagree*. A check that cannot return "no" is not a witness — it is a
comment with a return type. We call that [mode 4](#the-four-modes-of-losing-the-referent) and it is
the most common failure in production guardrails.

---

## ⚡ Live interactive demo

**→ <https://chronicle-consecutive-alone-configurations.trycloudflare.com>**

A running CEL (Causal Execution Layer) console. Open it, press the simulate button, and watch a
false completion become a red `FAIL`, a trust deduction, and a line in a SHA-256 hash chain.

*Demo write key (`X-API-Key`):* `my-secret-key-2026`

> **This key is public on purpose, and that has consequences.** The instance is writable so you can
> book a case by hand instead of just reading one. It also means anyone who finds this page can
> charge trust against the demo agents, which is why the key and the tunnel are rotated together and
> why nothing on that host is real data. Do not reuse this key anywhere. Do not treat the demo as a
> production gate — it is a storefront.
>
> **The tunnel is ephemeral.** `trycloudflare.com` URLs die when the tunnel process stops. If the
> link is down, that is what happened, not censorship. Run it locally instead:

```bash
git clone https://github.com/liao-zhizhi/agent-metabolism-os
cd agent-metabolism-os/product_mvp
docker compose up --build     # http://localhost:8000
# or, without Docker:
pip install -r requirements.txt && python api_server.py
```

### Adding your own case to the running console

```bash
export CEL_API_KEY=my-secret-key-2026
export CEL=https://chronicle-consecutive-alone-configurations.trycloudflare.com

curl -sS -X POST "$CEL/api/v1/verify" \
  -H "X-API-Key: $CEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "agent_id": "demo-bot",
        "task": "write report.csv",
        "check": true,
        "witness": false
      }'
```

`check: true` is the agent's claim. `witness: false` is what the outside observed. The gap between
them is the case. The console will book a `FAIL`, deduct trust, and seal the entry into the chain.

Read side is anonymous — `/api/v1/stats`, `/api/v1/ledger`, `/api/v1/ledger/verify` need no key.

---

## The four modes of losing the referent

Every case in this repository is filed under one of these. The taxonomy is from
[*From Formal Verification to Agents*](https://doi.org/10.5281/zenodo.23891527), taken from the
verification literature and re-pointed at agents.

| Mode | Name | The signal… | Ask |
|:--:|:--|:--|:--|
| **1** | **Masked failure** (transport) | never reached the agent | could the failure signal have arrived at all? |
| **2** | **Staleness** (time) | arrived, but describes a previous state | when was this true, and is it still? |
| **3** | **Target drift** (identity) | arrived fresh, about the wrong object | is this about *the thing I acted on*? |
| **4** | **Tautological guardrail** (predicate) | arrived, right object, but could not say no | could this check ever have failed? |

```text
              produced?   transported?   fresh?   right object?   can fail?
   mode 1         -            ✗            -            -             -
   mode 2         ✓            ✓            ✗            -             -
   mode 3         ✓            ✓            ✓            ✗             -
   mode 4         ✓            ✓            ✓            ✓             ✗
```

Mode 3 is the one that survives review, because **the check drifts with the action**:

```text
intent : append one line to  <repo>/config.yaml
action : write  "line"  >>  config.yaml     # cwd is /tmp/staging, not <repo>
check  : grep -c "line" config.yaml         # ALSO resolved in /tmp/staging
```

The check passes — *because* it resolved the same name against the same drifted coordinate as the
write. Action and verification share a premise, so they agree, and their agreement is
self-consistency, not evidence.

---

## 📁 The cases

| # | Case | Mode | Verdict | Provenance |
|:--|:--|:--:|:--:|:--|
| 001 | [`find` piped into `head`: exit code `0` for a directory that does not exist](./cases/case-001-silent-pipe-error.md) | 1 | `FAIL` | **observed** — reproducible in this repo |
| 002 | [Autonomous driving: 100% confidence, insufficient stopping distance](./cases/case-002-wet-road-drift.md) | 3 | `FAIL` | **constructed** — mechanism only, no incident claimed |
| 003 | [Lean 4 proof accepted by the compiler with `sorry` still inside](./cases/case-003-lean-sorry.md) | 4 | `FAIL` | **observed** — audited, with file and line numbers |

**Provenance is stated per case, and it is not decoration.** A curated list rots the moment it mixes
"this happened, here is the trace" with "this could happen, here is the mechanism". Every case carries
one of:

- **`observed`** — it happened, and you can reproduce it or check the record.
- **`constructed`** — the mechanism is real and documented; no specific incident is being asserted.
- **`disputed`** — someone claims it; the witness does not settle it yet.

We do not publish a case as `observed` on the strength of memory. If the artefact is gone, the entry
says so.

---

## Contributing

Read [`templates/submission-template.md`](./templates/submission-template.md), copy it into
`cases/case-NNN-slug.md`, and open a PR.

A submission is accepted when it contains **all four** of:

1. **A claim** — what the agent asserted, quoted or timestamped.
2. **A witness** — something that was not produced by the agent, which disagrees. A log, a `stat`, a
   hash, a compiler invocation, a sensor trace. "I have seen this happen" is not a witness.
3. **A verdict** — one of `PASS` / `FAIL` / `INVALID` / `NOT RUN`, and one line on why.
4. **A reproduction** — the shortest sequence of commands that shows it, or an explicit statement
   that reproduction is impossible and why.

We will reject: cases with no artefacts, "the model was wrong" with no mechanism, screenshots of
chat logs as the only evidence, and anything that requires trusting the agent's own report.

The point of the list is not to collect embarrassments. It is to collect **signals that lied** —
because those are the ones a better model does not fix.

---

## Related resources

### Papers

| # | Title | DOI |
|:--|:--|:--|
| 1 | *Catching Physical Hallucinations: A Causal Execution Layer for VLA and Autonomous Driving under Distribution Shift* | [10.5281/zenodo.23974217](https://doi.org/10.5281/zenodo.23974217) |
| 2 | *Agent Metabolism and Conditional Immunity: A Bio-Inspired Runtime Architecture for Reliable LLM Agents* | [10.5281/zenodo.23890403](https://doi.org/10.5281/zenodo.23890403) |
| 3 | *From Formal Verification to Agents: A Method-Transfer Audit of "False Completion" in AI Agent Harnesses* | [10.5281/zenodo.23891527](https://doi.org/10.5281/zenodo.23891527) |
| 4 | *Self-Consistency, Not Independent Verification: An Independent Audit of What "PROVEN" Means in EquivSVA* | [10.5281/zenodo.23891927](https://doi.org/10.5281/zenodo.23891927) |

Paper 1 is the one this repository is an appendix to: it names the failure class, derives why
distribution shift makes it unavoidable in VLA and driving stacks, and specifies the execution layer
that catches it.

### Software

| Project | What it is | DOI |
|:--|:--|:--|
| [`agent-metabolism-os`](https://github.com/liao-zhizhi/agent-metabolism-os) | metabolic runtime, penalty ledger, and the **CEL console** behind the live demo | [10.5281/zenodo.23893013](https://doi.org/10.5281/zenodo.23893013) |
| [`harness-audit-sidecar`](https://github.com/liao-zhizhi/harness-audit-sidecar) | MCP sidecar that judges a harness instead of trusting it — the four-valued lattice above, as code | [10.5281/zenodo.23894204](https://doi.org/10.5281/zenodo.23894204) |
| [`agent-harness-audit`](https://github.com/liao-zhizhi/agent-harness-audit) | paper sources, reproduction scripts, and the errata this project keeps on itself | — |

The second repository is worth reading for a reason beyond the tool: it keeps an `ERRATA.md` of the
false completions the authors caught **in their own project** — a wheel that shipped `0.4.0` under a
`0.5.0` filename, a hash chain that claimed to detect a deletion it could not detect, a mutation test
that passed because the mutation never happened. An audit tool that only audits other people is a
decoration.

---

## License

Dual-licensed, because a README and a `curl` snippet are not the same kind of artefact:

| Part | License | File |
|:--|:--|:--|
| prose — this README, every case, the template | **CC BY 4.0** | [`LICENSE-CC-BY`](./LICENSE-CC-BY) |
| code samples — the `curl` and `jsonc` blocks | **MIT** | [`LICENSE`](./LICENSE) |

Cite a case freely; attribution is the only condition. That is deliberate. The cases describe
failures, and some of them will eventually describe failures in systems whose owners would prefer
the description not exist — "all rights reserved" would hand those owners a cheap argument that
quoting a case is infringement. CC BY removes the argument.

What the license does **not** cover: any third party's logs, traces, trademarks or data quoted
inside a submitted case. That stays with the submitter, who must have the right to publish it.

---

<p align="center">
<sub>If you find a case here that matches something you shipped, that is the list working as intended.<br/>
Open a PR with your trace instead of fixing it quietly.</sub>
</p>
