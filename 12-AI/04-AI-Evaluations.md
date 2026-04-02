# AI Evaluations

AI Evaluations (Evals) represent the systematic process of testing Large Language Models and AI systems for accuracy, safety, and reliability using both automated and human-centric benchmarks.

## Summary

In traditional software, testing is deterministic (Input A always results in Output B). In AI engineering, Large Language Models are non-deterministic, making standard unit tests insufficient for gauging performance. **AI Evaluations (Evals)** fill this gap by establishing a continuous testing framework that measures **"Fuzzy Correctness."**

This involves creating **"Gold Sets"** (curated pairs of inputs and ideal outputs), using **"Model-based Evals"** (where a stronger LLM like GPT-4o judges the output of a smaller model based on a rubric), and measuring specific **RAG metrics** such as Faithfulness and Answer Relevancy. Evals are the "Testing Pyramid" of the AI world, providing the statistical confidence required to move generative features from prototype to production. Without evals, a simple prompt change is a "blind update" that may unexpectedly break downstream functionality.

**Key Characteristics:**
- **Non-Deterministic Testing**: Measuring "Pass/Fail" based on semantic meaning and intent rather than exact string matching.
- **Model-Graded Evals (LLM-as-a-Judge)**: Using an high-reasoning LLM to score complex outputs (e.g., tone, reasoning, safety) based on a detailed prompt rubric.
- **Benchmarking**: Comparing model versions using industry standards like MMLU (General knowledge), GSM8K (Math), or HumanEval (Python).
- **RAG Triad Assessment**: Focusing on three critical dimensions: Context Relevancy, Faithfulness (Grounding), and Answer Relevancy.
- **Guardrails**: Real-time filtering (e.g., Llama Guard, NeMo Guardrails) to prevent toxic, biased, or unauthorized outputs.
- **Hallucination Detection**: Specialized tests to identify when a model is "imagining" facts not present in its context or training data.
- **Red Teaming**: Proactive attempts to bypass model guardrails to identify security and safety vulnerabilities.

---

## Problem Statement

### The Challenge

"It looks correct to me" is the most dangerous phrase in AI development. Without a rigorous evaluation backend, a developer might tweak a prompt to fix one bug, only to inadvertently cause 10 new hallucinations in a different scenario. Generative AI requires a **Regression Suite** that can handle the variability and "creativity" of natural language output.

### Context

- **Historical Context**: Early LLM testing (2022) was purely qualitative ("vibe checks"). As models hit production, this approach failed to scale and led to high-profile AI failures.
- **Technical Context**: Modern "Small Language Models" (SLMs like Phi-3 or Llama-8B) must be compared against "Frontier" models to see if they are intelligent enough for specific sub-tasks.
- **Business Context**: Enterprises need to quantify the ROI of AI by proving it is "99% accurate" before replacing human support agents or sales reps.
- **The Prompt Drift Problem**: As model providers (OpenAI/Anthropic) update their weights, behavior can change overnight. Evals are the only way to detect this "Model Drift."

### Consequences of Not Addressing

- **Hidden Regressions**: A prompt update that makes the model "more helpful" might break its ability to output valid JSON for the API.
- **Hallucination Cascades**: One incorrect fact in a multi-step Agentic workflow causes the entire 10-step process to fail catastrophically.
- **Safety & Compliance Breaches**: The model providing instructions for illegal acts or leaking internal secrets because it wasn't tested with "Red Teaming."
- **Model Over-Reliance**: Deploying a model that is "90% good" but fails on the 10% of critical edge cases that cause business loss.
- **Flaky Products**: Users receiving different quality levels for the same query, leading to "Un-reproducible" customer bugs.
- **Reward Hacking**: The model "cheats" on simpler metrics (like BLEU score) while providing a nonsensical answer to the user.

---

## Solution

### The AI Evaluation Lifecycle

AI Evals integrate into the standard CI/CD pipeline, acting as the "Quality Gate" for model releases.

```
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │  Gold Set    │      │  Execution   │      │   Judging    │
    │ (Inputs)     │─────▶│  (Prompt)    │─────▶│ (LLM-Judge)  │
    └──────────────┘      └──────────────┘      └──────┬───────┘
           ▲                      ▲                    │
           │              Rules:  │                    │
           │              [ Tone  │              Metrics:
           └──────────────[ Facts │              [ Accuracy
                          [ Safety│              [ Toxicity ]
                                  └────────────────────┘
```

### The RAG Triad Metrics

1.  **Context Relevancy**: Is the retrieved context actually useful for answering the user's question?
2.  **Faithfulness (Grounding)**: Is the generated answer derived *strictly* from the provided context?
3.  **Answer Relevancy**: Does the answer address the user's query directly and accurately?

### Key Components of an Eval Suite

