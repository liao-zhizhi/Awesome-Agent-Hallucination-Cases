# case-NNN — <one line, mechanism first, not vibes>

<!--
COPY THIS FILE TO  cases/case-NNN-<slug>.md  AND FILL EVERY FIELD.
A field you cannot fill is the finding. Write "unknown" and say why --
do not write a plausible value. A wrong value that looks verifiable is
worse than an honest gap, because the next reader will trust it.

Delete these comments before opening the PR.
-->

| | |
|:--|:--|
| **Mode** | `1` / `2` / `3` / `4` — see the table in [`README.md`](../README.md#the-four-modes-of-losing-the-referent) |
| **Verdict** | `PASS` / `FAIL` / `INVALID` / `NOT RUN` |
| **Provenance** | `observed` / `constructed` / `disputed` |
| **Domain** | e.g. shell, browser automation, robotics, driving, formal verification, data pipeline |
| **Witness** | what observed the truth, in one phrase — `PIPESTATUS`, `#print axioms`, `stat`, a decelerometer, … |

---

## Context

What was the agent asked to do, and what did the surrounding system look like? Minimum viable
detail: the task, the tool or harness involved, and anything a replicator would have to know.
Two paragraphs maximum. If the context needs more than that, the case is probably two cases.

## Agent's claim

The claim, **quoted or timestamped**. Not paraphrased — paraphrasing a claim is how a case becomes
unfalsifiable.

```text
<paste the exact output, log line, or message. Include the model/tool name and version
 if the claim is version-dependent.>
```

If the agent made more than one claim, separate them. The interesting cases almost always have a
trivially true claim stacked under a false one:

1. *"The command ran"* — true.
2. *"The command succeeded"* — false.

Name which one is being disputed.

## Physical reality

What actually happened, **as observed by something that is not the agent**. Include the raw evidence:

```console
$ <the command you ran to check>
<its exact output>
$ echo $?
<exit status>
```

Rules for this section:

- If the evidence is a **file**, give its path, its size, and its sha256.
- If the evidence is a **measurement**, give the instrument, the units, and the timestamp.
- If the evidence is **gone**, say so, and say what remains (a commit, a report, a screenshot with a
  date). An entry whose artefact no longer exists is still valuable — as long as it says that.
- **Never** use the agent's own logs as the witness for the agent's claim. That is not evidence; it
  is the claim again, formatted.

## Verdict

One of the four, and **one line on why not the neighbouring value**. This is the part reviewers
check hardest:

```text
FAIL -- not INVALID: a witness exists and is unambiguous (<what it observed>).
        not NOT RUN: the check did execute (<where>).
```

A verdict without a "why not the adjacent one" is a guess with a capital letter.

## How CEL catches it

The `verify` payload, with the two inputs kept separate:

```jsonc
{
  "agent_id": "<stable id>",
  "task":     "<what was attempted>",
  "check":    true,          // the agent's claim
  "witness":  false,         // the independent observation
  "evidence": "<path, hash, probe id, or measurement>"   // optional but preferred
}
```

Then, in one or two sentences: **what makes the witness independent?** If it imports the same
constant, calls the same function, or reads the same file the agent wrote, it is not a witness — it
is the claim in a second file, and it will agree every time.

## What would not have helped

List the fixes that people will propose, and why the mechanism survives each. This section is what
turns a case into a lesson:

| Proposed fix | Why the mechanism survives it |
|:--|:--|
| <e.g. a better model> | <e.g. the signal is still destroyed in transit> |
| <e.g. retry on failure> | <e.g. the failure never surfaced, so nothing retries> |

If you cannot fill this table, the case is probably a bug report rather than a mechanism, and it
belongs in the tool's own issue tracker.

---

## Checklist — a PR missing any of these will be closed

- [ ] The **claim** is quoted, not paraphrased, and attributed to a named tool/model/version.
- [ ] The **witness** was produced by something other than the agent, and I have said what it is.
- [ ] The **verdict** says why it is not the neighbouring verdict.
- [ ] **Provenance** is one of `observed` / `constructed` / `disputed`, chosen honestly. A mechanism
      I reasoned about is `constructed`, no matter how confident I am that it happens.
- [ ] **Reproduction**: the shortest command sequence that shows it, *or* an explicit sentence
      saying reproduction is impossible and why. "Obviously reproducible" is not an answer.
- [ ] No secrets, no customer names, no personal data. Redact before submitting, not after review.
- [ ] The file is named `cases/case-NNN-<slug>.md` with the next free number.
- [ ] I have read [`case-001`](../cases/case-001-silent-pipe-error.md) and my entry is not a
      restatement of it with different nouns.

## Attribution

Name and link, or `anonymous`. If the case came from your own system, say so — the most useful
entries in this list are the ones where the submitter was the one who shipped the bug.

---

<p align="center"><sub>Submitting a case is not an accusation. Every entry here describes a signal
that lied, and signals are shared infrastructure.</sub></p>
