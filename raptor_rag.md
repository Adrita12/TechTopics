# RAPTOR RAG (Recursive Abstractive Processing for Tree-Organized Retrieval)

## What is it?
Traditional RAG (Retrieval-Augmented Generation) excels at answering highly specific, needle-in-a-haystack questions by finding 3 or 4 semantically similar text chunks. However, traditional RAG fails completely at **global holistic questions** (e.g., *"What is the overarching theme across this 500-page book?"*).

RAPTOR solves this by building a semantic "Tree". It takes thousands of document chunks, groups them into conceptual clusters, and uses an LLM to generate a single "Summary" for each cluster. Those summaries are then clustered and summarized *again* (and again), forming a hierarchical tree. When a user asks a high-level question, the database can retrieve the high-level summary nodes instead of scattered, disconnected leaf chunks.

## How to Implement it

### 1. Vector Clustering (Agglomerative)
You first embed your base document chunks. Instead of immediately storing them, you run them through a machine learning clustering algorithm (like `AgglomerativeClustering` from scikit-learn) to identify semantic neighborhoods.

```python
from sklearn.cluster import AgglomerativeClustering
import numpy as np

# Group 1000 chunks into roughly 100 semantic clusters
model = AgglomerativeClustering(n_clusters=100)
labels = model.fit_predict(chunk_embeddings)
```

### 2. Recursive Summarization
For every identified cluster, take all the text chunks inside it and feed them to an LLM to generate a macro-summary of that concept.

```python
def summarize_cluster(cluster_texts: list[str]) -> str:
    prompt = f"Summarize the overarching themes of these documents:\n\n{cluster_texts}"
    return llm.invoke(prompt)
```

### 3. Tree Depth and Flattening
You treat the newly generated summaries as your *new* base documents, and repeat the clustering process until you reach the "Root" of the tree (a universal summary).

Finally, you **flatten** the entire tree (leaves + all parent summaries) and insert every single node into your Vector Database (like `pgvector`).

## When to Use It
* **Long-Form Document Analysis:** Analyzing massive legal contracts, entire books, or multi-year financial reports where themes span across hundreds of pages.
* **Thematic Questioning:** When users are likely to ask "What is the general consensus regarding X?" rather than "What is the exact value of X on page 42?".

## When NOT to Use It
* **Simple Q&A:** If you are building a bot to purely fetch explicit error codes from an API documentation page, hierarchical summarization adds heavy upfront LLM token costs with zero retrieval benefit.
* **Rapidly Changing Data (Real-time feeds):** Re-calculating the Agglomerative Clusters and re-generating LLM summaries takes significant compute. It is designed for static or slow-moving Knowledge Bases, not a live Twitter feed.

---

## Real-World Example in This Codebase

In this application, standard RAG was insufficient to build accurate "Knowledge Bases" for complex Marketing Mix Optimization constraints.

1. **The Tree Builder:** In `backend/app/agents/rag_agent/tree.py`, the system defines a recursive `build_tree()` function. It accepts raw texts, clusters them using `sklearn.cluster.AgglomerativeClustering`, and passes the sub-clusters into `summarize_cluster()` which uses `gemini-2.5-flash` to recursively compress the semantic meaning.
2. **The Output Structure:** A custom `Node` class holds the `<text>`, `<embedding>`, `<level>`, and `<cluster_id>`. Level 0 nodes are the highest abstract summaries, progressing deeper into the tree until it hits the raw leaf chunks.
3. **Database Flattening:** Inside `backend/app/agents/rag_agent/build_raptor.py`, the `flatten()` function traverses the generated tree, accumulating the exact hierarchical path string (e.g., `L0-C1 > L1-C4 > L2-C12`). 
4. **Vector Storage:** Every single node (both the raw chunks and the AI-generated summaries) is embedded using `gemini-embedding-001` and blindly inserted into the standard PostgreSQL `pgvector` table, indexed using an `IVFFLAT` strategy.

Because both the macro-summaries and the micro-chunks live in the exact same vector space, the retrieval Agent doesn't even need to know RAPTOR exists. If the human asks a broad question, the vector math naturally pulls the relevant high-level `Level 0` summary node. If the human asks a hyper-specific question, the math naturally pulls the `Level 4` leaf node.