1.  **GOLD Set (Eval Set)**: A collection of 100-500+ "ideal" input-output pairs that represent the core use cases and high-risk edge cases of the product.
2.  **Metric Calculators**:
    - **Deterministic**: Regex, JSON schema validation, exact keyword matching.
    - **Statistical**: BLEU, ROUGE, BERTScore (for summarization).
    - **Model-Based**: G-Eval, DeepEval, RAGAS (using an LLM to assign scores).
3.  **LLM-as-a-Judge Rubric**: A set of criteria (e.g., "Score 1 if toxic, Score 5 if helpful") that guides the evaluator model.
4.  **Side-by-Side (Elo) Comparison**: Presenting two completions (A and B) and asking the judge "Which is better for Task X?". This creates an "Elo rating" for prompts.
5.  **Adversarial (Red Team) Prompts**: A library of "Jailbreak" attempts to test the robustness of safety guardrails.
6.  **Continuous Evaluation**: Running a subset of evals on live production traffic to detect real-time drift.

### How It Addresses the Problem

- **Quantifies Progress**: Instead of "feeling" the prompt is better, you can say "Accuracy improved from 82% to 91% on the Gold Set."
- **Standardizes Grading**: LLM judges use a consistent, non-tiring rubric, reducing the subjectivity and fatigue of human reviewers.
- **Identifies Edge Cases**: Automated "Stress Testing" with thousands of synthetic prompts uncovers failures a human would never have thought to type.
- **Detects Environmental Drift**: Continuous evaluation of production traffic identifies when a model provider starts behaving differently on the backend.

---

## When to Use

### Appropriate Scenarios

| Scenario | Suitability | Evaluator Type | Priority |
|----------|-------------|----------------|----------|
| RAG Pipeline Tuning | ⭐⭐⭐⭐⭐ Essential | RAGAS (Faithfulness) | High |
| System Prompt Updates | ⭐⭐⭐⭐⭐ Essential | Elo / Side-by-Side | High |
| Model Upgrades (e.g. v3 to v4) | ⭐⭐⭐⭐⭐ Essential | Golden Set (Regression) | High |
| Code Completion Tools | ⭐⭐⭐⭐ High | Unit Test Execution | High |
| Creative Writing Tools | ⭐⭐⭐ Good | Human-in-the-Loop | Low |
| Classification (Sentiment) | ⭐⭐⭐⭐ High | Exact Match / F1 Score | Medium |

### Metric Selection Hierarchy

- **High Stakes (Legal/Health)**: Priority = **Faithfulness** and **Safety**. Manual human audit is required.
- **Developer Productivity (Coding)**: Priority = **Functional Correctness** (Does the code run?).
- **Customer Support (Chat)**: Priority = **Answer Relevancy** and **Tone**. LLM-as-a-Judge is efficient here.
- **Data Extraction (JSON)**: Priority = **Schema Validation** (Exact match) and **Completeness**.

### Indicators for Adoption

- **"I changed one word and it broke everything"**: Proof that you need automated regressions.
- **High volume of customer complaints**: "The AI is hallucinating" reports.
- **You are switching model providers**: Need to know if Llama 3 is "safe enough" compared to GPT-4.
- **Latency-Accuracy tradeoff**: Choosing a faster model only after proving it doesn't sacrifice quality.

---

## Tradeoffs

### Advantages

| Advantage | Description |
|-----------|-------------|
| **Confident Releases** | Deployment becomes a technical decision based on data, not a "hope." |
| **Statistical Rigor** | Moves away from one-off failure reports to "Aggregate Quality" metrics. |
| **Developer Autonomy** | Engineers can iterate on prompts with immediate feedback from the "Judge." |
| **Cost Optimization** | Proven accuracy allows for downgrading from expensive models (GPT-4) to cheap models (Llama 8B). |
| **Safety Assurance** | Validated guardrails ensure compliance with legal and ethical standards. |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| **Recursive API Costs** | Running GPT-4 to judge 10,000 Llama 3 calls can be more expensive than the inference itself. |
| **Judge Bias** | The evaluation model (e.g. GPT-4) might prefer its own writing style or show bias toward specific model formats. |
| **Maintenance Burden** | GOLD sets must be updated as the product features change, leading to "Eval Rot." |
| **Non-Real-Time** | Comprehensive evals can take minutes or hours to run, slowing down the CI loop. |
| **Complexity** | Setting up a sophisticated framework (Promptfoo/DeepEval) requires significant initial engineering. |

### Performance Optimization

- **Parallel Judging**: Running evaluations in parallel batches to reduce CI wait time.
- **Small Evaluation Models**: Using specialized 7B-parameter "Judge" models (like Selene or Prometheus) to save costs.
- **Deterministic Evals First**: Running cheap Regex/Schema checks before moving to expensive LLM-calls.

---

## Implementation Example

### 1. Model-Based Evaluation (Python/DeepEval)

