# Experiment ledger

Every submission we recorded, in order. Scores are the **aggregate of the two public rows**, copied from the
leaderboard into our notes. Private-row scores are not shown here.

## Early ladder (my teammate's chassis)

These came from my teammate's earlier work, before I joined. They show which levers were already dead.

| Lever tested | Public score | Read |
|---|---:|---|
| Persuasion wording | 27.455 | Wording axis is dead |
| Forged context mixed into the pool | 31.400 | Neutral on its own |
| Forged drips + short URLs + larger list | 43.000 | +30%: a real gain |
| Per-model pools | 42.355 | About equal: specialisation is neutral |
| No injection text ("bare only") | 29.425 | Dead |
| Matched bare twins | 24.650 | −43%: the injection text is load-bearing |
| Bare + deeper chains | 29.160 | Dead (third proof) |
| Deeper forged chains + larger list | 45.500 | Best of the early ladder |

**Dead branches, confirmed more than once:** wording and persuasion searches · injection-stripped configurations ·
forged context alone · taint-based predicates under the public guardrail · destructive writes publicly ·
batch sizes below 8 · repeating identical candidates · self-selecting the best raw score.

## The later ladder

| ID | Change | Public score | Read |
|---|---|---:|---|
| S18 | Three-frame race with replay-safe cutoff 0.99 | 88.155 | Anchor |
| **S19** | **+ bounded 64-candidate tail** | **92.475** | Best |
| S20 | Best-first replay ordering | 79.020 | −9.1: generation latency did not transfer to replay ordering |
| S21 | Tail 64 × best-first | 90.900 | The interaction was +7.6, but ordering still cost 1.6 |
| S22 | Frame failover after six misses | 86.940 | Rejected |
| S23 | Tail 128 instead of 64 | 81.945 | Worse than 64 |
| *teammate* | Tail 256 on S19 | 90.540 | Reported to us; not audited |
| **S25** | **Byte-identical resubmission of S19** | **75.015** | **−17.46 on identical bytes** |
| S26 | S19 with one constant changed (multi-post 1 → 4) | 88.875 | Inconclusive against the noise |
| S27 | S19 + calibrated live validation of the first step | 89.640 | Inside the noise band |
| S28 | Tail 256 × multi-post 4, disjoint endpoints | 89.910 | A speculative final-day composite; highest of the three |
| S29 | S19 + only Tail 256 | 89.595 | The cleaner final-slot hedge |

One mistake worth recording: an upload intended as S28 was a stale file byte-identical to S26. We caught it from
the hashes. Every artifact had its source and notebook SHA-256 written down for this reason.

## The noise finding

S25 is S19. The hashes match; only the evaluation draw differed. 92.475 versus 75.015 is a **17.46-point
(−18.9%) swing on identical code**, and one earlier submission never scored at all.

What that does to the table above:

- S18 → S19 (+4.3) and S19 → S26 (−3.6) are both **well inside the noise band**. We cannot say the tail helped.
- S27, S28 and S29 (89.6–89.9) sit within 0.3 of each other and about 3 below S19: indistinguishable.
- S20 (−9.1) and S23 (−6.2) are also within it. We rejected them anyway, on the evidence we had.
- Only the large gaps tell us anything, such as the injection-text results in the early ladder (about −30% to −43%).

If we ran this again we would first submit the same file three times to measure the band, then decide how many
repeats each comparison needs. We did one repeat, near the end.

## Pre-registration: S19

Written before upload, so the result could not be argued afterwards.

- **Hypothesis:** S18's measured ledger, next-candidate reserve and 1% replay reserve leave some usable replay
  time; ordered replay will keep the exact S18 prefix and score part of a bounded tail.
- **Held constant:** every prompt byte, the three-frame race, two probe repetitions, the selection metric, the
  fired-probe retention, the live-fill order, the replay-safe fraction, the fallback and the wrapper.
- **Primary risk:** an unvalidated tail candidate can fail or occupy the final blocking replay call. A normal
  inner deadline keeps the findings, but a replay exception voids that model's evaluation.
- **No-tuning rule:** attribute the result only to the tail. Any change to tail length, replay fraction, framing
  or multi-post count gets a new experiment ID.

| Result versus S18 (88.155) | Decision |
|---|---|
| 0, null or a runtime failure | Keep S18; inspect packaging and the replay exception first |
| ≤ −0.45 | Keep S18; retire the tail unless logs show an unrelated failure |
| −0.45 to +0.45 | Inconclusive; do not stack the tail into another unproven lever |
| +0.45 to +0.90 | Capacity signal; keep S19 and seek confirmation |
| > +0.90 | Strong operational signal; use S19 as the next anchor |
| ≥ 95 | Goal reached; freeze |

S19 scored +4.3. By the table that is "strong". By the S25 result it is not distinguishable from a good draw.
**The thresholds were set before we knew the noise band, and they were too tight.** Writing them down first is
what let us see that.
