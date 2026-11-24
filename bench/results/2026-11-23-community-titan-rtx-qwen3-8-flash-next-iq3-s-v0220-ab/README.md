# v0.2.20 against the IQ3_S pack - paired with the 2026-11-13 sessions, TITAN RTX

Does engine 0.1.66 (upstream `v0.2.20`, merged at `38bceffa`) move the plain IQ3_S
pack on sm_75? Three new `strata_bench` sessions (2026-11-23) run with the current
build are matched session by session against the three 2026-11-13 sessions, which ran
engine 0.1.59 at the pre-merge branch tip `f7ce6aa0`.

- Old sessions: `2026-11-13-community-titan-rtx-qwen3-8-flash-next-iq3-s{,-2,-3}`
- New sessions: `2026-11-23-community-titan-rtx-qwen3-8-flash-next-iq3-s{,-2,-3}`
- New build: engine 0.1.66 (`src 723a753bd318e626`), compiled 2026-11-23 on the merge,
  CUDA 13.3.73, arch 75 - the same binary the UD-IQ3_XXS A/B of the same day tested.
- The current `terminal/strata-iq3_s.json` carries three environment settings added
  since 11-13 (`STRATA_KVA=0`, `STRATA_PREFILL_CPU_SHARE=0`, `STRATA_HC_Q8=1`). To keep
  the pair an engine-only change, the new sessions ran `terminal/strata-iq3_s.ab1113.json`,
  a copy with the env frozen to the 11-13 set (`STRATA_SM75_PERF=1`,
  `STRATA_ROUTE_TAIL_SKIP=0`, `STRATA_ROUTE_PRIOR=0`); everything else is unchanged.
- Method: the pairing follows `strata_bench --compare` (chats matched by run x question,
  prompt tests by run slot); Wilcoxon signed-rank on log ratios over the pooled pairs of
  the three matched session pairs, Holm-corrected across the six dimensions; floors
  2% prompt / 5% decode.

## Pooled verdicts

| dimension | pairs | B/A | 95% CI | raw p | Holm p | size |
|---|---:|---|---|---|---|---|
| 16K prompt tok/s | 15 | **x1.140** | 1.129 - 1.154 | 0.000061 | 0.000305 | faster, over the floor |
| 16K first token | 15 | **x1.139** | 1.128 - 1.153 | 0.000061 | 0.000305 | faster, over the floor |
| chat first token | 44 | **x1.157** | 1.108 - 1.199 | 0.000001 | 0.000001 | faster, over the floor |
| 4K prompt tok/s | 15 | **x1.111** | 1.073 - 1.147 | 0.000122 | 0.000366 | faster, over the floor |
| 4K first token | 15 | **x1.110** | 1.072 - 1.146 | 0.000122 | 0.000366 | faster, over the floor |
| chat decode tok/s | 44 | x1.019 | 0.987 - 1.045 | 0.316 | 1.000 | unknown (test not passed) |

A single session with `--runs 5` gives 5 pairs and can never be significant (the tool
says so); three matched sessions pooled give 15 prompt pairs and 44 chat pairs, whose
floor (2^-14) clears Holm with room.

## Medians per session

| session | chat decode | chat 1st tok | 4K prompt | 4K 1st tok | 16K prompt | 16K 1st tok | agent decode |
|---|---:|---:|---:|---:|---:|---:|---:|
| 11-13 (0.1.59) #1 | 54.3 | 0.69 | 988.5 | 4.10 | 1328.9 | 12.13 | 66.8 |
| 11-13 #2 | 52.6 | 0.77 | 1029.7 | 3.94 | 1314.4 | 12.26 | 67.2 |
| 11-13 #3 | 46.7 | 0.92 | 1038.4 | 3.90 | 1318.9 | 12.22 | 67.2 |
| 11-23 (0.1.66) #1 | 56.9 | 0.62 | 1127.6 | 3.60 | 1511.1 | 10.67 | 68.2 |
| 11-23 #2 | 48.7 | 0.66 | 1145.2 | 3.54 | 1499.8 | 10.75 | 73.0 |
| 11-23 #3 | 47.5 | 0.76 | 1087.3 | 3.73 | 1487.8 | 10.84 | 69.6 |

(agent rows stay descriptive: the sessions are different conversations.)

## Verdict

v0.2.20 is **statistically significantly faster on the IQ3_S pack** too:
16K prefill **+14.0% [+12.9%, +15.4%]**, 4K **+11.1% [+7.3%, +14.7%]**, and chat
first token **+15.7%** - every prefill dimension clears the 2% floor with Holm
p <= 0.0004. **Decode is unchanged**: chat decode x1.019 with CI up to 1.045, under
the 5% floor - matching the UD-IQ3_XXS A/B of the same day (decode flat there too).

## Against the UD-IQ3_XXS numbers (same engine pair)

- 0.1.59 on the two quants (11-13 sessions, same suite): 16K 1,314-1,329 (IQ3_S) vs
  1,319-1,328 (UD-IQ3_XXS) - prefill equivalent; chat decode ranges overlap.
- 0.1.66 gain: +11-14% on IQ3_S (strata_bench, this campaign) vs +21.5% at 6,345
  tokens on UD-IQ3_XXS (ab_engine harness). The two harnesses and prompt lengths
  differ, so this is not a paired quant-vs-quant test - it says both quants gained,
  XXS more at mid-length prompts.

## Not tested

The current config env (`STRATA_KVA=0` etc.) against 0.1.59 - those settings are
config changes, not engine changes, and stay out of the pair.
