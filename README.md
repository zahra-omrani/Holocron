# Holocron: Custom Information Retrieval System

This notebook, titled "Holocron", is an information retrieval system. Inspired by the ancient, knowledge-storing artifacts of the Star Wars universe, Holocron implements custom text preprocessing, lexicon generation, inverted and direct indexing, BM25 scoring, and Document-At-A-Time (DAAT) candidate traversal[cite: 1]. Evaluated against the Vaswani corpus using standard TREC benchmarks[cite: 1].

---

### Key Features

* **Text Normalization & Tokenization:** NFKD Unicode decomposition, ASCII stripping, lowercase normalization, acronym period removal, URL/email filtering, punctuation stripping, sub-word boundary splitting via `wordninja`, stopword removal, and Snowball stemming.
* **Modular Index Architecture:** Inverted index mapping term IDs to document postings, direct index mapping document IDs to term frequencies, and a synchronized global lexicon.
* **Document-At-A-Time (DAAT) Engine:** Parallel cursor iteration over posting lists with candidate skipping via `get_min_docid`.
* **Dynamic Top-K Heap:** Bounded min-heap (`TopKPriorityQueue`) with threshold tracking for fast top-k candidate ranking.
* **TREC Run & Evaluation:** End-to-end evaluation exporting TREC run files evaluated via `ir_measures` against official qrels.

---

### Architecture & Components

```text
Raw Query / Documents
        │
        ▼
[Text Preprocessing] (Unicode NFKD, Regex, Wordninja, Stopwords, Stemming)
        │
        ▼
[Index Construction] ───► Lexicon, Direct Index, Inverted Index (Posting Lists)
        │
        ▼
[DAAT Retrieval] ────────► Coordinated Traversal via get_min_docid()
        │
        ▼
[Scoring & Selection] ──► BM25 Formula + TopKPriorityQueue (Min-Heap)
        │
        ▼
[TREC Evaluation] ──────► ir_measures (P@5, nDCG@10, AP)
```
---
### BM25 Scoring Model

Relevance scores for matched query terms are calculated using the BM25 model:

$$\text{BM25}(i) = \log\left(\frac{N}{\text{df}_i}\right) \times \frac{\text{tf}_i}{K \cdot \left((1 - B) + B \cdot \frac{\text{dl}_i}{\text{avgdl}}\right) + \text{tf}_i}$$

* Default parameters: $K = 1.2$, $B = 0.75$
* $N$: Total collection document count
* $\text{df}_i$: Document frequency for term $i$
* $\text{tf}_i$: Within-document term frequency[cite: 1]
* $\text{dl}_i$: Document length; $\text{avgdl}$: Average collection length[cite: 1]

---

### Dataset Specifications

Experiments run on the **Vaswani** corpus from `ir_datasets` via `python-terrier`[cite: 1]:

| Property | Value |
| :--- | :--- |
| **Corpus** | 11,429 scientific abstracts[cite: 1] |
| **Lexicon Size** | 7,638 unique terms[cite: 1] |
| **Total Tokens** | 291,789 tokens[cite: 1] |
| **Queries** | 93 natural language queries[cite: 1] |
| **Qrels** | 2,083 relevance assessments[cite: 1] |

---

### Installation

```bash
pip install wordninja pyterrier python-terrier ir_measures polars pandas nltk matplotlib seaborn
```
### Quick Start

```python
import pyterrier as pt
import pandas as pd

# 1. Initialize PyTerrier & load Vaswani dataset
dataset = pt.get_dataset('irds:vaswani')
df_docs = pd.DataFrame(list(dataset.get_corpus_iter()))
df_queries = dataset.get_topics()

# 2. Tokenize & clean
df_docs["tokens"] = df_docs["text"].apply(preprocess)

# 3. Build index
lexicon, inv, doc_index, direct_index, stats = build_index(df_docs)

# 4. Instantiate search engine
ir_index = InvertedIndex(lexicon, inv, doc_index, direct_index, stats, k=1.2, b=0.75)

# 5. Query the index (DAAT top-10)
results = retrieve_query_results("digital computer logic circuits", ir_index, heap_size=10)
print(results)  # [(score, docid), ...]
```
### Benchmark Results

Aggregated performance metrics across the 93 benchmark queries using `ir_measures`[cite: 1]:

| Metric | Score | Description |
| :--- | :--- | :--- |
| **P@5** | **0.4581** | ~46% of retrieved documents in top-5 ranks are relevant.[cite: 1] |
| **nDCG@10** | **0.4379** | Moderate ranking gain and document position discounting.[cite: 1] |
| **AP** | **0.1656** | Mean average precision across all recall cutoffs.[cite: 1] |
