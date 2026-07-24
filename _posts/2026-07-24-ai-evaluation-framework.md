---
layout: post
title: "Beyond Chatbots: Building an AI Evaluation Framework That Judges Other AI Models"
date: 2026-07-24
categories: [AI, LLM]
tags: [AI Evaluation, LLM, Prompt Engineering, Benchmarking]
excerpt: "How I stopped comparing AI models by guesswork and built a framework to evaluate them systematically."
---
# Beyond Chatbots: Building an AI Evaluation Framework That Judges Other AI Models

> *How I stopped comparing LLMs by intuition and built a framework to evaluate them systematically.*

---

# The AI Problem Nobody Talks About

Every time a new language model was released, I found myself asking the same question: Which one should I use?

I tried reading benchmarks, comparing responses manually, and watching YouTube reviews, but none of them answered the question for my own use case.

That's when I decided to build a small evaluation framework that could send the same prompt to multiple AI models, measure their performance, and let another LLM judge the quality of their responses.

This blog walks through how I built that framework.

---

## The Problem

Public benchmarks like **MMLU**, **HumanEval**, **GPQA**, **SWE-Bench**, and **Chatbot Arena** are great for comparing the general capabilities of language models. They provide a standardized way to measure reasoning, coding, mathematics, and knowledge across many tasks.

The problem is that they don't answer a developer's most important question:

> **Which model works best for my application?**

For example, if you're building an AI-powered nutrition assistant, benchmark scores won't tell you:

* Which model follows every instruction correctly?
* Which produces the most practical meal plan?
* Which responds the fastest?
* Which offers the best balance between quality, latency, and API cost?

These are application-specific questions. I needed a way to evaluate models using **my own prompts**, **my own evaluation criteria**, and **my own success metrics** rather than relying entirely on public benchmarks.

That realization became the motivation for building my own evaluation framework.

---

## Designing the Solution

Before writing any code, I focused on making the evaluation process fair and repeatable. If different models receive different prompts, temperatures, or token limits, the comparison becomes unreliable.

I designed the framework around four simple principles:

* **Fairness** — Every model receives identical inputs.
* **Repeatability** — Running the evaluation again should produce comparable results.
* **Automation** — The workflow should require minimal manual effort.
* **Transparency** — Every score should be backed by measurable evidence.

With those principles in place, the evaluation pipeline became straightforward:

```text
User Prompt
      │
      ▼
Multiple LLMs
      │
      ▼
Collect Responses
      │
      ▼
Python Metrics
      │
      ▼
LLM-as-a-Judge
      │
      ▼
Leaderboards & Reports
```

The core idea is simple: every model receives the **same prompt** under the **same evaluation settings**, ensuring that the only variable being tested is the model itself.

```python
# Evaluate every model using the same prompt
responses = []

for model in MODELS:
    response = evaluate(model, PROMPT)
    responses.append(response)
```

Python measures objective metrics such as latency, token usage, and instruction compliance, while an **LLM-as-a-Judge** evaluates response quality using a predefined scoring rubric. Combining these two layers produces a more balanced evaluation than relying on either one alone.

> **A fair comparison isn't about giving every model the same score—it's about giving every model the same opportunity to succeed.*

---

# Architecture

Instead of placing all the logic in a single Python file, I designed the project as a modular evaluation pipeline. Each module has one responsibility, making the framework easier to understand, test, and extend as new models or evaluation tasks are added.

The overall architecture looks like this:

```text
                     User Prompt
                         │
                         ▼
                     main.py
                         │
                         ▼
                 evaluator.py
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
   utils.py                         config.py
(API Calls & Metrics)          (Models & Settings)
        │
        ▼
   OpenRouter API
        │
        ▼
 Model Responses
        │
        ▼
     judge.py
        │
        ▼
 Structured Results
        │
        ▼
    report.py
        │
        ▼
 Leaderboard
```

The biggest advantage of this design is that each module can evolve independently. I can add new models, evaluation tasks, or scoring methods without rewriting the entire application.

---

# Project Structure

```text
prompt_eval/
│
├── main.py
├── config.py
├── evaluator.py
├── utils.py
├── judge.py
└── report.py
```
### `config.py` — Configuring the Evaluation

Instead of hardcoding models and settings throughout the project, I centralized all experiment configuration in a single file. This made it easy to swap models or tune evaluation parameters without touching the evaluation logic.

```python
MODELS = [
    "deepseek/deepseek-v4-flash",
    "qwen/qwen3-235b-a22b",
    "meta-llama/llama-4-maverick"
]

JUDGE_MODEL = "qwen/qwen3-235b-a22b"

MAX_RESPONSE_TOKENS = 1500
TEMPERATURE = 0.3
MAX_RETRIES = 3
```

