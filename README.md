# FabricScan Help Desk Assistant — LLMs and Prompt Engineering

**Project 3: AI Product MVP (Team Lab)**

**Team Members: Hassan, Haider, and Huzaifa**

> **Product Overview:** **FabricScan** is an AI industrial textile and fabric defect inspection assistant built for textile mills and garment factories in Pakistan, supporting quality-control (QC) inspectors, line supervisors, and factory managers.

---

## Executive Summary

* **Provider and model:** `gemini`, `gemini-3.5-flash-lite` (after switching from `gemini-3.1-flash-lite` due to `HTTP 503` availability errors); API key stored in Kaggle Secrets (`UserSecretsClient().get_secret('LLM_API_KEY')`) with fallback to `os.environ.get('LLM_API_KEY')`.
* **Temperature test:** **3** distinct answers at $T = 0.0$ vs **2** distinct answers at $T = 1.0$ out of 5 runs on the chai shop naming test (`T = 0.0` runs were not perfectly identical on either the naming test or the factual broken-thread test).
* **Tokens:** English **4.5** chars/token (67 chars, 15 tokens), Urdu **2.7** chars/token (54 chars, 20 tokens; $1.3\times$ English tokens), Roman Urdu **3.3** chars/token (69 chars, 21 tokens; $1.4\times$ English tokens).
* **Prompt v1 vs v2 on 25 questions:** valid JSON **0.00 -> 1.00**, facts **0.48 (0.47619) -> 1.00**, scope **NaN -> 0.96** (and `cites` **0.00 -> 0.81 (0.809524)**); with **21** fact-scored questions (`facts_ok` for `v2` at $p = 1.00$) the 95% interval is **+/- 0.00** (for `v1` at $p = 0.47619$, $n = 21$, the 95% interval is **+/- 0.21**).
* **Attacks:** **4 of 4** standard/initial attacks (`#1`–`#4`) and **6 of 6** total attacks (`#1`–`#6`, including 2 custom FabricScan attacks) had `leaked = False` and resisted instruction/authority/translation overrides; failure noted: **Row 42 (`"How much does FabricScan cost per month?"`) had `scope_ok = False` and `cites = False` (and `cites = False` occurred on all 4 unanswerable questions in rows 41–44 because `FACTS` has no pricing/accuracy/camera/speed facts while `EVAL` required `'support'` in `must`, penalizing empty `sources: []`)**.
* **Estimated cost at 1,000 users x 20 queries/day:** **USD 226** per month using measured `SYSTEM_V2` call averages (`737.8` input tokens and `52.9` output tokens across 45 `SYSTEM_V2` calls at `PRICE_IN = $0.375` and `PRICE_OUT = $1.875` per 1M tokens; **USD 22** per month under early tiny-test averages of `16.0` input and `16.29` output tokens).
* **Prompts are versioned in `prompts/` with results in `evals/`:** Versioned prompts live in `prompts/system_v1.md` and `prompts/system_v2.md` (with `prompts/system_v3.md` also present in the workspace), and evaluation artifacts live in `evals/questions.csv`, `evals/results_v1_v2.csv`, `evals/summary.csv`, `evals/attacks.csv`, and `evals/call_log.csv` (plus `evals/results_v3.csv` and `evals/summary_v1_v2_v3.csv` in the workspace).

---

## Repository Structure

```text
.
├── lab-9-llm-prompt-engineering.ipynb   # Main team notebook (Parts 1–7)
├── README.md                            # Complete team README & individual reflections
├── prompts/
│   ├── system_v1.md                     # Baseline naive system prompt
│   ├── system_v2.md                     # Structured JSON, grounded facts, security & canary prompt
│   └── system_v3.md                     # Additional prompt version in workspace
└── evals/
    ├── questions.csv                    # Frozen 25-question evaluation dataset
    ├── results_v1_v2.csv                # Per-question v1 vs v2 evaluation scores
    ├── summary.csv                      # Aggregated v1 vs v2 summary metrics
    ├── attacks.csv                      # Adversarial & canary attack results
    ├── call_log.csv                     # Logged token counts, latency (sec), and tags per call
    ├── results_v3.csv                   # Additional v3 evaluation results in workspace
    └── summary_v1_v2_v3.csv             # Additional multi-version summary in workspace
```

