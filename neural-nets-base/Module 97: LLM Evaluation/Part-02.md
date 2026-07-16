Shawhin’s video gets to the absolute core of what separates amateur prompt-hacking from production-grade AI engineering.

When building software with traditional code, execution is deterministic (Input A always yields Output B). But Large Language Models (LLMs) are **probabilistic**—they predict the next token based on probability distributions. This means you can't test them with traditional unit assertions, and it's why so many developers fall into the trap of "vibe coding" (endlessly tweaking a prompt, only to accidentally break something else).

To build reliable AI apps, you must treat your LLM's outputs like qualitative data. **Error Analysis** is the structured, scientific way to do exactly that.

---

## The LLM Error Analysis Workflow

Below is the step-by-step methodology used by top AI engineering teams (heavily inspired by qualitative research methods like *Open Coding* and *Axial Coding*).

1. **1. Pull Failed Traces:** The Open Coding Phase.
Gather a sample of **50 to 100 real user conversations (traces)** from your logs. Don't rely on imaginary test cases yet—you want to see what actual users are doing.


2. **2. Journal and Annotate:** Find the first point of failure.
Read every trace chronologically. When you spot a bug, **write a free-text, human-readable note** describing what went wrong. Focus intensely on the **earliest failure point**—errors cascade, so fixing the first domino usually resolves the downstream chaos.


3. **3. Cluster into a Taxonomy:** The Axial Coding Phase.
Group your free-text notes into distinct, recurring **failure categories** (e.g., "hallucination", "retrieval_miss"). You don't need a massive list—aim for 5 to 10 categories that capture 80% of your problems.


4. **4. Quantify and Prioritize:** Turn quality into data.
Tag every trace in your sample set against your new taxonomy. Count the occurrences of each failure category. This turns vague feelings like *"the bot is acting weird"* into precise, actionable data (e.g., *"Our formatting fails in 18% of runs"*).


5. **5. Build the Evals:** The Handshake.
Convert your most critical failure cases into a **"Golden Dataset"** (must-pass tests). Write automated evaluations (like LLM-as-a-Judge) targeting those specific failure modes before pushing new prompts to production.


---

## 2026 Industry Standard Failure Taxonomy

When you begin grouping your notes, you don't need to reinvent the wheel. Most LLM and Agent failures fall into a few well-defined buckets:

| Failure Category | What It Looks Like | Common Root Cause | How to Fix It |
| --- | --- | --- | --- |
| **Retrieval Miss (RAG)** | The model answers "I don't know" or misses key context. | Bad search index, poor chunking, or weak query expansion. | Improve your chunking strategy or add a reranking step. |
| **Hallucination** | The model confidently asserts a factually incorrect detail. | Overconfidence, loose grounding, or too-long context windows. | Tighten system prompts, lower temperature, or add a validation step. |
| **Schema Violation** | The generated JSON or tool-call output is malformed. | Ambiguous instructions or weak model instruction-following. | Use structured output features (like Pydantic/JSON Schema). |
| **Refusal Mismatch** | The model rejects a safe, valid user query. | Overly sensitive safety guardrails or defensive prompting. | Calibrate the system prompt to explicitly state what is allowed. |
| **Tone/Style Mismatch** | The output sounds too robotic, overly formal, or lacks empathy. | Vague styling guidelines or a lack of real-world examples. | Implement **Few-Shot Prompting** by adding 3 to 5 "Golden Examples". |

---

> **The Golden Rule of LLM Evals:** Do not write generic evaluation metrics (like general "helpfulness" or "friendliness") that you found online. Your metrics should be born directly from your error analysis. If you haven't seen a specific failure mode in your real logs, do not waste time writing an evaluation for it.

By moving from passive monitoring to structured error analysis, you transition from "AI whispering" to rigorous, iterative engineering.
