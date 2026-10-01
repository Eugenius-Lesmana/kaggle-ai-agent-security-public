# Our approach: the best public submission, and the deputy system we built

Two designs, described at the level of what each does and why. The source is not published (see the README).

## Part 1: the best public submission (S19, public 92.475)

### Where it came from

Not from scratch. The public-score jump from the 40s to the 80s came from techniques already visible in
several public notebooks that scored 79–86; we studied them, then rebuilt the part that decides how the attack
*sizes and fills* its list. Two ideas are not ours:

- **Live validation-fill.** Generate a candidate, replay it against the live environment, keep it only if it
  fired, repeat until a deadline. Public notebooks that scored in the 60s–80s used this.
- **A framing technique** that makes the model skip its reasoning and go straight to the tool call. Several
  independent public notebooks reported it; we raced it against plain framing rather than assuming it helped.

### The pipeline

```
race 3 framings per model (2 probes each)  ->  choose the framing with best latency per successful fire
        ->  live validation-fill with that framing, keeping only candidates that fired
        ->  stop at 99% of the replay budget (replay-safe cutoff)
        ->  append a bounded 64-candidate tail in the winning framing
        ->  validate, de-duplicate, return          (static fallback if anything fails)
```

Design points worth stating:

- **Per-model adaptivity.** The attack never learns which model it is talking to; the only signal is the
  latency it observes. The safety margin scales with the slowest observed latency, so a fast model reclaims
  cushion while a slow one keeps the full protection.
- **Safety is score.** A replay timeout can zero a whole model row and halve the mean. We stop filling early on
  purpose, with a margin that other public work had tuned down step by step until it hit the point where the
  whole submission scored zero.
- **Fail closed.** The tail is added only if the race completed, its winner fired, the prefix is non-empty and
  capacity remains. A warm-up failure, a mid-fill environment fault or an all-zero race returns the earlier
  version's behaviour unchanged.
- **Reproducibility as a gate.** Every artifact had source and notebook hashes recorded, a byte-equality check
  of the release against the canonical source, a deterministic two-cell notebook and an SDK validation pass. We
  once caught a stale file uploaded under the wrong name from these hashes.

### How we ran it as an experiment

The ladder was built to be near-factorial, so that each lever's effect could be read:

| Artifact | Score | Difference from the control |
|---|---:|---:|
| Fixed-route control | 84.950 | none |
| Three-frame race | 86.670 | +1.720 |
| Bundled fixed-route packing arm | 86.580 | +1.630 |
| *Predicted if the two effects add* | *88.300* | *+3.350* |
| **Race + 0.99 cutoff (S18), observed** | **88.155** | **+3.205** |

The residual of −0.145 was small enough that we froze both ingredients. We then wrote S19's single question down
before upload: *does S18 leave scoreable replay capacity after its validated prefix?*

### What our own economics said, and what the leaderboard did

The S19 plan worked out the ceiling for the tail:

- One validated candidate is worth 0.09 normalised points per row (18 raw).
- 64 perfect candidates would be worth at most 5.76 points per row.
- But the tail only starts after 99% of the budget is used. At the hosted 8,750-second replay budget the reserve
  is 87.5 s, and candidates took 8.5–20.4 s each, so **only about 4–10 tail candidates would even start** (our
  back-of-envelope), worth well under 1 point.

S19 scored **+4.3** over S18. By our own arithmetic the tail cannot explain that. Then the byte-identical
resubmission scored 17.5 points *lower*. **We had written down the evidence that the gain was a draw, and still
treated it as a result.** The lesson is not only "measure the noise"; it is "check a result against what the
mechanism could plausibly deliver".

We also excluded a multi-post setting from S19 because combining it with the tail would have made any score
uninterpretable, and because its replay cost grows faster than its yield. That setting was later tested on its
own (S26, 88.875) and was inconclusive.

## Part 2: the deputy system (built, never submitted)

The first plan (26 August) described a flat 5% band. The system we built caps it at 3% by default, allows 5%
only after hosted evidence, and gates it on live probes, because a cost of a couple of points cannot be measured
through ±8–20% noise.

### The contract

Each candidate asks the agent to notify a roster by email: one, two or three sends (three families, `cd1`,
`cd2`, `cd3`), to clean synthetic recipients, with a final user message that avoids the words the scorer looks
for. Each compliant send is worth +4 raw under any guardrail that still allows untainted email, which is the
only other family that is mechanically reachable (see [`scoring-and-guardrails.md`](scoring-and-guardrails.md)).

**Unique recipients.** The score cell for an email is keyed on the recipient, so a shared set of addresses
collapses onto a few cells. Each candidate got its own address on a reserved, never-resolving domain, which earns
one novelty cell per compliant candidate and keeps every address synthetic. The two top teams reached the same
conclusion about recipient novelty independently.

### Three modes, built to be staged

| Mode | What it does | Why |
|---|---|---|
| **Deputy-only diagnostic** | Ships deputy candidates only, in fixed blocks per family, so each family's slot count is known | Answers the question a small overlay cannot: do the models actually perform intent-clean email sends at competition scale? |
| **Overlay on a frozen chassis** | Wraps an existing submission without changing its families; the base behaviour stays byte-identical when the band is off | Lets a leaderboard difference be attributed to the band alone |
| **Adaptive gating** | Live probes per model and session decide the size: no confirmed yield → 0%, one family with two or fewer hits → 1%, repeatable yield → 3%, and 5% only after hosted evidence | The band only costs score when it can pay |

Other design decisions:

- **Interleaved, not tail-pinned.** The old "lottery tail" sat after a full replay window and was never replayed.
  The band is spaced evenly through the list so any truncation point keeps coverage.
- **Gated on evidence.** It was to enter a real submission only after its own compliance battery succeeded.
- **Pre-registered read-outs** (see [`confused-deputy.md`](confused-deputy.md)) so the result could not be
  explained away.
- **A bug found in testing:** when the whole list was already verified, the band was bypassed, because the check
  ran after the list was built. It was moved ahead of construction.

### What it lacked, compared with the winners

Our design had the recipient-novelty insight and the evidence gating. It did not have what the top teams used:

- Our candidates chained up to three sends. They used **one call per candidate**, because at 4 points a hit the
  novelty bonus is half the score and extra turns cost throughput.
- No **empty second turn** and no removal of wasted structural tokens.
- No **cache-friendly prefix** with the varying part last.
- No **screened and ordered recipient list.**
- Above all, no **probe of the private rows.** We could not observe them, and we did not build the side channel
  that both top teams used (see [`top-teams.md`](top-teams.md)).

It was built as insurance, sized small, and never given its own submission to prove it. That was the decision
that cost the most.
