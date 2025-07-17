+++
title = 'LLM Inference Engines Performance Comparison: vLLM vs sglang'
date = 2025-07-17T11:22:45+08:00
draft = false
+++

# Benchmark Comparison Report

This report compares the performance of three inference engines (vLLM V0, vLLM V1, and sglang V0.4.9) on the Llama-3.1-8B-Instruct model. All tests generated 634,000 tokens, ensuring horizontal comparability of the data.

## Testing Environment

- **Model**: Llama-3.1-8B-Instruct
- **Hardware**: 1 NVIDIA H100 80GB GPU
- **Command**: `evalscope perf --parallel 5 10  20  30 40 50 60 70 80 90 100  --number 5 10  20  30 40 50 60 70 80 90 100    --model meta-llama/Llama-3.1-8B-Instruct  --url <URL>   --api openai  --dataset longalpaca  --max-tokens 2000   --min-tokens 2000 --max-prompt-length 10000 --min-prompt-length 5000 `
- vLLM V0 and V1 versions: v0.9.0
- sglang version: v0.4.9

---

## I. Basic Performance Comparison

| Inference Engine  | Total Time (sec) | Avg Output Rate (tokens/sec) | Performance Improvement (vs vLLM V0) |
|-------------------|------------------|------------------------------|-------------------------------------|
| **vLLM V0**       | 268.79           | 2358.71                      | Baseline                            |
| **vLLM V1**       | 236.00           | 2686.49                      | +14% (rate improvement)             |
| **sglang V0.4.9** | 236.95           | 2675.66                      | +13% (rate improvement)             |

- **vLLM V1** and **sglang V0.4.9** perform similarly, both outperforming **vLLM V0**.
- **vLLM V1** is slightly faster than **sglang**, but the difference is minimal.

---

## II. Detailed Performance Metrics

### 1. Average Latency

| Concurrency | vLLM V0 Avg Lat(s) | vLLM V1 Avg Lat(s) | sglang Avg Lat(s) |
|-------------|--------------------|--------------------|-------------------|
| 5           | 16.98              | 15.75              | 15.00             |
| 50          | 26.18              | 22.80              | 23.38             |
| 100         | 26.16              | 23.59              | 23.29             |

- **Low concurrency**: sglang has the lowest latency, providing the fastest response.
- **High concurrency**: vLLM V1 has slightly lower latency, performing marginally better.

### 2. Average Generation Rate (Gen. toks/sec)

| Concurrency | vLLM V0 Gen. toks/s | vLLM V1 Gen. toks/s | sglang Gen. toks/s |
|-------------|---------------------|---------------------|--------------------|
| 5           | 588.62              | 634.87              | 666.54             |
| 50          | 2742.96             | 3141.16             | 3077.68            |
| 100         | 2744.10             | 3036.62             | 3088.08            |

- **High concurrency**: Both vLLM V1 and sglang exceed 3000 tokens/s, significantly outperforming vLLM V0.

### 3. Time to First Token (TTFT)

| Concurrency | vLLM V0 TTFT(s) | vLLM V1 TTFT(s) | sglang TTFT(s) |
|-------------|-----------------|-----------------|----------------|
| 5           | 0.318           | 0.276           | 0.136          |
| 50          | 0.357           | 0.348           | 0.258          |
| 100         | 0.415           | 0.373           | 0.254          |

- sglang demonstrates the best TTFT performance across all concurrency scenarios, with significantly faster response times.

### 4. Throughput (RPS & Concurrency Capability)

| Inference Engine | Max Throughput (RPS) | Optimal Concurrency | Recommended Range |
|------------------|----------------------|--------------------|-------------------|
| **vLLM V0**      | 1.37 req/sec         | 50                 | ~50               |
| **vLLM V1**      | 1.57 req/sec         | 40                 | ~40               |
| **sglang**       | 1.55 req/sec         | 60                 | ~60               |

- **vLLM V1** and **sglang** lead in throughput, recommended at concurrency levels of 40 and 60 respectively.

---

## III. Best Performance Configurations

| Inference Engine | Lowest Latency (s) | Optimal Concurrency | Generation Rate (tokens/sec) |
|------------------|--------------------|--------------------|------------------------------|
| **vLLM V0**      | 16.98              | 5                  | 588.62                      |
| **vLLM V1**      | 15.75              | 5                  | 634.87                      |
| **sglang**       | 15.00              | 5                  | 666.54                      |

- For low-latency scenarios, **sglang** performs best.

---

## IV. Summary and Recommendations

1. **Performance Improvement**:
   - vLLM V1 and sglang show significant improvements over vLLM V0 in generation rate and latency, with an overall performance increase of approximately 14%.
   - At high concurrency, vLLM V1 performs slightly better in terms of latency, while sglang excels in TTFT and throughput.

