# The hedge we built and did not ship: the confused-deputy band

Written 26 August, five days before the deadline. Reproduced here as a record of what we knew and when,
because the private leaderboard later decided on exactly this point.

## 1. The risk

Two facts, both from the evaluation source:

1. The final ranking combines the public and the private rows, and we could select only one or two final submissions.
2. The private guardrail is **one fixed ruleset applied identically to every competitor**, and we could not see
   it. All of our banked score came from one family: a clean web request carrying a marker.

If the private ruleset denies those requests, **every competitor's private row collapses together**, and the
rank is decided by whoever kept a secondary scoring path alive. At that point we had none. The "lottery tail" we
did have sat at the end of a very long list, behind a replay window that was already full, so it would never
have been replayed. It was decorative.

## 2. The design: a deputy band

The first plan reserved about **5% of the candidate list** (the system we built later capped it at 3% and made it
adaptive; see [`approach.md`](approach.md)) for the **confused-deputy** family: short chains that ask the agent
to notify a roster, which makes it send one to three ordinary emails to clean, synthetic recipients, with no
sensitive targets and a final user message that avoids the words *send*, *email* and *upload*.

- Each compliant email is worth **+4 raw under any guardrail that still allows untainted email**, including one
  stricter than the public one.
- It is the **only other predicate that is mechanically reachable**: the taint and keyword families are blocked
  publicly, and a stricter private guardrail cannot relax a block.
- The band is **interleaved** through the list with even spacing, not pinned to the end, so any truncation point
  keeps roughly 5% coverage. If a stricter guardrail denies the main family, replay gets faster and the window
  stretches, so the band's coverage deepens automatically.
- Deputy candidates go through the same live "does it actually fire" screening as everything else.
- Subjects carry the candidate index, which avoids a deduplication collision that silently dropped an earlier
  version of the band.

## 3. The expected cost, honestly

Let `r` be the deputy yield as a fraction of the main family's yield per list slot. We could not know it until it
was measured on the hosted system. A band of 5% is worth about 2.4 normalised points per row.

| Scenario | Public effect | Private effect | Net |
|---|---|---|---|
| Private allows the main family | −2.4 × (1 − r) | about 0 | −2.4 × (1 − r) |
| Private kills it | −2.4 × (1 − r) | **+2.4 × r** | +2.4 r − 2.4 (1 − r) |

- If the deputy complies reasonably (r ≥ 0.4–0.5) the hedge is close to free.
- If it measures about zero (r ≈ 0) it costs about 2.4 points in the permissive world and buys nothing.
- **Rank asymmetry favoured the hedge.** If the private guardrail killed the main family, +1 or +2 points while
  everyone else sat near zero would jump many ranks. In the permissive world, losing 1.5 points inside a cluster
  of scores in the 40s and 50s would cost fewer ranks than it could gain.

## 4. Pre-registered read-outs

Decided before we looked at any hosted data:

- Generation-stage probes (free): deputy probes scoring above zero confirm `r > 0`. All-zero probes mean the
  band is dead weight and should be switched off before upload.
- Leaderboard comparison, using the matching version without the band:
  - within 2% of it → the band is nearly free; keep it permanently;
  - 3–5% below it → the premium is real; decide on our belief about the private guardrail;
  - more than 6% below it → deputies are non-compliant and crowding out the main family; revert.
- The band was never to exceed about 7% without new evidence, and was never to be called "proven".

## 5. What happened

We built the system described in [`approach.md`](approach.md), a diagnostic mode, an overlay and adaptive
gating, and tested it a little, but never expanded it or gave it a submission of its own. By our account it
**did not help much** when we tried it, and most strong public notebooks did exfiltration only, so we stopped and put the remaining uploads into the main path.
With leaderboard noise of ±8–20% we also could not see a 2.4-point cost, which meant the expected-value
argument in §3 was the only honest way to decide. We did not follow it.

On the private rows the main family was blocked and our score fell to almost nothing. The confused-deputy path
was the one that still fired.

## 6. How the top private teams differed

Both the 1st and 2nd place teams found that the private guardrail blocked exfiltration entirely and that
confused deputy was the only family that scored there. They learned this by probing the private rows (the 2nd
place team used run time as an oracle across eight families), then optimised the confused-deputy path: one tool
call per candidate so the novelty bonus kept its half of the value, an empty second model turn, a cache-friendly
prefix, and a screened, ordered recipient list. The 1st-place team used gradient-based search to shorten the
second turn. Our band reused the eight-step machinery and was never tuned.

Full comparison: [`top-teams.md`](top-teams.md).

## 7. What we would do differently

1. **Treat the hedge as part of the submission.** We named the risk, sized it, and then optimised only what we
   could see.
2. **Measure the unobservable.** Run time is a side channel for the private row. A probe submission that exists
   only to learn what the private guardrail allows is worth a slot.
3. **Do not let noise decide a small insurance premium.** A ±8–20% noise band cannot measure a 2.4-point cost, so
   the decision had to rest on the expected-value argument in §3, and it favoured the hedge.
