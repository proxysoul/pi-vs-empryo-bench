# Empryo vs pi — coding-agent head-to-head on real bugs

Two coding agents. Same models (Claude Haiku 4.5 / Sonnet 4.6 / Opus 4.8), same
machine, same three bug reports written the way a human writes them — no file
names, no hints. Hidden acceptance tests grade the fixes. **Separate Anthropic
API keys per agent**, so the provider's Console is the independent judge of
what each agent really consumed.

## Console-verified result (Anthropic billing, 2026-07-11)

| | [Empryo](https://empryo.com) | [pi](https://github.com/badlogic/pi-mono) |
|---|---|---|
| bugs fixed | **8 / 9** | 7 / 9 |
| input tokens billed | **1,089,262** | 6,209,183 (**5.7×**) |
| output tokens billed | **14,405** | 91,179 (**6.3×**) |
| cost billed | **$1.13** | $1.58 |
| wall clock | **4m 16s** | 10m 0s |

### The self-accounting finding

Because each agent had its own API key, we could audit each agent's own cost
reporting against the actual bill:

- **Empryo reported its usage 100% correctly** — 1,089,262 input tokens and
  $1.13, matching the Anthropic Console **to the token and to the cent**.
- **pi under-reported**: its own JSON events summed to 1.70M input tokens /
  $1.42, but Anthropic billed **6.21M input tokens / $1.58** — pi's telemetry
  misses ~73% of the input tokens it actually sends. (pi's Console total
  includes ~$0.10 of our setup probes; deducting them changes nothing
  material.)

If you benchmark agents on their self-reported numbers, use separate keys.

## Why the gap

Empryo injects a **code-intelligence graph** (symbols, calls, imports, ranked
by usage — the "genome") into context, so the model starts every task already
knowing where things live and what the code *actually does*. pi explores with
grep/read from zero every time.

The decisive task: a sort bug guarded by a stale comment claiming event ids
are ULIDs ("id order == arrival order") while the id generator went random two
versions earlier. Text exploration finds the comment and believes it — pi
(haiku) burned 35 steps / 165s / $0.18 self-reported and still shipped a wrong
fix. Empryo's graph doesn't repeat the comment's lie — haiku fixed it in 10
steps / 20s / $0.04. **Comments lie; graphs don't.**

Also observed: pi cannot run Opus 4.8 out of the box (its model registry ends
at 4-7 and its default thinking payload is rejected by the 4.8 API) — its opus
tier ran with a custom model id + `--thinking off`, the only working config.
And on the CSV task, *both* agents failed on opus with over-clever non-RFC
escaping that haiku and sonnet got right.

## Repository layout

- `fixture/` — **hookboard**, a realistic TypeScript webhook-inbox app with
  three genuinely committed bugs (lost-update race, CSV quoting, id-vs-time
  sort). Its own tests pass — the bugs are the kind tests didn't cover.
- `harness/run-vs.ts` — the driver (standalone: needs `bun`, `pi`, and
  `empryo` on PATH, or `EMPRYO_BIN`). `--sanity` proves the harness gates
  correctly with zero LLM calls.
- `harness/checks/` — the hidden acceptance tests (dropped in AFTER each run).
- `results/` — raw + merged JSON, including the Console ground-truth block.
- `report.html` — self-contained visual report.

## Reproduce

```sh
bun harness/run-vs.ts --sanity          # no LLM calls; verifies the harness
PI_BENCH_KEY=sk-ant-... EMPRYO_BENCH_KEY=sk-ant-... \
  bun harness/run-vs.ts --label my-run --budget-stop 12
```

Use two fresh API keys and compare your Console breakdown against the emitted
JSON. Single runs are stochastic — expect per-cell variance; the aggregate
direction reproduced across all nine cells for us.

## Honest caveats

Single trial per cell; three tasks, one language, one small repo; per-task
charts use self-reported per-cell figures (pi's true per-task costs are
proportionally higher than its self-report). Empryo indexes the fixture inside
the run (~2s, no LLM). Costs shown per each agent's pricing tables; billed
totals from the Console are the source of truth.

## Console-verified result — real-world round (Anthropic billing, 2026-07-16)

| | [Empryo](https://empryo.com) | [pi](https://github.com/badlogic/pi-mono) 0.80.7 |
|---|---|---|
| bugs fixed | **7 / 10** | 6 / 10 |
| cost billed | **$7.08** | $9.19 (**+30%**) |
| steps | **274** | 382 |
| wall clock | **22m 30s** | 32m 55s |

Self-accounting audit, round two: **Empryo's self-report matched the console
again** ($7.06 said vs $7.08 billed — display rounding). pi 0.80.7 fixed round
1's per-message undercounting (its Haiku tier matched exactly) but still
under-reported by **$2.94 (32%)** — its crashed Opus cell burned ~8 minutes of
billed model work and reported $0.00 because the process died before emitting
its accounting event. Raw rows: `results/vs-real-round3-final-merged.json`
(the voided cell and its retry are both preserved).

Tier story: at **Haiku** Empryo swept — 3/5 vs 2/5 fixed, 19% cheaper, exactly
half the wall clock. At **Opus** both fixed 4/5; only Empryo cracked zod's
parser regression on the first attempt (33 steps / $2.10 vs a hung first try
and a 69-step / $2.42 retry). ky's extend-retry bug defeated both agents at
both tiers.

## Round: real-world bugs (harness-real/)

The follow-up round leaves the synthetic fixture behind: **five real bugs from
real OSS repos** (hono ×2, zod, ky ×2), each taken from a merged fix PR:

- the agent gets the **real issue text verbatim** — user-written symptom
  reports, no file names, no hints
- the workspace is the repo at the fix PR's **parent commit** (bug live), with
  git history rewritten to a single baseline commit so the real fix is
  unreachable (`git log` reveals nothing)
- hidden acceptance = **the fix PR's own regression tests**, held outside the
  workspace and dropped in after the run
- every fix merged **after 2026-01** — past the models' training cutoffs, so
  no agent can regurgitate a memorized patch
- `harness-real/validate.ts` proves every task discriminates (tests fail by
  assertion on the bug, pass on the real fix) before a single token is spent

```sh
bun harness-real/validate.ts                     # no API keys needed
PI_BENCH_KEY=sk-... EMPRYO_BENCH_KEY=sk-... \
  bun harness-real/run-real.ts --tiers haiku,opus --label my-real-run
```

Note on methodology: official runs pre-warm Empryo's genome index and report
the wall-clock separately (`prewarmMs` per row). The portable harness can't
pre-warm without Empryo's source, so indexing lands inside the timed run —
Empryo's numbers here are slightly worse than the official method, never
better.