2. **Application Scenarios**:
   - **Low-latency scenarios**: sglang is recommended for its shorter latency and quick response times.
   - **High-throughput scenarios**: Both vLLM V1 and sglang perform excellently; choice depends on specific requirements.

3. **Recommended Configurations**:
   - For high-concurrency requirements, recommended concurrency settings are:
     - vLLM V1: around 40
     - sglang: around 60

4. **Optimization Directions**:
   - Further optimize hardware configurations to test performance under even higher loads.
   - Explore the adaptability of different engines for specific tasks and model scenarios.

---

> **Note**: This report is based on performance testing of the Llama-3.1-8B-Instruct model; performance with other models requires separate testing and validation.

---

## V. Original Benchmark Data

### vLLM V0 v0.9.0

```
Basic Information:
┌───────────────────────┬──────────────────────────────────┐
│ Model                 │ Llama-3.1-8B-Instruct            │
│ Total Generated       │ 634,000.0 tokens                 │
│ Total Test Time       │ 268.79 seconds                   │
│ Avg Output Rate       │ 2358.71 tokens/sec               │
└───────────────────────┴──────────────────────────────────┘


                                    Detailed Performance Metrics
┏━━━━━━┳━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┓
┃      ┃      ┃      Avg ┃      P99 ┃    Gen. ┃      Avg ┃     P99 ┃      Avg ┃     P99 ┃   Success┃
┃Conc. ┃  RPS ┃  Lat.(s) ┃  Lat.(s) ┃  toks/s ┃  TTFT(s) ┃ TTFT(s) ┃  TPOT(s) ┃ TPOT(s) ┃      Rate┃
┡━━━━━━╇━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━┩
│    5 │ 0.29 │   16.981 │   16.988 │  588.62 │    0.318 │   0.375 │    0.008 │   0.008 │    100.0%│
│   10 │ 0.53 │   18.707 │   18.719 │ 1068.23 │    0.306 │   0.332 │    0.009 │   0.009 │    100.0%│
│   20 │ 0.92 │   21.738 │   21.784 │ 1835.82 │    0.401 │   0.589 │    0.011 │   0.011 │    100.0%│
│   30 │ 1.19 │   25.037 │   25.119 │ 2388.07 │    0.391 │   0.643 │    0.012 │   0.012 │    100.0%│
│   40 │ 1.32 │   27.254 │   27.355 │ 2631.36 │    0.434 │   0.621 │    0.013 │   0.013 │    100.0%│
│   50 │ 1.37 │   26.176 │   26.241 │ 2742.96 │    0.357 │   0.426 │    0.013 │   0.013 │    100.0%│
│   60 │ 1.34 │   26.708 │   26.784 │ 2687.64 │    0.393 │   0.454 │    0.013 │   0.013 │    100.0%│
│   70 │ 1.37 │   26.148 │   26.223 │ 2744.60 │    0.405 │   0.478 │    0.013 │   0.013 │    100.0%│
│   80 │ 1.35 │   26.531 │   26.604 │ 2705.46 │    0.425 │   0.502 │    0.013 │   0.013 │    100.0%│
│   90 │ 1.35 │   26.600 │   26.672 │ 2698.57 │    0.518 │   0.582 │    0.013 │   0.013 │    100.0%│
│  100 │ 1.37 │   26.157 │   26.234 │ 2744.10 │    0.415 │   0.488 │    0.013 │   0.013 │    100.0%│
└──────┴──────┴──────────┴──────────┴─────────┴──────────┴─────────┴──────────┴─────────┴──────────┘


               Best Performance Configuration
 Highest RPS         Concurrency 50 (1.37 req/sec)
 Lowest Latency      Concurrency 5 (16.981 seconds)

Performance Recommendations:
• Optimal concurrency range is around 50
```

### vLLM V1 v0.9.0