```python
# AI Evaluation Implementation Example
from deepeval import assert_test
from deepeval.test_case import LLMTestCase
from deepeval.metrics import FaithfulnessMetric, AnswerRelevancyMetric

# Configuration: The Metric logic
# Faithfulness: Is the answer derived ONLY from the context?
faithfulness_metric = FaithfulnessMetric(threshold=0.8)
relevancy_metric = AnswerRelevancyMetric(threshold=0.8)

# Test Case: Input, Context, and the Model's output
test_case = LLMTestCase(
    input="What is the interest rate on the Prime savings account?",
    actual_output="The interest rate is 4.5% APY.",
    retrieval_context=[
        "Prime account interest is currently 4.5%. Basic accounts are 2.0%."
    ]
)

# Execution: Running the Eval (This uses GPT-4o as the judge)
def test_savings_accuracy():
    # Will fail if score < 0.8
    assert_test(test_case, [faithfulness_metric, relevancy_metric])

# Result Interpretation
# metric.score will be 1.0 (Success) or <1.0 (Fail)
```

### 2. Multi-Prompt Comparison (Promptfoo YAML)

```yaml
# promptfoo.config.yaml
# Comparing two prompts across a dataset of test cases
prompts:
  - "Answer professionally: {{query}}"
  - "You are a friendly support bot. Help the user: {{query}}"

providers:
  - openai:gpt-4o

tests:
  - vars:
      query: "How do I reset my password?"
    assert:
      - type: javascript
        # Custom logic: must include word 'settings'
        value: output.toLowerCase().includes('settings')
      - type: llm-rubric
        value: "does not sound like a robot"
```

---

## Anti-Pattern

### Common Mistakes

#### 1. The "Vibe Check" Deployment
Relying on a developer typing 3 queries into a chat window before deploying a change.
**Result**: Total failure to catch regressions in the other 95% of use cases.
**Fix**: Establish a minimal "Golden Set" of at least 20 entries for every new feature.

#### 2. Evaluating the Wrong Model
Using a 3.5-level model to judge a 4-level model.
**Result**: The judge is "dumber" than the examinee, leading to inaccurate or random scores.
**Fix**: Always use the most capable model available (e.g. GPT-4o or Claude 3.5 Opus) as the evaluator.

#### 3. Neglecting "Context Grounding"
Testing the LLM's answer without verifying if it matches the RAG retrieval data.
**Result**: The LLM might give a "correct" answer based on its training data, even though your RAG system failed to provide the source material.
**Fix**: Separately evaluate **Retrieval** (did you find the right doc?) and **Generation** (is the answer based on that doc?).

#### 4. Hardcoded String Matching (Non-Fuzzy)
Failing a test because the model said "four percent" instead of "4%".
**Result**: High volume of "False Failures" that developers eventually start ignoring.
**Fix**: Use semantic similarity (Cosine similarity) or LLM evaluation.

### Warning Signs

- **"I think the prompt is better now"**: Subjective language in PRs without metric support.
- **Evals always pass 100%**: Likely because the metrics are too lenient or the GOLD set is too similar to the training data.
- **Tests take too long to run**: Check for sequential LLM judging instead of parallel.
- **Model drift goes unnoticed**: You aren't running evals on production samples.

---

## Related Patterns

### Complementary Patterns

- [LLM Architecture](./01-LLM-Architecture.md) - Understanding the "Context Window" limitations that lead to hallucinations.
- [MLOps Pipeline](./03-MLOps-Pipeline.md) - Evals act as the "Unit Tests" in the AI deployment pipeline.
- [Observability](../10-Observability/04-Metrics.md) - Exporting Eval scores to Prometheus/Grafana to track performance over time.

### Alternative Frameworks

- **Promptfoo**: The standard for CLI-based side-by-side prompt evaluation.
- **RAGAS**: Specialized metrics for RAG systems (Faithfulness, Relevancy).
- **LangSmith**: Managed platform from LangChain for production trace evaluation.
- **Arize Phoenix**: Local evaluation of RAG traces and embedding-space drift.

### See Also

- [Large Language Model Evaluations (Guide by Anthropic)](https://www.anthropic.com/news/evaluating-llms)
- [How to Build an LLM Evaluation Pipeline](https://www.wandb.courses/courses/evaluating-llms)
- [OpenAI Evals Framework (GitHub)](https://github.com/openai/evals)
- [DeepEval: The Unit Testing Framework for LLMs](https://github.com/confident-ai/deepeval)

---

## Evaluation Maturity Checklist

- **Level 1**: Human "Vibe Checks" on a handful of queries.
- **Level 2**: Static "Gold Sets" with regex/string match assertions.
- **Level 3**: LLM-as-a-Judge for tone and reasoning quality.
- **Level 4**: Automated RAG assessments (Faithfulness/Grounding) in CI.
- **Level 5**: Continuous Production Evaluation with drift alerts.

### Summary Conclusion

AI Evaluation is the bridge between a "Science Project" and a "Product." In a world where AI output is fluid, Evals provide the deterministic anchor that allows engineering teams to move fast without breaking things. If you aren't measuring your AI, you aren't engineering it; you are just wishing for it to work.
