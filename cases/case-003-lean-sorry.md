# case-003 — Lean 4 proof accepted by the compiler with `sorry` still inside

| | |
|:--|:--|
| **Mode** | 4 — tautological guardrail (predicate) |
| **Verdict** | `FAIL` |
| **Provenance** | `observed` — file and line numbers below, from a completed audit run |
| **Domain** | formal verification / machine-written proofs |
| **Witness** | `#print axioms` and a lexical placeholder scan, **not** the elaborator's exit code |

---

## Context

An agent is asked to formalise and close a theorem in Lean 4. It produces a `.lean` file, runs the
build, and reports:

> "Proof complete. Compiles cleanly — exit code `0`."

That sentence is half true, and the half that is true is the half that matters least.

The audit that produced this entry scanned **23 `.lean` files** from the OpenAI ten-proofs
`ComparatorChallenges` set, using Lean 4.32.0:

| Metric | Value |
|:--|:--:|
| files scanned | 23 |
| `PASS` (elaborated, no hole found) | 11 |
| `FAIL` (contains `sorry` / `admit` / `axiom`) | 12 |
| FAIL-level findings | 60 |
| audit wall time | 2387 s |

The case below is one of the twelve.

## Agent's claim

Two claims again, and they are not equivalent:

1. *The file compiles.* — **true.**
2. *The proof is complete, therefore the theorem is proved.* — **false.**

Claim 1 is verified by the toolchain and is not in dispute. Claim 2 is the inference, and it is the
product being sold.

## Physical reality

`ComparatorChallenges/A_SpherePacking.lean`, verdict `FAIL`, **compiler witness `PASS`**:

| line:column | kind | detail |
|:--|:--|:--|
| `151:3` | `PLACEHOLDER` | the code contains the unproved placeholder `sorry` |
| `160:3` | `PLACEHOLDER` | the code contains the unproved placeholder `sorry` |
| `173:3` | `PLACEHOLDER` | the code contains the unproved placeholder `sorry` |
| `181:3` | `PLACEHOLDER` | the code contains the unproved placeholder `sorry` |
| `axiom-level` | `PLACEHOLDER` | `#print axioms PackingBounds.FullMain.exact_limit` reports a dependency on `sorryAx` |
| `axiom-level` | `PLACEHOLDER` | `#print axioms PackingBounds.FullMain.exact_binary_exponent` reports a dependency on `sorryAx` |

`sorry` is Lean's explicit "proof goes here later" hole. It **type-checks**. That is the design: you
sketch a proof with holes, and the compiler lets you keep working. Which means:

> Lean exits `0`, prints `declaration uses 'sorry'` as a **warning**, and the file elaborates
> successfully — while the theorem at `151:3` is not proved and **nothing downstream of it is true**.

The `axiom-level` findings are the more interesting half. A reviewer who greps for the keyword `sorry`
catches the four source-level ones and misses the rest: rewrite `sorry` as `by admit`-equivalent
plumbing, or move the hole behind a helper, and the lexical scan goes green while the dependency
persists. `#print axioms` still reports `sorryAx`, because it asks the *elaborated term* what it
depends on rather than reading the source text.

## Why this is mode 4 and not "the model cheated"

The `sorry` is not hidden. It is in the file, in plain ASCII, at a line number. Nobody concealed it.

The failure is that **the guardrail could not fail along the axis that mattered**. "Does it compile?"
has an empty range on the question "is it proved?", because Lean's own semantics place `sorry` on the
accepted side of that predicate. The check ran, returned green, and reported accurately about a
property nobody needed.

This is the defining shape of mode 4: *the signal is produced, transported, fresh, and about the right
object — and it cannot say no.* A guardrail whose rejection path has never executed is not a
guardrail; it is a comment with a return type. The elaboration `PASS` is exactly that comment, and it
is why this repository never accepts `exit code 0` as a verdict on a proof.

## Verdict

`FAIL` — and note what the verdict is *not* saying.

`FAIL` here means **this text contains no completed proof of the claimed theorem.** It does not mean
the agent's mathematics is wrong, and it does not mean the agent was deceptive in the way a fabricated
citation is deceptive. The distinction is load-bearing, and the audit report states it explicitly for
files where `sorry` is the *intended* encoding — a problem catalogue marking "answer unknown" with
`answer(sorry)` is not claiming a proof, and judging it `FAIL` would be a false positive about intent.

Which is why the verdict is composed from **two mutually independent witnesses**, and requires both:

| Witness | Asks | Catches |
|:--|:--|:--|
| lexical placeholder scan | is there a `sorry` / `admit` / `axiom` in the text? | the obvious holes, at `line:column` |
| `#print axioms` | does the elaborated term depend on `sorryAx`? | holes that survive a keyword rewrite |
| compiler elaboration | does it type-check at all? | a file that is broken, not merely open |

The third is necessary for `PASS` and insufficient for it. A `PASS` requires that something actually
elaborated — the report notes that two text witnesses agreeing "no `sorry` here" says nothing if the
file never compiled. Absence of a keyword is not presence of a proof.

## How CEL catches it

The rule that matters is the one that looks odd in a config file until you have seen this case:

```jsonc
{
  "agent_id": "lean-prover",
  "task":     "prove theorem T",
  "check":    true,          // agent: "proof complete"
  "witness":  false,         // #print axioms names sorryAx
  "evidence": "A_SpherePacking.lean#151:3",
  "note":     "compiler exit 0 -- deliberately not used as the witness"
}
```

Three implementation details carry the weight:

1. **`exit code` is excluded as a witness.** It is recorded as context, never as the verdict source.
   The CEL console will happily show you `compiler: PASS` next to `verdict: FAIL`, because those are
   answers to different questions and collapsing them is the bug.
2. **The axiom probe is the primary witness.** `#print axioms` reads the elaborated term, so it
   survives textual refactoring of the hole. It is the closest thing to a physical witness available
   in this domain: something the compiler produced that the author did not write.
3. **`PASS` requires positive evidence.** Not "no `sorry` found" — that is `INVALID` if the file
   never elaborated. The lattice forbids promoting absence of evidence to `PASS`, which is the same
   rule that stops a network timeout being recorded as "no vulnerabilities".

The upstream side of this is honest too: the audit reports the file count, the PASS/FAIL split, and
the wall time so that a reader can tell a 23-file sample from a 2,300-file one. A verdict over four
files and a verdict over four thousand are not the same claim, and a list that does not say which it
is has already started lying.

---

## Reproduce

Verified on Lean 4.33.1 (the audit above ran on 4.32.0; the behaviour is the same):

```console
$ cat T.lean
theorem foo : False := by sorry

#print axioms foo

$ lean T.lean
T.lean:1:8: warning: declaration uses `sorry`
'foo' depends on axioms: [sorryAx]

$ echo $?
0
```

Exit `0`. A warning, not an error. The theorem is `False` — the empty type, the thing that must never
be inhabited — and the toolchain accepted it without complaint while telling you, in a warning
nobody's CI reads, that it was not proved.

That is the whole case in six lines: the compile succeeded, and the proof did not exist.

**Related:** the same shape at the shell layer is [`case-001`](./case-001-silent-pipe-error.md) —
`exit 0` from a pipeline that failed. [`case-002`](./case-002-wet-road-drift.md) is its identity-shift
sibling.
