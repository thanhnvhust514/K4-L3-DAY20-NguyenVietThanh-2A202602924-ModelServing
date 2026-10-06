# Quality comparison: Qwen3.5 0.8B

Real local calls. Same prompts and settings for both quantizations. This is a small diagnostic sample, not a general quality benchmark.

Settings: {"threads": 8, "ngl": 99, "ctx": 2048, "max_tokens": 160, "temperature": 0, "seed": 42, "reasoning": "off", "port": 8097}

## definitions

Prompt:

Context: TTFT means Time To First Token: elapsed time from sending a request to receiving its first output token. TPOT means Time Per Output Token: average time per output token after the first. Using only this context, define TTFT and TPOT in two concise sentences. Do not invent alternative expansions.

Expected: Correct expansions and timing definitions for both TTFT and TPOT.

### Q4_K_M

TTFT is the elapsed time from sending a request to receiving its first output token, while TPOT is the average time per output token after the first.

Finish reason: stop

### UD-Q2_K_XL

TTFT is the elapsed time from sending a request to receiving its first output token, while TPOT is the average time per output token after the first.

Finish reason: stop

## arithmetic

Prompt:

Return only the integer result of 17 * 23 + 19.

Expected: 410

### Q4_K_M

501

Finish reason: stop

### UD-Q2_K_XL

17 * 23 + 19 = 39 + 19 = 58

Finish reason: stop

## json

Prompt:

Extract this record into JSON. Return only a JSON object with exactly keys name, model, threads. Record: Name: Thanh. Model: Qwen3.5 0.8B. Threads: 8. threads must be an integer.

Expected: {'name': 'Thanh', 'model': 'Qwen3.5 0.8B', 'threads': 8}

### Q4_K_M

{
  "name": "Thanh",
  "model": "Qwen3.5 0.8B",
  "threads": 8
}

Finish reason: stop

### UD-Q2_K_XL

{"name": "Thanh", "model": "Qwen3.5 0.8B", "threads": 8}

Finish reason: stop

## Checked outcomes

| Check | Q4_K_M | UD-Q2_K_XL |
|---|---|---|
| TTFT/TPOT timing meanings | Correct meanings, combined into one sentence | Correct meanings, combined into one sentence |
| 17 * 23 + 19 (expected 410) | Incorrect: 501 | Incorrect: 58 |
| JSON fields and integer threads | Correct | Correct |

Three prompts do not establish general quality equivalence. Arithmetic failures occurred in both quantizations.
