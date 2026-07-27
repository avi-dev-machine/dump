# DPDP Compliance Platform — Interview Study Guide
### How to explain this project confidently, from concepts to implementation

---

## 1. The 30-Second Pitch (memorize this)

> "I built a RAG-based legal compliance chatbot for India's Digital Personal Data Protection Act, 2023. Instead of fine-tuning an LLM on the law — which is expensive and goes stale the moment a rule is amended — I built a pipeline that turns the raw Act, Rules, and Schedules into a structured knowledge base, retrieves the relevant provisions for any user question using semantic search, and forces the LLM to generate answers strictly grounded in that retrieved text, with citations. The system also has a self-correction loop, where the model can ask for more documents mid-answer if the first retrieval wasn't enough, and multiple anti-hallucination guardrails since this is a legal domain where invented section numbers are a real risk."

Everything below unpacks each part of that sentence — general concept first, then exactly how it was implemented here.

---

## 2. Why This Architecture Exists — The Problem Being Solved

| Naive approach | Why it fails for legal Q&A |
|---|---|
| Ask a raw LLM ("ChatGPT, what does DPDP say about consent?") | LLM either doesn't know the Act (post-training-cutoff or niche), or hallucinates a plausible-sounding but wrong section number |
| Fine-tune an LLM on the Act text | Expensive, slow to update when Rules are amended, and fine-tuning doesn't reliably teach a model to *cite exact sections* — it blends facts into fuzzy weights |
| Keyword search over the Act | Users don't phrase questions in legal language ("my data got leaked" vs. "personal data breach") — keyword search misses the match |

**The fix: Retrieval-Augmented Generation (RAG).** Keep the LLM frozen (no training), and instead retrieve the actual relevant legal text at query time and hand it to the model as context, instructing it to answer *only* from what's provided. This makes answers traceable, auditable, and instantly updatable (edit a file, don't retrain a model).

---

## 3. Core Concept #1 — What is RAG?

