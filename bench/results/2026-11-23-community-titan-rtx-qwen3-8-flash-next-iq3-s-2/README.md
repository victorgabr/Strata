# Community benchmark on NVIDIA TITAN RTX

Measured on 2026-11-23 with `tools/strata_bench.py` (suite 1, 5 runs per test, medians with ranges). Speeds are the engine's own timings; first-token times are measured by the client over HTTP. Speed only: no correctness check was run.

## Hardware and software

- GPU and VRAM: NVIDIA TITAN RTX, 24 GiB VRAM
- CPU: Intel(R) Xeon(R) CPU E5-2660 v3 @ 2.60GHz (10 cores, 20 threads); installed RAM: 125.7 GiB; storage: sda     WDC WD6003FZBX-00K5WB0 sata; sdb     WDC WD6003FZBX-00K5WB0 sata; sdc     WDC WD161KRYZ-01AGBB0  sata; sdd     WDC WD161KRYZ-01AGBB0  sata; nvme0n1 Sabrent                nvme
- OS: Linux 6.1.183-1-MANJARO; GPU driver: 580.178.04; CUDA (driver supports up to): 13.0
- Strata: project 0.1.66, commit not a git checkout; engine 0.1.66
- GPU detail (after the run, near idle): power_limit_w=280.00, power_draw_w=262.40, pcie=gen3 x16, pstate=P2, temp_c=80
- Background workloads: not recorded (GPU power limit/draw read above)

## Model and configuration

- Model and quantization: qwen3.8-flash-next-iq3_s
- Context: 262144 tokens; parallel slots: 1
- Settings: the install's own configuration, unchanged; thinking off for the suite; temperature 0

```text
strata --pack iq3_s --native Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf --ple-gguf Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf --expert-profile expert-profile.bin --expert-cache auto --prefill auto --spec 4 --mtp rt --max-context 262144 --kv int8 --ple-io ram --kv-resident 32768 --vision --vram-reserve-mib 700 --pcie-frac 0.33 --spec-min-p 0.70 --adapt-every 1 --adapt-swaps 80 --adapt-decay 0.97
```

Environment of the config: `STRATA_SM75_PERF=1`, `STRATA_ROUTE_TAIL_SKIP=0`, `STRATA_ROUTE_PRIOR=0`

## Method

Run by `python tools/strata_bench.py --runs 5` against the HTTP API of a Strata server (started by the benchmark from the install configuration); model loading is not timed. One warm-up chat and a 1K prompt are discarded first. Every prompt starts with a unique tag, so the conversation cache cannot help the fresh-prompt tests.

- short chat: the three questions of `tools/ab_engine.py`, up to 100 new tokens, 5 runs
- 4K / 16K prompts: generated word lists sized by a calibration request, 24 new tokens, 5 fresh prompts each
- agent session: system prompt and four tools, then 5 turns that each add about 1500 words of a project file and continue from the model's own earlier answers (the way agent clients resend a conversation), 64 new tokens per turn, 5 sessions; the cache numbers come from the answer's usage (`cached_tokens`) and timings (`prompt_n`)
- concurrency (when run): N requests sent at the same moment, total tokens over the wall time of the burst, latency over all requests
- significance: the test result (Wilcoxon signed-rank p, Holm-corrected) is the ground truth of whether anything changed and the --compare table always shows it; the floors (--floor-decode 5% / --floor-prefill 2%) only judge size, never the test. With 5 runs the prompt tests hold 5 pairs and the chats 5x3, and no metric can be significant below n>=8 pairs

## Results

| Configuration | Actual prompt tokens | Reused tokens | Generated tokens | Runs | Prompt tok/s median and range | Decode tok/s median and range | TTFT seconds median and range |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| short chat | 46 (43-51) | 0 | 100 (100-100) | 15 | not measured | 48.7 (39.2-61.6) | 0.66 (0.50-0.89) |
| 4K prompt, fresh | 4019 (4019-4019) | 0 (0-0) | 24 cap | 5 | 1145.2 (1127.3-1146.1) | not measured | 3.54 (3.54-3.60) |
| 16K prompt, fresh | 16058 (16058-16058) | 0 (0-0) | 24 cap | 5 | 1499.8 (1498.0-1514.3) | not measured | 10.75 (10.65-10.76) |
| agent session, follow-up turns | 14919 (14916-14930) at the last turn | see below | up to 64 | 5 | see below | 73.0 (57.9-84.7) | 3.45 (2.13-4.09) |

Ranges are the lowest and highest of the runs; `Runs` counts requests that returned a token.

### Agent session (conversation cache)

Cache hit over the follow-up turns: 70.8 % (70.8-70.8); earlier-turn tokens read again per session: 5 (5-5) (earlier turns that the cache did not hold; the 14068 (14068-14068) new tokens of the files are read once); first turn (nothing cached) first token: 2.25 s (1.86-2.84). Corpus: `repo_docs` (ea6b8b1b2640) [README.md, docs/HOW_IT_WORKS.md, docs/DETAILS.md, docs/INSTALL.md, docs/MODELS.md].

| Turn (run 1) | Prompt tokens | Reused | Read | Prompt tok/s | First token (s) | Decode tok/s |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | 609 | 0 | 609 | 333.0 | 1.862 | 43.9 |
| 1 | 4995 | 621 | 4374 | 1082.4 | 4.087 | 65.3 |
| 2 | 6531 | 5058 | 1473 | 670.5 | 2.234 | 66.0 |
| 3 | 9801 | 6594 | 3207 | 837.7 | 3.873 | 75.6 |
| 4 | 11887 | 9854 | 2033 | 805.9 | 2.562 | 64.5 |
| 5 | 14916 | 11930 | 2986 | 866.9 | 3.488 | 57.9 |

Per-request data: `results.json`. Engine log lines from the start: `engine-log-excerpt.txt` (when this run started the server).

## Correctness and limitations

Not measured: this is a speed suite. The conversation uses the model's own replies, so token counts differ a little between runs. Machine details come from the server, `nvidia-smi`/`rocm-smi` and the OS; background load is not recorded. Total time of the suite: 235 s.
