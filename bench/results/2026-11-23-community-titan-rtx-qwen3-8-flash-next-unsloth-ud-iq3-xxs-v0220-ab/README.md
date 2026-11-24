# v0.2.20 merged in - is it faster? TITAN RTX, UD-IQ3_XXS pack

Paired A/B of the engine before and after merging upstream `v0.2.20` (engine 0.1.66,
which claims Unsloth-quant CUDA speedups "sm_60..sm_121") into `feature/paired-ab-stats`,
on the `strata-unsloth-ud-iq3_xxs.json` config.

Builds:

- **A (baseline)**: engine 0.1.59, the installed binary whose `BUILD.json` stamp
  (`src a1e7988656dc0588`) equals `source_hash(ENGINE_SOURCES)` recomputed at the
  pre-merge branch tip `f7ce6aa0` (upstream 0.2.13 + this branch's sm_75 prefill/GDN
  work). CUDA arch 75, built 2026-11-09.
- **B**: engine 0.1.66 (`src 723a753bd318e626`), compiled 2026-11-23 on the merge
  commit `38bceffa` (upstream/main `v0.2.20` + the branch), CUDA 13.3.73, arch 75,
  vision gpu.

Both arms ran under the same post-merge `terminal/serve/server.py` (0.2.20; its
`MIN_ENGINE` 0.1.46 accepts 0.1.59) and the same config/env
(`STRATA_SM75_PERF=1`, MTP drafting, `--spec 4`, KV int8, 262k context).

Arms (10 paired rounds, alternating order, `terminal/tools/ab_engine.py --stats`,
floors 2% prompt / 5% decode):

| metric | B/A | 95% CI | raw p | Holm p | size |
|---|---|---|---|---|---|
| long prompt (6,345 tok) | **x1.215** | **1.160 - 1.275** | 0.0020 | 0.0098 | faster, over the floor |
| chat0 decode | x0.957 | 0.922 - 1.032 | 0.1934 | 0.7734 | unknown (test not passed) |
| chat1 decode | x1.008 | 0.944 - 1.060 | 0.8457 | 1.0000 | unknown (test not passed) |
| chat2 decode | x1.011 | 0.931 - 1.093 | 0.6523 | 1.0000 | unknown (test not passed) |
| long decode | x0.972 | 0.902 - 1.046 | 0.3223 | 0.9668 | unknown (test not passed) |

(`chat2` has 9 usable pairs: one pair's ratio was exactly 1 and drops out of the
signed-rank test; all 80 request rows ran.)

Medians over 10 rounds: long prompt 1,178.9 -> 1,440.5 tok/s; decode chat0 50.8 -> 48.0,
chat1 54.0 -> 54.5, chat2 54.6 -> 56.7, long 60.0 -> 58.0 tok/s. The short-chat prompt
numbers (not a test dimension) also move up together, 50.8/35.2/35.2 -> 68.7/56.0/50.0.

## Verdict

v0.2.20 is **statistically significantly faster at prefill** for this quant on this
card: **+21.5% [+16.0%, +27.5%]** on the 6,345-token prompt, Holm p = 0.0098 over five
metrics. **No decode speedup is visible**: every decode CI straddles 1 and none clears
the 5% floor; the chat0/long CIs (upper ends 1.032 / 1.046) even make a >= 5% decode
gain implausible. The release's CUDA MoE-decode gains (1.04x-1.67x, fused `EXP_MODE=8`
expert pairs, INT8 TC dense rows) do not show on sm_75 with the UD-IQ3_XXS pack and
this adaptive-expert-cache config.

Greedy outputs are not byte-identical between the arms (the MTP drafter's accepted
length per request changed, n 124-160 vs 133-161); the timings normalize per generated
token.

## Not tested

Other quants and packs, other GPUs, `--speed` settings, vision requests, agent-turn
workloads.

Raw rows `ab_engine.jsonl`, the harness table `ab_engine.out`.