### `utils.py` — Calling Models and Measuring Performance

Every model is queried through the same helper function. Along with the generated response, I also record latency and token usage so every model can be compared objectively.

```python
start = time.time()

response = client.chat.completions.create(
    model=model_name,
    messages=[{"role": "user", "content": prompt}],
    max_tokens=MAX_RESPONSE_TOKENS,
    temperature=TEMPERATURE
)

latency = round(time.time() - start, 2)
```

### `evaluator.py` — Running Every Evaluation

The evaluator sends the same prompt to every model, collects the response, and computes runtime metrics.

```python
for model in MODELS:

    result = call_model(
        model,
        PROMPT.format(task=task["task"])
    )

    metrics = calculate_metrics(
        result["answer"],
        result["latency"],
        result["total_tokens"]
    )
```
One task required a **strictly vegetarian** meal plan, so I added a deterministic validation step before the response reached the LLM judge.

```python
found_nonveg = []

for item in NON_VEG:
    if re.search(rf"\b{re.escape(item)}\b", answer_lower):
        found_nonveg.append(item)

vegetarian_followed = len(found_nonveg) == 0
```

### `judge.py` — LLM-as-a-Judge

Instead of manually reviewing responses, I asked another language model to score every response using a structured rubric.

```python
judge_prompt = f"""
Score each response for:

- Accuracy
- Instruction Following
- Completeness
- Medical Safety

Return the winner and scores.
"""
```

### `main.py` — Orchestrating the Pipeline

The entry point coordinates the entire evaluation workflow—from running model evaluations to generating the final leaderboard.

```python
results = evaluate_models()

judge_output = llm_judge(...)

leaderboard.append(...)

print_summary(results)
```

### `report.py` — Building the Leaderboard

Once every evaluation is complete, the framework aggregates runtime metrics and ranks models based on overall efficiency.

```python
ranked_models = sorted(
    stats.items(),
    key=lambda x: efficiency[x[0]],
    reverse=True
)
```

Each file has a single responsibility:

| File | Responsibility |
|------|----------------|
| `main.py` | Starts and orchestrates the complete evaluation workflow |
| `config.py` | Stores model names, prompts, tasks, and evaluation settings |
| `utils.py` | Handles API calls, latency measurement, retries, and helper functions |
| `evaluator.py` | Sends prompts to each model and collects evaluation data |
| `judge.py` | Uses an LLM to score responses against predefined criteria |
| `report.py` | Aggregates results and generates the final leaderboard |

Rather than tightly coupling everything together, this modular design makes the framework easier to maintain, debug, and extend.

# Key Code Snippets

## Measuring What Python Can Measure

The first layer of the framework focuses on **objective metrics** that can be measured deterministically. Rather than relying on subjective opinions, Python records performance statistics for every model under identical conditions.

> **Python doesn't guess—it measures.**

For every response, the framework captures:

| Metric | Purpose |
|---------|---------|
| Latency | Response time |
| Prompt & Completion Tokens | API usage |
| Tokens per Second | Generation speed |
| Word & Character Count | Response length |
| Reading Time | Estimated reading duration |
| Instruction Validation | Basic rule compliance |

These metrics make it possible to compare runtime performance consistently across all evaluated models.

### Runtime Metrics

```python
metrics = calculate_metrics(
    result["answer"],
    result["latency"],
    result["total_tokens"]
)
```

### Rule-Based Validation

Some evaluation criteria don't require another language model.

For the nutrition task, I added a deterministic validation step that checks whether a supposedly vegetarian meal plan contains prohibited ingredients.

```python
found_nonveg = []

for item in NON_VEG:
    if re.search(rf"\b{re.escape(item)}\b", answer_lower):
        found_nonveg.append(item)

vegetarian_followed = len(found_nonveg) == 0
```

This ensures instruction-following violations are detected before the response is sent to the LLM judge.

---

## Measuring What Python Cannot Measure

Performance metrics tell us **how efficiently** a model generated a response, but they don't tell us **how good** that response actually is.

For example, Python cannot determine:

- Which answer is more accurate.
- Which explanation is more helpful.
- Which meal plan is nutritionally safer.
- Which response best follows the user's intent.

To evaluate these qualitative aspects, I introduced a second evaluation layer using an **LLM-as-a-Judge**.

Every response is scored using the same rubric, allowing the framework to compare models more consistently than manual review.

| Criterion | Purpose |
|----------|---------|
| Accuracy | Correctness of the response |
| Instruction Following | Compliance with the prompt |
| Completeness | Coverage of requested information |
| Medical Safety | Avoidance of unsafe advice |
| Overall Score | Final quality assessment |