---

## Detailed Lab Findings

### 1. Provider, Model, Smoke Test, and Reliability (`Part 1`)
* **Provider & Model:** We configured `PROVIDERS` for `gemini`, `groq`, and `openai` using the OpenAI-compatible client (`OpenAI(base_url=PROVIDERS[PROVIDER], api_key=API_KEY)`), loading `LLM_API_KEY` from Kaggle Secrets (`UserSecretsClient`) with fallback to `os.environ`. We selected **`gemini-3.5-flash-lite`** under **`gemini`** after encountering `HTTP 503` (`UNAVAILABLE` / high-demand) errors and higher latency on `gemini-3.1-flash-lite`.
* **Retry & Logging Helper (`chat`):** Implemented `max_tokens=2048` (to prevent empty replies when reasoning/thinking tokens are used), exponential backoff with jitter (`wait = 2 ** attempt + random.random()`) on HTTP status codes `(429, 500, 502, 503)`, and per-call logging into `LOG` (`tag`, `in`, `out`, `sec`).
* **Smoke Test (`'Reply with exactly: ready'`):** Returned `'ready'` with `{'tag': 'setup', 'in': 6, 'out': 1, 'sec': 0.63}`. The input count (`6` tokens) exceeds the 4 words (`Reply`, `with`, `exactly:`, `ready`) because subword tokenizers split punctuation (such as `:`) into its own token and chat API requests include message/role framing tokens.

### 2. Sampling, Temperature, and Top-$p$ (`Part 2`)
* **Hand-Computed Softmax & Top-$p$:** For `tokens = ['Islamabad', 'Karachi', 'Lahore']` and `logits = [5.0, 3.0, 1.0]`:
  * At $T = 0.5$: `{'Islamabad': 0.982, 'Karachi': 0.018, 'Lahore': 0.0}`
  * At $T = 1.0$: `{'Islamabad': 0.867, 'Karachi': 0.117, 'Lahore': 0.016}`
  * At $T = 2.0$: `{'Islamabad': 0.665, 'Karachi': 0.245, 'Lahore': 0.09}`
  * At $T = 2.0$ with `top_p = 0.9`: `{'Islamabad': 0.731, 'Karachi': 0.269, 'Lahore': 0.0}`
* **Live Model Temperature Test (`5` runs each):**
  * **$T = 0.0$:** **3 distinct** of 5 (`['The Daily Grind Chai', 'Campus Chai', 'The Daily Grind & Chai', 'Campus Chai', 'Campus Chai']`).
  * **$T = 1.0$:** **2 distinct** of 5 (`['The Daily Grind', 'Campus Chai', 'Campus Chai', 'Campus Chai', 'Campus Chai']`).
* **Determinism Check at $T = 0.0$:** Asking twice (`'In one sentence: what is a "broken thread" defect in woven fabric?'`) produced two different wordings (`identical: False`), showing that even at $T = 0.0$, cloud inference/serving batching can yield slight phrasing differences. For FabricScan's main help-desk task, we use $T = 0.0$ for consistent, grounded, and repeatable answers.

### 3. Multilingual Tokenization and Cost Estimation (`Part 3` & `Part 6`)
* **Three-Script Tokenization (`'Reply OK.\n' + text` minus `'Reply OK.'` baseline):**