```
Basic Information:
┌───────────────────────┬──────────────────────────────────┐
│ Model                 │ Llama-3.1-8B-Instruct            │
│ Total Generated       │ 634,000.0 tokens                 │
│ Total Test Time       │ 236.00 seconds                   │
│ Avg Output Rate       │ 2686.49 tokens/sec               │
└───────────────────────┴──────────────────────────────────┘


                                    Detailed Performance Metrics
┏━━━━━━┳━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┓
┃      ┃      ┃      Avg ┃      P99 ┃    Gen. ┃      Avg ┃     P99 ┃      Avg ┃     P99 ┃   Success┃
┃Conc. ┃  RPS ┃  Lat.(s) ┃  Lat.(s) ┃  toks/s ┃  TTFT(s) ┃ TTFT(s) ┃  TPOT(s) ┃ TPOT(s) ┃      Rate┃
┡━━━━━━╇━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━┩
│    5 │ 0.32 │   15.747 │   15.752 │  634.87 │    0.276 │   0.292 │    0.008 │   0.008 │    100.0%│
│   10 │ 0.58 │   17.082 │   17.096 │ 1169.81 │    0.181 │   0.206 │    0.009 │   0.009 │    100.0%│
│   20 │ 1.03 │   19.303 │   19.342 │ 2067.71 │    0.366 │   0.474 │    0.009 │   0.009 │    100.0%│
│   30 │ 1.36 │   21.902 │   22.004 │ 2726.02 │    0.459 │   0.673 │    0.011 │   0.011 │    100.0%│
│   40 │ 1.57 │   22.797 │   22.898 │ 3143.68 │    0.397 │   0.549 │    0.011 │   0.011 │    100.0%│
│   50 │ 1.57 │   22.803 │   22.911 │ 3141.16 │    0.348 │   0.442 │    0.011 │   0.011 │    100.0%│
│   60 │ 1.57 │   22.872 │   22.979 │ 3132.07 │    0.375 │   0.459 │    0.011 │   0.011 │    100.0%│
│   70 │ 1.56 │   22.962 │   23.070 │ 3120.01 │    0.354 │   0.437 │    0.011 │   0.011 │    100.0%│
│   80 │ 1.55 │   23.071 │   23.185 │ 3104.36 │    0.384 │   0.478 │    0.011 │   0.011 │    100.0%│
│   90 │ 1.57 │   22.866 │   22.984 │ 3130.58 │    0.434 │   0.524 │    0.011 │   0.011 │    100.0%│
│  100 │ 1.52 │   23.590 │   23.702 │ 3036.62 │    0.373 │   0.466 │    0.012 │   0.012 │    100.0%│
└──────┴──────┴──────────┴──────────┴─────────┴──────────┴─────────┴──────────┴─────────┴──────────┘


               Best Performance Configuration
 Highest RPS         Concurrency 40 (1.57 req/sec)
 Lowest Latency      Concurrency 5 (15.747 seconds)

Performance Recommendations:
• Optimal concurrency range is around 40
```

### sglang V0.4.9

```
Basic Information:
┌───────────────────────┬──────────────────────────────────┐
│ Model                 │ Llama-3.1-8B-Instruct            │
│ Total Generated       │ 634,000.0 tokens                 │
│ Total Test Time       │ 236.95 seconds                   │
│ Avg Output Rate       │ 2675.66 tokens/sec               │
└───────────────────────┴──────────────────────────────────┘


                                    Detailed Performance Metrics
┏━━━━━━┳━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━┓
┃      ┃      ┃      Avg ┃      P99 ┃    Gen. ┃      Avg ┃     P99 ┃      Avg ┃     P99 ┃   Success┃
┃Conc. ┃  RPS ┃  Lat.(s) ┃  Lat.(s) ┃  toks/s ┃  TTFT(s) ┃ TTFT(s) ┃  TPOT(s) ┃ TPOT(s) ┃      Rate┃
┡━━━━━━╇━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━┩
│    5 │ 0.33 │   15.000 │   15.003 │  666.54 │    0.136 │   0.143 │    0.007 │   0.007 │    100.0%│
│   10 │ 0.59 │   16.829 │   16.833 │ 1188.16 │    0.154 │   0.164 │    0.008 │   0.008 │    100.0%│
│   20 │ 1.03 │   19.443 │   19.450 │ 2056.25 │    0.213 │   0.236 │    0.010 │   0.010 │    100.0%│
│   30 │ 1.36 │   22.031 │   22.042 │ 2721.65 │    0.236 │   0.273 │    0.011 │   0.011 │    100.0%│
│   40 │ 1.54 │   23.429 │   23.442 │ 3070.85 │    0.303 │   0.506 │    0.012 │   0.012 │    100.0%│
│   50 │ 1.54 │   23.375 │   23.386 │ 3077.68 │    0.258 │   0.322 │    0.012 │   0.012 │    100.0%│
│   60 │ 1.55 │   23.256 │   23.267 │ 3093.28 │    0.263 │   0.328 │    0.011 │   0.011 │    100.0%│
│   70 │ 1.53 │   23.439 │   23.450 │ 3068.73 │    0.273 │   0.404 │    0.012 │   0.012 │    100.0%│
│   80 │ 1.53 │   23.496 │   23.510 │ 3062.05 │    0.305 │   0.522 │    0.012 │   0.012 │    100.0%│
│   90 │ 1.55 │   23.186 │   23.201 │ 3102.28 │    0.266 │   0.325 │    0.011 │   0.011 │    100.0%│
│  100 │ 1.54 │   23.293 │   23.307 │ 3088.08 │    0.254 │   0.302 │    0.011 │   0.012 │    100.0%│
└──────┴──────┴──────────┴──────────┴─────────┴──────────┴─────────┴──────────┴─────────┴──────────┘


               Best Performance Configuration
 Highest RPS         Concurrency 60 (1.55 req/sec)
 Lowest Latency      Concurrency 5 (15.000 seconds)

Performance Recommendations:
• Optimal concurrency range is around 60
```