Instead of free-form feedback, the judge returns structured results that can be parsed automatically and used to generate leaderboards.

> **Python measures how a model performs. An LLM judge evaluates how well it performs.**

---

# Building the Leaderboard

Evaluating individual responses is only half the job. The real goal is to identify **which model performs most consistently across multiple tasks**.

After each evaluation, the framework combines the judge's scores with runtime metrics to generate a final leaderboard.

```text
Evaluation Tasks
        │
        ▼
Model Responses
        │
        ▼
LLM-as-a-Judge
        │
        ▼
Parsed Scores
        │
        ▼
Aggregate Results
        │
        ▼
Leaderboard
```

One interesting challenge was that the judge returns **natural language**, while the reporting pipeline needs **structured data**.

For example, the judge might produce:

```text
Winner: Model B

Overall:
Model A: 8.5/10
Model B: 9.4/10
Model C: 8.8/10
```

The framework parses this output automatically, extracts the winner and scores, and converts them into structured objects that can be ranked and analyzed.

### Parsing Judge Scores

```python
m = re.match(
    r"(Model\s+[A-Z])\s*:\s*([\d\.]+)/10",
    line
)

if m:
    overall_scores[m.group(1)] = float(m.group(2))
```

Once every task has been processed, the framework aggregates the scores, counts model wins, calculates efficiency metrics, and generates the final leaderboard.

### Ranking Models

```python
ranking = sorted(
    efficiency.items(),
    key=lambda x: x[1],
    reverse=True
)
```

This transforms qualitative LLM evaluations into structured rankings, making it easy to compare model performance across multiple tasks.

====================================================
FINAL LEADERBOARD
================================================
```
Nutrition - Basic Diet      Qwen 3 235B        9.3/10
Nutrition - Personalized    Llama 4 Maverick  9.1/10
```
====================================================
OVERALL RESULT
====================================================
```
Qwen 3 235B
Wins: 2
Average Judge Score: 9.2
```
---

# Challenges

Building the framework wasn't just about sending prompts to different models—it was about making the evaluation process reliable and repeatable. Along the way, I ran into a few engineering challenges that shaped the final design.

### 1. Managing API Costs

Evaluating multiple models across several tasks can quickly become expensive. To reduce token usage, I truncated long responses before sending them to the LLM judge and limited response previews where full outputs weren't necessary. This kept evaluation costs lower while preserving enough context for meaningful comparisons.

### 2. Handling Unpredictable API Responses

LLM APIs don't always return perfect responses. Some requests failed, some returned empty outputs, and others were incomplete. Instead of stopping the entire evaluation, I added retry logic and validation checks so the pipeline could recover automatically and continue processing the remaining models.

```python
for attempt in range(MAX_RETRIES):
    try:
        return call_model(...)
    except Exception:
        time.sleep(2)
```

### 3. Parsing Inconsistent LLM Output

Although the judge followed a structured prompt, small formatting differences—such as extra Markdown or inconsistent spacing—could break automated parsing. I made the parser more resilient using regular expressions and fallback logic so it could reliably extract winners and scores.

```python
m = re.match(
    r"(Model\s+[A-Z])\s*:\s*([\d\.]+)/10",
    line
)
```

### 4. Designing Prompts for Software

One lesson I didn't expect was that prompts written for machines look very different from prompts written for people. Free-form explanations were easy to read but difficult to process programmatically. By enforcing a fixed response structure, the framework could automatically parse scores, generate reports, and build leaderboards without manual intervention.

> **The biggest lesson from this project was that reliable AI systems aren't built by assuming everything works—they're built by planning for when things don't.**

---

# What I Learned

Building this framework changed the way I think about AI development.

I started the project assuming the hardest part would be generating responses from multiple language models. In reality, the bigger challenge was building a system that could evaluate those responses fairly and consistently.

A few key lessons stood out:

- **Evaluation matters as much as generation.** A great prompt is only useful if you have a reliable way to measure the quality of its output.
- **LLMs are inherently non-deterministic.** Consistent evaluation requires controlled prompts, identical settings, and repeatable testing—not a single impressive response.
- **Objective and subjective evaluation should work together.** Runtime metrics explain how efficiently a model performs, while an LLM judge helps assess how well it performs.
- **Reliability is built into the system.** Retry logic, validation, structured outputs, and defensive parsing proved just as important as the evaluation logic itself.

The biggest takeaway wasn't discovering the "best" model—it was realizing that trustworthy AI applications depend on trustworthy evaluation pipelines.