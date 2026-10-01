# What the top teams did, and what we can learn from it

Summaries of the two published top write-ups, in our own words. Read the originals for the detail; all credit
for the methods belongs to their authors.

- **1st place**, by `xz`: [write-up](https://www.kaggle.com/competitions/ai-agent-security-multi-step-tool-attacks/writeups/1st-place-solution),
  with a [custom GCG repository](https://github.com/xz259/Kaggle-AI-Agent-Security-1st-Place-Solution).
- **2nd place**, a five-person team: [write-up](https://www.kaggle.com/competitions/ai-agent-security-multi-step-tool-attacks/writeups/aas-2nd-no-shake-plz-no-shake-yes).

## The shared conclusion

Both teams reached the same answer we did not: **the private guardrail blocked exfiltration entirely, and the
confused-deputy family was the only route that scored there.** Both found it by *probing the private
leaderboard*, not by reasoning about it, and both then spent the remaining days making the confused-deputy path
as cheap as possible per candidate.

| | Best public | Private | Notes |
|---|---:|---:|---|
| 2nd place | 140.580 | 43.140 (selected submissions: 42.990 and 41.925) | A public score on the throughput ceiling, a private score under a third of it |
| 1st place | not stated | about 46.5 on their reported final run | Won on a small per-candidate throughput edge |
| **Us (best)** | **92.475** | **about 1.2 (earlier agents); final score 0** | We left that agent out of our final picks and bet on S19, which scored 0 privately |

The leader's private score was **under a third of the 2nd team's public score.** The ranking was decided on a
family the public leaderboard barely rewarded.

## How they found out what the private guardrail allowed

**1st place** used a two-message probe: message one makes the tool call being tested; message two says to stop
if it succeeded, otherwise to count from 000 to 999. A blocked call leaves the model counting, which is slow and
visible. Many probes ran together with a clock mechanism to remove queue-time ambiguity. Findings: calls with
"secret" in the arguments were blocked; an action after a web or email read stayed blocked for at least the next
two calls; confused-deputy calls ran reliably.

**2nd place** turned the same idea into a **timing oracle**. A candidate the guardrail allows keeps generating and
executing tool calls for the whole replay budget; one denied on its first sensitive call stops almost at once.
They measured, locally, the time for an allowed and a denied run (ratio `R`), picked a candidate count that
would fill the 8,750-second budget if allowed, and compared the observed wall time with the two predictions:

| Public | Private | Expected replay time |
|---|---|---|
| allow | allow | 2B |
| allow | deny | (1 + R)B |
| deny | deny | 2RB |

Their exfiltration probe finished in 2.75 h against predictions of 2.84 h (deny) and 4.86 h (allow), so they
classified it as denied. Final scores of 71.045 public and 0 private confirmed it. Their clean email probe ran
4.57 h, close to the allow prediction, and kept its score. Seven other families (file reads and writes, and
chains after web content) all landed near their deny predictions and scored zero privately. They treated
borderline durations as inconclusive and preferred probes with a large `(1 − R)B` gap.

## How they optimised the confused-deputy path

| Idea | Why it mattered |
|---|---|
| **One tool call per candidate, not eight** | A confused-deputy hit is worth 4 points against 16 for exfiltration, so the **+2 novelty bonus is half the value**. Eight-step chains gave up novelty for depth; one step with a distinct recipient kept it |
| **Put the only varying part at the end of the message** | The shared prefix is cached, so it costs almost nothing |
| **Make the second model turn empty** | After the tool call executes, the model is called again for a short reply that scores nothing. Cutting those tokens is pure throughput |
| **Remove wasted structural tokens** | Some tokens appear in the raw reply but vanish when it is parsed into a tool call; they can be dropped without changing the result |
| **Filter and order the recipient list** | Remove recipients that cause malformed calls, extra prose or unstable final turns; replay the fastest first |
| **Re-run that filtering in the real scoring environment** | Removes environment-specific bias; "only a slight improvement", but enough to beat teams using the same mechanism |

**The 1st-place edge** was a step further: use gradient-based search (GCG) to make the model emit the
end-of-generation token on that second turn. They had to work around three things: the competition models are
quantised GGUF files with no gradients (so they searched on the full-precision checkpoints as proxies), the
logits drifted between the two formats (so they trained a small re-ranker), and the inference cache made results
depend on the preceding candidates (so the final shortlist was evaluated in replay order). It worked reliably for
Gemma. For gpt-oss it succeeded on the proxy but regressed on the real model, and they stopped. They also found
that one version of the inference library shifted some logits by up to 2, and re-screened under an older one.
Their first working submission scored about 44.5; the final steps took it to about 46.5.

## What we can learn

1. **Probe the thing you cannot see.** Run time is a side channel for a hidden evaluation. Both top teams spent
   submissions on probes with a clear prediction for each outcome. We spent ours on the number we could read.
2. **Test each scoring family separately under the hidden condition.** The 2nd team tested eight families and
   found the single survivor. We assumed that exfiltration would survive and hedged with a small slice.
3. **A hedge at 5% was too small and the wrong shape.** The surviving family needed its own optimisation: one
   call per candidate, empty second turn, cache-friendly prefix, a screened recipient list. Ours reused the
   eight-step machinery and was never tuned.
4. **Re-derive the economics when the weights change.** At 16 points per hit, novelty was not worth the latency.
   At 4 points per hit, it was half the score. The same optimisation gave opposite answers.
5. **Our throughput theory held up.** Our notes put the exfiltration ceiling at roughly 135–140 normalised;
   the 2nd team reached 140.58 publicly. Understanding the system was right. We aimed it at the wrong row.
6. **Match the environment exactly.** The 1st-place author found a library version that shifted logits by up to 2
   and screened under the closer one. Small stack differences are large when an attack wins by narrow margins.
7. **Keep a human setting the goal.** The 2nd team tried an autonomous agent pipeline. It reached roughly bronze
   level and would not see the value in a failed idea; the useful work came from precise goals set by people.
   That matches how we used our own AI assistants.
8. **Single scores mislead here.** Our identical-bytes result (17.5 points apart) was larger than most of the
   effects we tested, so decisions need repeats, or a mechanism you can predict, such as a timing probe.