**General concept:**
RAG = Retrieval-Augmented Generation. A pipeline with two stages:
1. **Retrieval** — given a query, find the most relevant pieces of an external knowledge base (not the model's training data).
2. **Generation** — feed those retrieved pieces into the LLM's prompt as context, so the model answers using that specific, current, verifiable information instead of (or in addition to) whatever it memorized during training.

Why it exists: it separates *knowledge* from *reasoning ability*. The LLM stays generically smart at language and reasoning; the knowledge base is a swappable, auditable, constantly-updatable external store — like giving an open-book exam instead of expecting someone to have memorized the textbook.

**How it was implemented in this project:**
- The "book" is a structured wiki of markdown pages built from the Act, Rules, and hand-written concept summaries (see Section 5).
- The "open-book search" is a vector database (ChromaDB) that finds semantically relevant pages for a question.
- The "answering with the book open" step is a Gemini LLM call where retrieved pages are injected directly into the prompt, with a system prompt that explicitly forbids answering from anything *not* in that context.
- On top of vanilla RAG, this project adds: **query rewriting** (so user phrasing matches legal phrasing), **multi-query retrieval** (splitting complex questions into multiple searches), and a **self-correction loop** (the model can request more documents mid-generation if the first retrieval wasn't enough). These three additions are what separate this from a "toy" RAG demo — good to mention explicitly if asked "what's sophisticated about your RAG?"

---

## 4. Core Concept #2 — What is a Vector Database, and How Was It Built Here?

**General concept:**
A vector database stores documents as **embeddings** — long lists of numbers (e.g. 384 dimensions) produced by a neural network (a "sentence transformer") such that documents with *similar meaning* end up as numerically *close* vectors, regardless of exact wording. Search becomes "find the nearest vectors to my query vector" instead of "find exact keyword matches."

Why it's needed: legal text and everyday questions use different vocabulary. A keyword search for "data leak" won't find "personal data breach." An embedding-based search will, because the two phrases are semantically close even though they share no words.

**How similarity search actually works, step by step:**
1. Text is tokenized (broken into sub-word pieces).
2. A transformer network (12 layers, in this case) computes a vector for each token.
3. A pooling layer compresses all token vectors into **one vector per document**.
4. At query time, the same model embeds the user's query into a vector.
5. **Cosine similarity** (or an equivalent distance metric) is computed between the query vector and every stored document vector — vectors pointing in a similar "direction" in that 384-dimensional space are semantically similar.
6. Because comparing a query against millions of vectors one-by-one is slow, real vector databases use **approximate nearest-neighbor (ANN) algorithms** — this project's engine, ChromaDB, uses **HNSW** (Hierarchical Navigable Small World), a graph-based structure that finds "close enough" matches in roughly logarithmic time instead of scanning everything.

**How it was actually built in this project:**
- **Embedding model:** `BAAI/bge-small-en-v1.5`, a 33M-parameter sentence transformer, deliberately chosen over the more common `all-MiniLM-L6-v2` (22M params). Reason: bge is trained specifically on *asymmetric* pairs — a short question vs. a long declarative document — which is exactly the shape of this problem (short user query vs. a full wiki page). Its retrieval benchmark score (MTEB) is meaningfully higher (51.7 vs 40.1).
- **Storage engine:** ChromaDB — a lightweight, file-based (no external server needed) vector database. Each wiki page is embedded and stored with metadata: page ID, page type, source document, title, tags, file path.
- **Score conversion:** ChromaDB returns L2 (Euclidean) distance, not cosine similarity directly. Since vectors are normalized to unit length before storage, `score = max(0, 1 - distance)` is a valid simplification (mathematically, for unit vectors, `L2² = 2(1 − cosine similarity)`).
- **Score bands used to interpret results:**

  | Score | Meaning |
  |---|---|
  | 0.85 – 1.00 | Near-exact semantic match |
  | 0.60 – 0.85 | Strong topical match |
  | 0.35 – 0.60 | Moderate relevance |
  | 0.25 – 0.35 | Weak/borderline |
  | < 0.25 | Low confidence — likely off-topic, triggers a guardrail (see Section 8) |

- **Freshness handling:** the index is rebuilt automatically if any wiki markdown file has a modification timestamp newer than the ChromaDB database file — meaning the vector index can never silently go stale relative to the source content. There's also a version tag on the collection name (`dpdp_wiki_v3`) so that bumping it forces a full rebuild during development (e.g. after switching embedding models).

**One-line interview answer if asked "what is a vector database" cold:**
> "It's a database that indexes documents by their *meaning* rather than their exact words, by storing each document as a numeric vector from a language model, so a search for 'data leak' can still find a document that only says 'personal data breach.'"

---

## 5. Core Concept #3 — What is the "Wiki," Why Was It Needed, and How Was It Formed?

This is the most distinctive design decision in the project — worth being very sharp on this section specifically.

**What it is:**
A folder of clean markdown files — one file per legal provision or synthesized topic (e.g. `wiki/act/section_8.md`, `wiki/rules/rule_7.md`, `wiki/concepts/consent.md`). Each file has YAML frontmatter (title, tags, source, path) plus a structured body (summary, key points, verbatim legal text, cross-references to other sections).

**Why it was needed (the core insight):**
The raw legal source material was *not directly usable* for retrieval:
- The **Act** existed as deeply nested JSON (chapter → section → clause → sub-clause) — not a coherent "document" you could hand to an embedding model as-is.
- The **Rules** were extracted from a government Gazette PDF, full of formatting noise: page-separator banners, repeated Hindi-language headers, and inconsistent spacing.
- Real compliance questions are **cross-cutting** — "what are my obligations for children's data?" pulls from multiple sections and multiple rules that don't sit next to each other in the source material at all.

The wiki is the answer to all three problems at once: it's a **data engineering layer** that sits between messy raw sources and the retrieval layer, so that by the time anything reaches the vector database, it's already clean, self-contained, and annotated with exactly the metadata the LLM needs to cite it correctly.

**How it was formed — the ingestion pipeline (`ingest.py`):**

1. **Act → section pages:** the nested JSON is recursively flattened (chapter → section → clauses → sub-clauses) into plain text. Each section is auto-tagged via a keyword-to-tag dictionary (e.g. any section containing "breach" or "personal data breach" gets tagged `breach`), and cross-references are extracted with a regex matching patterns like `section 8` or `§8`. One markdown file is written per section.

2. **Rules → rule pages:** the Gazette-extracted text is first cleaned with regexes (stripping `=== PAGE N of 18 ===` separators and repeated header blocks), then individual rules are parsed out using a regex that matches the "N. TITLE —" pattern that starts every rule. Each rule is then paired with its associated Schedule content via a hardcoded rule-to-schedule mapping, so the LLM receives both the rule text *and* its schedule in a single retrievable document — avoiding a situation where the rule is retrieved but its critical schedule (e.g. penalty amounts, technical standards) is not.

3. **Concept pages (synthesis):** six hand-authored pages — `consent`, `data_fiduciary`, `data_principal_rights`, `penalties`, `children_data`, `data_transfer` — each pre-combine content from multiple Act sections and Rules. These exist because a real user question ("what are my children's-data obligations?") shouldn't require the retrieval system to independently discover and stitch together four separate provisions at query time; the synthesis is done once, upfront, at ingestion time.

4. **Index page:** `wiki/index.md` catalogues every page in the wiki with a summary and its tags — a navigable map of the whole knowledge base, which the LLM can also use to orient itself.

**Interview soundbite:**
> "The wiki turns retrieval into an already-solved problem before a single query is even run — every document that can be retrieved is pre-cleaned, self-contained, and pre-annotated with the metadata the model needs to cite it correctly. Most of the hard 'RAG quality' work in this project actually happened at the data-engineering stage, not the retrieval stage."

---

## 6. How the Chatbot Actually Functions — Full Trace of One Query

Use this section to "walk through" the system live if an interviewer says "trace a query for me."

**Example question:** *"A hospital collected patient data for treatment. Can they use it to train an AI model?"*

**Step 0 — Query Encoding** (Gemini 2.0 Flash, temperature = 0.0)
The raw question is rewritten by a small LLM call into a structured legal search spec:
```json
{
  "query_type": "situation",
  "actors": "hospital (Data Fiduciary), patients (Data Principals)",
  "core_inquiry": "Does using treatment data for AI model training require fresh consent?",
  "concepts": "consent, purpose limitation, legitimate use, section 7",
  "search_queries": [
    "purpose limitation consent new use section 6",
    "legitimate uses without consent section 7",
    "research archiving exemption section 17"
  ]
}
```
This step exists because the raw question and "Section 7 — Certain legitimate uses of personal data" are semantically distant as plain text — this rewriting step bridges that gap before embedding search even runs. Temperature is 0.0 (not the 0.1 used later) because this step must output clean, parseable JSON; any randomness risks a broken parse. If parsing fails for any reason, there's a fallback that just uses the raw question as its own search query, so the pipeline never crashes.

**Step 1 — Retrieval**
Each of the 2–3 generated search queries is run separately against the ChromaDB wiki collection (top 10 results each). Results are merged and deduplicated by page ID (keeping the highest score if a page appears from more than one query), then capped at the top 10 overall.

**Step 2 — Context Building**
The retrieved pages are formatted into a single context block, each with a header showing its source label (`[Act]` / `[Rules]` / `[Wiki]`), title, file path, relevance score, and tags — followed by the **full page text**, not a short snippet. If the best score across all retrieved pages is below 0.25, a `LOW_CONFIDENCE` warning is prepended to the very top of the context, instructing the model to answer strictly from explicit text and not infer.

**Step 3 — Generation with Self-Correction** (Gemini 2.5 Flash, temperature = 0.1, streamed)
The first ~150 tokens of the model's response are buffered (held back from the user) while the system checks:
- If `<think>` appears → the model has decided the context is sufficient and is reasoning normally → the buffered tokens are released and streaming continues live.
- If `[READ: page_id, ...]` appears → the model is signalling it needs more documents. The system fetches the requested pages instantly from an in-memory dictionary (bypassing the vector database entirely, since it's a direct lookup by known page ID), and re-invokes the model with the expanded context — with an added instruction not to request yet another round, preventing an infinite loop.

The user never sees a broken or restarted answer — this decision happens silently before anything is shown.

**Step 4 — Tiered Reasoning inside the Answer**
The model classifies its own query into one of three tiers before answering:
- **Tier 1 (simple):** direct 1–3 sentence answer with citations, no visible reasoning block.
- **Tier 2 (multi-provision):** must show a `<think>[EVIDENCE]...</think>` reasoning block, then a numbered, cited answer.
- **Tier 3 (complex/conflicting):** must show a structured reasoning chain — `[DECOMPOSITION] → [ENTITY MAPPING] → [EVIDENCE] → [CLAIM GROUNDING] → [CONFLICT RESOLUTION] → [ANSWER SKELETON] → [COVERAGE CHECK]` — before the final answer.

The hospital/AI-training example is Tier 3, because it involves a genuine legal tension: Section 6 says a new purpose requires new consent, while Section 7(b) provides a research exemption — the model must surface and resolve that conflict explicitly rather than silently picking a side.

**Step 5 — Guardrails Applied Throughout** (see Section 8 for detail)

**Step 6 — Memory & Logging**
The turn (a compressed version of the question and answer) is appended to a rolling conversation history, and the full exchange is logged for auditability.

---

## 7. Supporting Infrastructure (good breadth-of-engineering talking points)

| Feature | What it does | Why it matters |
|---|---|---|
| **API key pool (round-robin)** | Rotates across multiple Gemini API keys on every call; on a 429 (rate limit) or 503 error, immediately retries with the next key instead of failing | Free-tier LLM APIs have strict per-minute quotas — a single key would exhaust itself in one demo session or one benchmark run |
| **Conversation memory** | Keeps a rolling buffer of the last 4 turns (8 messages), storing the *compressed* `core_inquiry` (not raw user text) and a truncated 500-character answer, not the full response | Keeps token cost bounded while preserving enough context for natural follow-up questions ("What about SDFs?" after a consent question) |
| **Index freshness / auto-rebuild** | Compares the modification time of every wiki markdown file against the vector database's file timestamp; auto-rebuilds the index if any wiki page is newer | Prevents the vector index from silently drifting out of sync with the underlying legal text after edits |
| **Wiki linter** | Scans the wiki for broken cross-reference links (`[[page_id]]` pointing to a page that doesn't exist) and orphan pages (pages nothing links to) | A broken cross-ref could cause the `[READ:]` self-correction mechanism to request a page that doesn't exist — the linter catches this before it becomes a runtime failure |
| **Benchmark evaluation framework** | Runs the full pipeline against 25 curated questions (mixing simple retrieval, multi-hop reasoning, and genuine legal-conflict questions), saving results after *every single question* so partial progress survives an API quota failure | Gives a repeatable way to measure answer quality across the three difficulty tiers, and a reproducible shuffle (fixed random seed) so results are comparable across runs |
| **Interactive web viewer** | A single self-contained HTML file with the entire Act's JSON data embedded inline — no server or network calls required | Zero-dependency browsing/reference tool; works offline, supports native browser search (Ctrl+F) because everything is rendered at once rather than loaded on demand |
| **Compliance mind map** | A zoomable tree-diagram view of the whole compliance framework, backed by a hand-structured JSON tree with detailed legal analysis at every node | A standalone conceptual reference, not just a navigation aid — useful for explaining the *shape* of the law, not just looking up one section |

---

## 8. Core Concept #4 — Anti-Hallucination Design (a legal-domain-specific concern)

**General concept:** LLMs are known to "hallucinate" — state facts confidently that are not true, or that are not present anywhere in their given context. In a legal-compliance tool, a hallucinated section number or invented obligation is not a cosmetic bug — it's a trust-destroying failure. RAG *reduces* hallucination risk (because the model has real text to draw from) but does not eliminate it on its own; the model still has to be constrained to actually *use* that text rather than free-associate.

**How it was implemented — five layers, stacked together:**

1. **Explicit prohibition list** in the system prompt: never invent section/rule numbers, never fabricate obligations or penalties not stated in the retrieved text, never claim something "must" happen unless the text actually says so.
2. **Explicit permission list**: the model *is* allowed to explain a provision in plain language, synthesize across multiple provisions, and draw reasonable conclusions that logically follow from the text — this draws a clear line between "reasoning from the text" (encouraged) and "inventing text" (forbidden), rather than making the model overly timid.
3. **Confidence calibration policy**: when the text is clear, answer directly with no hedging; when genuinely ambiguous, say so explicitly — the goal is to avoid the opposite failure mode of a model that hedges every single answer into uselessness ("it depends, consult a lawyer" for everything).
4. **Low-confidence flag**: if the best retrieval score is below 0.25, a warning is injected at the very top of the context telling the model to restrict itself to only the explicit text provided, with no inference.
5. **Ambiguity-resolution rule**: a two-step procedure — first check if any retrieved provision explicitly resolves an apparent conflict; if not, present *both* interpretations and their practical implications rather than picking one, and recommend qualified legal counsel.

**Interview soundbite:**
> "RAG gets you most of the way to reducing hallucination, but it doesn't guarantee the model actually sticks to the retrieved text. I layered explicit forbidden/allowed rules, a confidence-based context warning, and an ambiguity-resolution procedure on top, specifically because a legal tool that confidently states the wrong penalty is worse than one that says 'this is unclear, consult a lawyer.'"

---

## 9. Key Engineering Decisions Worth Highlighting (shows judgment, not just implementation)

- **Two vector-store implementations exist in the codebase** — an earlier, more granular "three-tier" store (section/clause/leaf level, with automatic cross-reference expansion and parent-chunk promotion) and the current production "wiki-page" store (page-level granularity, richer pre-synthesized documents, simpler retrieval). This is a good story about **iteration**: the first version optimized for retrieval precision at the cost of complexity (three separate database queries per search, plus cross-reference expansion logic); the second version simplified retrieval by investing more in the data-engineering stage instead, and moved the "did we retrieve enough?" problem from an automatic mechanism into an LLM-driven `[READ:]` self-correction step.
- **Streaming with token buffering** — rather than committing to whatever the model outputs first, the system holds back the first ~150 tokens to silently decide whether a self-correction round is needed, so the user-facing experience is always a single clean, complete answer.
- **Temperature choices are deliberate, not defaults** — 0.0 for the structured-JSON query encoder (any randomness risks breaking the parser), 0.1 for the final answer (near-deterministic factual content, but with enough variation to avoid robotic, identical phrasing on repeated similar questions).
- **Async-readiness in the API design** — the planned FastAPI endpoint uses `run_in_executor` to offload blocking HTTP calls (to the Gemini API) onto a thread pool, since the underlying `requests` library is synchronous and would otherwise block FastAPI's async event loop.

---

## 10. Rapid-Fire Q&A / Flashcards

**Q: What is RAG in one sentence?**
A: Retrieving relevant external documents at query time and injecting them into an LLM's prompt, instead of relying purely on what the model memorized during training.

**Q: What is a vector database in one sentence?**
A: A database that indexes documents by semantic meaning (via numeric embeddings) rather than exact keywords, so a search can match paraphrased or differently-worded queries.

**Q: Why not just fine-tune the LLM on the Act?**
A: Fine-tuning is expensive, goes stale the moment the law is amended, and doesn't reliably teach precise citation — RAG keeps knowledge external, swappable, and auditable.

**Q: Why build a "wiki" instead of embedding the raw JSON/PDF directly?**
A: The raw sources were structurally unusable for retrieval (deeply nested JSON, PDF extraction noise) and didn't support cross-cutting questions that span multiple sections — the wiki pre-cleans and pre-synthesizes this once, upfront.

**Q: What embedding model was used and why?**
A: `BAAI/bge-small-en-v1.5`, chosen over `all-MiniLM-L6-v2` because it's trained on asymmetric query-document pairs (short question vs. long document), matching this exact retrieval shape, and scores meaningfully higher on retrieval benchmarks.

**Q: How does the system avoid hallucination?**
A: Five layers — explicit forbidden/allowed instruction lists, confidence calibration guidance, a low-confidence context warning below a similarity threshold, and an explicit ambiguity-resolution procedure that presents both readings rather than picking one when the law is genuinely silent.

**Q: What happens if the first retrieval doesn't have enough information?**
A: The model can emit a `[READ: page_id, ...]` flag mid-generation; the system detects this during a token-buffering window, fetches the requested pages, and re-generates the answer with expanded context — capped at one retry via prompt instruction.

**Q: How do you keep the vector index in sync with the legal text?**
A: File-modification-time comparison — if any wiki markdown file is newer than the vector database's file, the index is automatically rebuilt on next startup.

**Q: What's the difference between the query encoder and the answer generator?**
A: The encoder (temp 0.0) turns an ambiguous user question into structured legal search terms and outputs strict JSON; the answer generator (temp 0.1) takes the retrieved legal text and produces the final cited, streamed response.

---

## 11. Glossary (quick reference)

| Term | Meaning |
|---|---|
| **RAG** | Retrieval-Augmented Generation — retrieve relevant docs, then generate an answer grounded in them |
| **Embedding** | A numeric vector representation of text that captures semantic meaning |
| **Vector database** | A database optimized for storing and searching embeddings by similarity |
| **Cosine similarity** | A measure of how "aligned" two vectors are — used to rank document relevance to a query |
| **HNSW** | Hierarchical Navigable Small World — a graph algorithm for fast approximate nearest-neighbor search |
| **Chunking** | Splitting a document into smaller retrievable units (here: whole wiki pages, rather than small paragraph chunks) |
| **Hallucination** | An LLM confidently stating something false or unsupported by its given context |
| **Sentence transformer** | A neural network fine-tuned specifically to produce good sentence/document-level embeddings |
| **Frontmatter** | Structured metadata (YAML) at the top of a markdown file, separate from its main content |
| **mtime** | File modification timestamp, used here to detect when the vector index has gone stale |

---

*Study guide compiled from the DPDP Compliance Platform technical reference — covers ingestion, vector search, wiki formation, the query→retrieval→generation pipeline, self-correction, anti-hallucination design, and supporting infrastructure.*