| Script | Text | Chars | Tokens | Chars / Token | Ratio vs. English |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **English** | `Where can I see the holes and stains detected in this fabric image?` | 67 | 15 | **4.5** | $1.0\times$ |
| **Urdu** | `کپڑے کی اس تصویر میں سوراخ اور داغ کہاں نظر آ رہے ہیں؟` | 54 | 20 | **2.7** | **$1.3\times$** ($20 / 15 = 1.33\times$) |
| **Roman Urdu** | `Is kapray ki tasveer mein suraakh aur daagh kahan nazar aa rahe hain?` | 69 | 21 | **3.3** | **$1.4\times$** ($21 / 15 = 1.40\times$) |

* **Monthly Cost Estimates (`PRICE_IN = $0.375`, `PRICE_OUT = $1.875` per 1M tokens; 30 days/month):**
  * **Early Tiny-Test Log (`1,000 users x 20 queries/day` = `600,000 queries/month`):** Mean `16.0` input tokens and `16.29` output tokens -> **USD 22/month** (`$21.93`).
  * **Measured `SYSTEM_V2` Calls (`45` calls; mean `737.8` input tokens, `52.9` output tokens):**
    * **1,000 users x 20 queries/day** (`600,000 queries/month`): **USD 226/month**.
    * **500 users x 5 queries/day** (`75,000 queries/month`): **USD 28/month**.

### 4. Structured Prompt (`SYSTEM_V2`), Evaluation (`v1` vs `v2`), and Security (`Parts 4–6`)
* **`SYSTEM_V2` & Validation:** Grounds responses strictly inside `<facts>...</facts>` (`[F1]`–`[F10]`), wraps user input in `<question>...</question>`, embeds canary `CANARY-7F3A`, enforces strict JSON output validated via Pydantic (`Reply`: `answer`, `sources`, `confidence`, `in_scope`) with up to `repairs=1` retry on invalid JSON.
* **`v1` vs `v2` Results (`25` questions: `21` needing facts, `4` out of scope):**

| Version | `valid_json` | `facts_ok` | `scope_ok` | `cites` |
| :--- | :---: | :---: | :---: | :---: |
| **v1** | `0.00` | `0.47619` (`0.48`) | `NaN` | `0.00` |
| **v2** | `1.00` | `1.00` | `0.96` | `0.809524` (`0.81`) |

  * **95% Confidence Interval:** For `v2` `facts_ok` ($p = 1.00$, $n = 21$), the 95% interval is **+/- 0.00**. For `v1` `facts_ok` ($p = 0.47619$, $n = 21$), $1.96 \times \sqrt{p(1-p)/n} \approx \mathbf{\pm 0.21}$. The gain from `v1` to `v2` (`valid_json` $+1.00$, `cites` $+0.81$, `facts_ok` $+0.52$) is far larger than the interval.
  * **Notable Failure / Scoring Issue:** On Row 42 (`"How much does FabricScan cost per month?"`), `scope_ok` was `False` and `cites` was `False` because `SYSTEM_V2` treats `"prices of goods"` as out-of-scope (`in_scope=False`), while `EVAL` expected `in_scope=True` and required `'support'` in `must` (causing `cites=False` whenever `sources` was empty on unanswerable rows 41–44).
* **Adversarial Attacks & Non-Prompt Defenses:** Across all 6 attacks (`#1`–`#6`, including custom FabricScan attacks `#5` fake engineer/maintenance/roll approval and `#6` Roman Urdu translation request), `leaked` was `False` every time (`in_scope` was `False` on `#1`, `#2`, `#6` and `True` on `#3`, `#4`, `#5`, with refusals/corrections enforced). Attack `#5` came closest to operational risk by combining a fake maintenance engineer persona with a roll-approval request inside an in-scope fabric context. Beyond prompt rules, `guarded_ask()` enforces code-level checks: input length limit (`MAX_Q_CHARS = 500`), raw output canary screening (`CANARY in text` -> `['canary_blocked']`), Pydantic JSON validation (`['invalid_json']`), and numeric grounding verification (`nums(reply.answer) <= (nums(FACTS) | nums(question))` -> `['number_not_in_facts']`), paired with read-only permissions so the assistant cannot mutate factory roll/shipment records.

