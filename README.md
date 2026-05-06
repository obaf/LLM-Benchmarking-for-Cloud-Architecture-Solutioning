# AWS Advanced Networking MCQ Benchmark with Promptfoo

A reproducible benchmark that uses [Promptfoo](https://www.promptfoo.dev/) to
evaluate three LLMs on AWS Advanced Networking Specialty–style multiple-choice
questions (MCQs). All three models are accessed through
[OpenRouter](https://openrouter.ai/) so a single API key is enough.

The benchmark currently runs **15 questions × 3 providers = 45 test cases** (this is
where the `45t` in the result file name comes from).

---

## What's in this folder

| File | Purpose |
| --- | --- |
| `promptfooconfig.yaml` | Promptfoo config — defines providers, prompts, tests, and the grading assertion. |
| `prompt.txt` | The prompt template that wraps each question. Forces the model to answer with a single letter `A`–`D`. |
| `aws_promptfoo_questions.yaml` | Question bank. Each entry has `id`, `domain`, `prompt` (full question text), `choices`, and the correct `answer`. |
| `eval-45t-2026-05-06T07_35_04.csv` | Latest evaluation export from Promptfoo — one row per question, one column-group per model. |
| `.env` | Local secrets (not committed). Must contain `OPENROUTER_API_KEY=...`. |

---

## Models under test

All three are pulled from OpenRouter with `temperature: 0` and `max_tokens: 128`.

| Provider id (OpenRouter) | Short name used below |
| --- | --- |
| `openrouter:openai/gpt-5.4-mini` | GPT-5.4-mini |
| `openrouter:google/gemini-3-flash-preview` | Gemini 3 Flash (preview) |
| `openrouter:deepseek/deepseek-chat-v3.1` | DeepSeek Chat v3.1 |

`max_tokens: 128` is intentionally generous because Gemini 3 Flash emits a long
"thinking" trace before producing the final letter — the grading logic compensates
for that (see below).

---

## How the prompt is built

Every test case fills two variables — `{{prompt}}` and `{{choices}}` — into
`prompt.txt`:

```1:8:prompt.txt
Answer the following multiple choice question.
Return ONLY a single letter: A, B, C, or D.
Do NOT include any explanation, punctuation, or extra text.
Your entire response must be exactly one character.

{{prompt}}

{{choices}}
```

So the model receives the full question (with the four options inlined inside
`prompt`), then a separate `A\nB\nC\nD` enumeration as `choices`.

---

## How answers are graded

`promptfooconfig.yaml` declares a single JavaScript assertion that runs against
the model output:

```25:33:promptfooconfig.yaml
defaultTest:
  assert:
    - type: javascript
      value: |
        // Gemini thinks for a long time then outputs the letter last
        // Extract the LAST A-D letter in the output
        const matches = output.match(/[A-D]/g);
        const letter = matches ? matches[matches.length - 1] : null;
        return letter === context.vars.answer;
```

The grader extracts **the last `A`/`B`/`C`/`D` character** in the response and
compares it to `context.vars.answer`. This is what allows Gemini's verbose
chain-of-thought output (which can mention several letters along the way) to be
graded fairly — only its final letter counts.

---

## Question set

`aws_promptfoo_questions.yaml` contains 15 ANS-C01-style scenarios in the
`network` domain (Direct Connect, Transit Gateway, PrivateLink, NLB/ALB, Route 53
Resolver DNS Firewall, etc.). Each entry looks like:

```1:6:aws_promptfoo_questions.yaml
- vars:
    id: q_190
    domain: network
    prompt: "Question #190 Topic 1: A company is building an API-based application on AWS ..."
    choices: "A\nB\nC\nD"
    answer: A
```

Question IDs in the current run: `q_190, q_192, q_194, q_195, q_196, q_197,
q_198, q_200, q_201, q_202, q_203, q_204, q_205, q_206, q_211`.

---

## How to run it

### 1. Prerequisites

- Node.js 18+
- An OpenRouter account + API key
- `npm install -g promptfoo` (or run via `npx`)

### 2. Configure secrets

Create a `.env` file in the project root:

```bash
OPENROUTER_API_KEY=sk-or-...
```

### 3. Run the evaluation

```bash
promptfoo eval -c promptfooconfig.yaml
```

### 4. Inspect results

```bash
promptfoo view            # interactive web UI
promptfoo eval -o results.csv   # export to CSV (same shape as the file in this repo)
```

---

## Results — `eval-45t-2026-05-06T07_35_04.csv`

The CSV is a wide format produced by Promptfoo: the first 5 columns describe the
test case (`id, domain, prompt, choices, answer`), then each model contributes a
6-column block (`<model output>, Status, Score, Named Scores, Grader Reason,
Comment`).

### Per-model accuracy (15 questions)

| Model | Correct | Accuracy |
| --- | ---: | ---: |
| Gemini 3 Flash (preview) | 11 / 15 | **73 %** |
| DeepSeek Chat v3.1 | 8 / 15 | 53 % |
| GPT-5.4-mini | 5 / 15 | 33 % |

### Per-question outcome

`✓` = correct, `✗` = wrong (extracted letter shown in parentheses).
Expected answers come from the question bank.

| Question | Expected | GPT-5.4-mini | Gemini 3 Flash | DeepSeek v3.1 |
| --- | :---: | :---: | :---: | :---: |
| q_190 | A | ✓ (A) | ✓ (A) | ✗ (B) |
| q_192 | B | ✗ (D) | ✗ (D) | ✗ (A) |
| q_194 | A | ✗ (C) | ✓ (A) | ✓ (A) |
| q_195 | B | ✗ (D) | ✓ (B) | ✓ (B) |
| q_196 | B | ✗ (C) | ✓ (B) | ✓ (B) |
| q_197 | C | ✗ (A) | ✗ (B) | ✗ (D) |
| q_198 | B | ✗ (A) | ✓ (B) | ✗ (D) |
| q_200 | A | ✗ (B) | ✓ (A) | ✗ (D) |
| q_201 | D | ✗ (A) | ✗ (A) | ✓ (D) |
| q_202 | B | ✗ (A) | ✓ (B) | ✗ (A) |
| q_203 | D | ✓ (D) | ✓ (D) | ✓ (D) |
| q_204 | B | ✓ (B) | ✓ (B) | ✓ (B) |
| q_205 | B | ✓ (B) | ✓ (B) | ✓ (B) |
| q_206 | B | ✗ (C) | ✗ (C) | ✗ (A) |
| q_211 | C | ✓ (C) | ✓ (C) | ✓ (C) |

### Observations

- **Unanimous successes (4):** `q_203`, `q_204`, `q_205`, `q_211` — all three
  models agree on the correct answer.
- **Unanimous failures (3):** `q_192`, `q_197`, `q_206` — every model gets these
  wrong, suggesting either genuinely hard scenarios or potentially ambiguous /
  truncated question text in the bank (some entries in
  `aws_promptfoo_questions.yaml` are abridged with `...`).
- **Gemini vs GPT divergence:** Gemini gets 6 questions right that GPT-5.4-mini
  misses (`q_194, q_195, q_196, q_198, q_200, q_202`), with no questions going
  the other way. This is consistent with Gemini's longer reasoning trace
  benefiting these scenario-style network questions.
- **DeepSeek is in the middle**, sharing most of Gemini's wins on the simpler
  items but joining GPT on several of the harder ones.

---

## Reproducing or extending the benchmark

- **Add questions:** append more entries to `aws_promptfoo_questions.yaml` using
  the same shape (`id`, `domain`, `prompt`, `choices`, `answer`). The output
  filename `eval-Nt-...csv` reflects the total number of test cases
  (`questions × providers`).
- **Add or swap models:** edit the `providers:` block in
  `promptfooconfig.yaml`. Any OpenRouter-hosted model id works; just keep
  `temperature: 0` for deterministic comparisons and consider `max_tokens` if the
  model emits internal reasoning before the final letter.
- **Tighten grading:** the current grader takes the *last* `A–D` letter found.
  For models that reliably answer with a single character you can switch to a
  stricter `equals` or `regex` assertion. The current logic is intentionally
  permissive so that "thinking" models aren't penalized.

---

## License / data note

The questions are paraphrased AWS networking scenarios used purely for LLM
evaluation. Several entries in this snapshot are intentionally truncated
(`...`) — re-expand them with the full prompts before drawing strong
conclusions about absolute model accuracy.
