# StudyFlash – Intelligent Summary Editing Assistant

## Overview

This project implements a text classification system that processes user requests about summaries using LLMs. The system is designed to efficiently handle different types of summary-related queries by determining the appropriate context needed for each request.

The StudyFlash already creates per‑block summaries and a merged *Final Summary (FS)* for an uploaded document.  What I added is an **after‑the‑fact editing layer**: users can ask the system to tweak the summary—globally or by highlighting text—without us having to re‑process the entire PDF.

The key idea is to give the LLM **only the minimum context it needs** for each edit request.  We classify every user prompt into one of five scope classes and build a tailored prompt that references just the relevant pieces of the original data.  This cuts token usage while still allowing rich edits such as “elaborate on concept X” or “add more technical examples about X”.

## System Architecture

```
User Request ─▶ Classifier ─▶ Retriever (if needed) ─▶ Prompt Builder ─▶ LLM ▶ Result
```

1. **Classifier** (zero‑shot + rules) chooses exactly one label using **meta‑llama/llama‑3.3‑70b‑instruct**:

   - `FS_ONLY` – work solely with the Final Summary
   - `FS_PLUS_BS[i]` – FS + one best‑matching Block Summary
   - `FS_PLUS_BS_ALL` – FS + all Block Summaries
   - `FS_PLUS_SRC[i]` – FS + one raw Source Text chunk
   - `FS_PLUS_SRC` – FS + whole raw Source Text

2. **Retriever** (for *[i]* classes) finds the best block or source chunk via `sentence‑transformers/all‑MiniLM‑L6‑v2` embeddings.

3. **Prompt Builder** fills one of five concise templates that (a) restate the grounding rules and (b) embed the chosen context.  Templates are pure f‑strings—easy to maintain.

4. **LLM Orchestrator** sends the prompt to *openai/gpt-4o-mini* on OpenRouter (best latency/price in my tests).  A simple switch lets us fall back to Groq if needed.

5. **Post‑processor** returns the edited text to the front‑end.

### Why this works

- **Token efficiency** – the smallest viable context is always preferred.
- **Deterministic scope** – the classifier guarantees the prompt never drifts outside the allowed data.
- **Modularity** – swapping retrievers or LLMs is one line of code.