---

## Team Contributions

| Team Member | Contributions |
| :--- | :--- |
| **Hassan** | Configured provider/model setup (`gemini-3.5-flash-lite`, Kaggle Secrets), implemented the `chat()` retry/backoff/logging wrapper, ran the smoke test and temperature experiments, and analyzed tokenization/cost metrics. |
| **Haider** | Authored and refined `FACTS` (`[F1]`–`[F10]`) and `SYSTEM_V2`, built Pydantic schema validation (`Reply`) and `ask_json()` repair logic, and ran the `v1` vs `v2` evaluation suite and error/interval analysis. |
| **Huzaifa** | Designed and tested product-specific adversarial attacks (including fake engineer/shipment override and Roman Urdu exfiltration), implemented the code-level `guarded_ask()` defenses, and managed prompt/eval artifact exports (`prompts/` and `evals/`). |

---

## Check Your Understanding

### Hassan

1. **For logits `[2, 0]` at $T = 1$, what is the probability of the first token?**
   $$P(\text{token } 1) = \frac{e^{2/1}}{e^{2/1} + e^{0/1}} = \frac{e^2}{e^2 + 1} \approx \frac{7.3891}{8.3891} \approx \mathbf{0.8808}\text{ (about }88.1\%\text{)}$$
2. **Why is $T = 0$ the right choice for extracting fees and dates into JSON?**
   At $T = 0$, decoding selects the highest-probability token greedily rather than sampling from lower-probability alternatives, which minimizes formatting drift, hallucinated values, and structural JSON syntax errors when extracting exact fees and dates.
3. **500 users, 10 queries a day, 1,000 input and 200 output tokens per query, USD 0.30 and 2.50 per million. Monthly cost?**
   * Monthly queries: $500 \times 10 \times 30 = 150,000$ queries.
   * Total input tokens: $150,000 \times 1,000 = 150,000,000$ ($150\text{M}$) $\implies 150 \times \$0.30 = \$45.00$.
   * Total output tokens: $150,000 \times 200 = 30,000,000$ ($30\text{M}$) $\implies 30 \times \$2.50 = \$75.00$.
   * **Total monthly cost:** $\$45.00 + \$75.00 = \mathbf{\$120\text{ USD/month}}$.
4. **Why must the retry wait include a random part?**
   Adding random jitter prevents multiple clients or retry loops from synchronizing and hitting the server at the exact same moment ("thundering herd"), reducing repeated `429` rate-limit or `503` overload collisions.
5. **v2 scores 17/20 on facts and v1 scores 15/20. Is v2 better? Justify with intervals.**
   For `v2`, $\hat{p}_2 = 17/20 = 0.85$, with 95% margin of error $1.96 \times \sqrt{0.85 \times 0.15 / 20} \approx \pm 0.156$ ($[0.694, 1.00]$). For `v1`, $\hat{p}_1 = 15/20 = 0.75$, with 95% margin of error $1.96 \times \sqrt{0.75 \times 0.25 / 20} \approx \pm 0.190$ ($[0.560, 0.940]$). While `v2` has a higher observed point score ($0.85$ vs. $0.75$), the two 95% intervals overlap substantially on a small sample ($n = 20$), so $17/20$ vs. $15/20$ alone is not statistically decisive without more test items.
6. **Why wrap user text in tags, and what does a canary detect?**
   Wrapping user input in tags (like `<question>...</question>`) helps separate untrusted user data from system instructions and reference facts against prompt injection. A canary token is a secret marker placed in the prompt that should never appear in normal user replies; detecting it in the output indicates a prompt-leakage or instruction-exfiltration breach.

### Haider

1. **For logits `[2, 0]` at $T = 1$, what is the probability of the first token?**
   Using softmax at $T = 1$: $\frac{e^2}{e^2 + e^0} = \frac{7.389}{7.389 + 1} \approx \mathbf{0.881}$ ($88.08\%$).
