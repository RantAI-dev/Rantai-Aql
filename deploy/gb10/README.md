# GB10 (DGX Spark) profile

Nafs-Aql's reference profile runs both models on four x86 CPU cores with Smiley Core. On the GB10 (ARM64 with a GPU)
the same gateway runs in front of two vLLM GPU servers instead:

| Role | Model | Server |
|---|---|---|
| System 1 (quick draft) | Qwen/Qwen3-1.7B, BF16 | `aql-s1:8001`, thinking off, no reasoning parser |
| System 2 (reasoning) | Qwen/Qwen3-4B-Thinking-2507, BF16 | `aql-s2:8002`, `--reasoning-parser qwen3` |
| Gateway | `nafs-gateway` built for ARM64 from `deploy/docker/Dockerfile.gateway` | port 8096, bearer token |

Configuration: `config/rantai-aql.gb10.toml` (the docker configuration with the two backend addresses and `python3` for
SymPy). Knowledge base, embeddings and ShadowPrefill are off on this profile.

## First run on the box (16 Nov 2026)

| Question | Path | Assurance | Time | Correct |
|---|---|---|---|---|
| Capital of Indonesia (English and Indonesian) | System 1 | unverified | 0.4 s | yes |
| 17 × 23 + 144 / 12 | System 1, three agreeing samples | consensus | 3.5 s | yes (403) |
| Train 245 km at 70 km/h from 09:40 | System 1 | unverified | 4.8 s | yes (13:10) |
| How many n < 100 make n² + n + 41 composite | escalated to System 2 | consensus | 86 s | **no** (60; the answer is 14) |
| Python `is_prime` | System 1 | unverified | 4.7 s | not checked |

The last row is the documented limit of agreement: samples that agree on a wrong answer still read as consensus.

## Benchmarks on the GB10 (16 Nov 2026)

Stored runs: `benchmarks/gb10_gsm8k60_three_systems.json`, `benchmarks/gb10_regression120_fast_only.json`.

**GSM8K, 60 questions (seed 0, shuffled), three strategies on the same questions:**

| Strategy | Correct | Median | Mean | Total | Escalated | Reasoning tokens |
|---|---|---|---|---|---|---|
| `fast_only` (System 1) | 49/60 | 2.4 s | 3.0 s | 3.0 min | 0 | 0 |
| `nafs_aql` (gate) | **59/60** | 2.6 s | 12.0 s | **12.0 min** | 11 | 9,764 |
| `slow_only` (System 2) | 59/60 | 28.8 s | 32.6 s | 32.6 min | 60 | 34,551 |

The gate matches always-thinking accuracy at **37 % of its time** (28 % of its reasoning tokens), the same ratio the
README reports on the CPU reference (36 %).

**Regression set, 120 items, System 1 only, against the stored CPU run** (`benchmarks/rollback_drill/candidate_run_3cca7cad.json`):

| Machine | Correct | Median per item | Total |
|---|---|---|---|
| i3-8100, 4 cores, Smiley Core GGUF Q4_K_M | 85/120 | 36.5 s | 77.1 min |
| GB10 GPU, vLLM BF16 | 89/120 | 2.5 s | 5.7 min |

About **14×** faster at the same accuracy (the GPU runs BF16 weights, the CPU run Q4_K_M).

## Notes

- **ShadowPrefill on vLLM.** The gateway now sends shadows to vLLM backends too (`Backend::supports_shadow`): a
  one-token request whose prompt lands in vLLM's prefix cache, rendered without a generation prompt when it ends with
  the assistant's reply, so the next turn's prompt starts with exactly the cached tokens. vLLM has no priority classes,
  so there is no duty cap; on the GPU a prefill costs little. `--enable-prompt-tokens-details` lets the shadow report
  how much was already cached.
- **Smiley Core on ARM.** `smiley-core` has AVX2 kernels only (`simd.rs` is `x86_64`-only); on ARM it compiles to the
  scalar path, which its own documentation measures as compute-bound. A NEON port is needed before the CPU profile and
  ShadowPrefill can run here.
- **vLLM nightly streaming bug.** With `--reasoning-parser qwen3`, a streamed reply with `enable_thinking: false`
  arrives entirely in `reasoning`, so the gateway's streamed System 1 draft was empty. Non-streamed replies are
  correct. Removing the parser from System 1 fixes it.