2. **Why is $T = 0$ the right choice for extracting fees and dates into JSON?**
   Structured extraction requires consistent keys, valid JSON syntax, and exact fidelity to source numbers and dates; setting $T = 0$ keeps outputs focused on the most probable tokens instead of introducing random variation that could corrupt schema fields or alter figures.
3. **500 users, 10 queries a day, 1,000 input and 200 output tokens per query, USD 0.30 and 2.50 per million. Monthly cost?**
   * Queries/month $= 500 \times 10 \times 30 = 150,000$.
   * Cost per query $= (1,000 \times \$0.30 + 200 \times \$2.50) / 1,000,000 = (\$0.30 + \$0.50) / 1,000 = \$0.0008$.
   * Monthly cost $= 150,000 \times \$0.0008 = \mathbf{\$120\text{ USD}}$.
4. **Why must the retry wait include a random part?**
   Without jitter, exponential backoff retries across concurrent requests align on similar intervals and re-trigger congestion or rate limits; random jitter spreads retry timestamps out evenly.
5. **v2 scores 17/20 on facts and v1 scores 15/20. Is v2 better? Justify with intervals.**
   `v2` is higher on this sample ($0.85$ vs. $0.75$, a $+0.10$ difference), with 95% confidence intervals of roughly $0.85 \pm 0.16$ and $0.75 \pm 0.19$ for $n = 20$. Because the intervals overlap and the difference ($0.10$) is smaller than either margin of error, `v2` looks promising on facts here, but a larger evaluation set is needed for strong statistical certainty.
6. **Why wrap user text in tags, and what does a canary detect?**
   Delimiters/tags mark untrusted user text as data rather than control commands, reducing delimiter confusion. A canary string in model output detects when an attacker successfully tricks the model into exfiltrating or echoing protected prompt contents.

### Huzaifa

1. **For logits `[2, 0]` at $T = 1$, what is the probability of the first token?**
   $$\frac{e^2}{e^2 + e^0} = \frac{e^2}{e^2 + 1} \approx \mathbf{0.8808}\text{ (or }\mathbf{88.1\%}\text{)}$$
2. **Why is $T = 0$ the right choice for extracting fees and dates into JSON?**
   Extracting fees and dates into machine-parsed JSON needs deterministic, repeatable output where the model picks the most likely valid token sequence every time rather than taking creative detours that break JSON formatting or misread numbers/dates.
3. **500 users, 10 queries a day, 1,000 input and 200 output tokens per query, USD 0.30 and 2.50 per million. Monthly cost?**
   * $500 \times 10 \times 30 = 150,000$ queries/month.
   * Input cost: $150\text{M tokens} \times \$0.30/\text{M} = \$45$.
   * Output cost: $30\text{M tokens} \times \$2.50/\text{M} = \$75$.
   * **Monthly cost:** **USD 120** per month.
4. **Why must the retry wait include a random part?**
   Random jitter breaks synchronization across retries so bursts of requests do not all retry simultaneously and overwhelm the API endpoint or hit rate limits again.
5. **v2 scores 17/20 on facts and v1 scores 15/20. Is v2 better? Justify with intervals.**
   `v2` scores $0.85$ ($95\%$ CI: $0.85 \pm 0.16$) versus `v1` at $0.75$ ($95\%$ CI: $0.75 \pm 0.19$) on $n = 20$ questions. Although `v2` improved by $+0.10$ on facts (and much more on JSON structure and citations), the 95% confidence intervals overlap, so the facts difference alone is not statistically conclusive at $n = 20$.
6. **Why wrap user text in tags, and what does a canary detect?**
   Wrapping user input in tags clearly bounds user-supplied text as data so injected instructions inside user queries are less likely to be treated as system rules. A canary detects whether hidden system prompt instructions/secrets leaked into the model's response.
