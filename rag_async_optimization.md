# RAG (Retrieval-Augmented Generation) Async Concurrency

## What is it?
Retrieval-Augmented Generation (RAG) pipelines empower LLMs to search a factual database (Knowledge Base) to inject proprietary truth into their context windows before answering a question. 

Advanced LLM workflows rarely fire a single overarching query to the vector database. Instead, they behave like human researchers, generating 5 distinct sub-questions out of the user's prompt (e.g., "What was the revenue in Q1?", "What was the revenue in Q2?", "Did any leadership change?"). 

Because fetching answers from vector databases (like `pgvector`) takes time, executing these 5 sub-queries sequentially (`for query in sub_queries: wait()`) drastically spikes the total latency. Async concurrency fixes this.

## How to Implement it

### 1. The Asynchronous Vector Search Function
First, ditch the synchronous Python database drivers (like `psycopg2` or `sqlalchemy` sync engine). You must use `async`/`await` powered drivers like `asyncpg`.

```python
import asyncpg
from typing import List

async def fetch_vector_similarities(query: str) -> List[dict]:
    # Use native purely asynchronous drivers
    conn = await asyncpg.connect('postgresql://user:pass@host/db')
    try:
        results = await conn.fetch('''
            SELECT content FROM documents 
            ORDER BY embedding <-> $1 LIMIT 3
        ''', query)
        return results
    finally:
        await conn.close()
```

### 2. The `asyncio.gather` Dispatcher
Instead of using a `for` loop, you build an array of "Futures" (unstarted asynchronous tasks), and tell Python to fire them down the network socket at the exact same moment via `asyncio.gather`.

```python
import asyncio

async def advanced_rag_search(user_prompt: str):
    # The LLM decided to break the prompt into 3 sub-queries
    sub_queries = [
        "Financial results 2023",
        "Employee headcount 2023",
        "CEO quotes 2023"
    ]
    
    # 1. Build an un-evaluated list of tasks
    tasks = [fetch_vector_similarities(q) for q in sub_queries]
    
    # 2. FIRE! Python executes all 3 network calls concurrently.
    # The total time is Equal to the SINGLE SLOWEST query, not the sum of all 3.
    results = await asyncio.gather(*tasks)
    
    return [item for sublist in results for item in sublist] # flatten
```

## When to Use It
* **Multi-hop RAG architectures:** Any architectural pipeline that breaks a complex question into multiple distinct semantic vector searches.
* **Aggregating multiple APIs:** If an agent needs data from Jira, Slack, and Github to answer a prompt, hit all 3 concurrently.

## When NOT to Use It
* **Sequential Chain-of-Thought:** If query B's existence strictly fundamentally requires knowing the answer to query A (e.g., "Find the user ID for John. Then look up the purchase history for that User ID.").

---

## Real-World Example in This Codebase

In this application, the `usecase_planner_agent.py` was previously severely rate-limited by sequential execution. The Planner Agent was instructed to generate multiple hypotheses and queries regarding the user's request (e.g., generating 4 distinct SQL table structure questions or RAG documentation lookups separated by a pipe `|`).

However, because the `rag_agent/query.py` utilized unmanaged, purely synchronous `psycopg2` database connections and the synchronous Gemini SDK (`genai.Client().models.generate_content`), every loop iteration aggressively blocked the main thread. To circumvent the 2-minute API timeouts caused by executing four 15-second searches sequentially, developers literally implemented a `break` hard-stop into the loop to artificially truncate the AI's research.

To optimize latency without stripping the AI's intelligence:
1. The engineering team rebuilt `query.py` to expose pure asynchronous functions: `async_embed_texts()`, `async_pgvector_search()`, and `async_query()`.
2. To avoid `RuntimeError` event loop collisions (as `AsyncSessionLocal` pools bind to the FastAPI boot sequence), `async_pgvector_search` ditched the global SQLAlchemy pool entirely and bound `asyncpg.connect()` straight into the localized worker instance.
3. Finally, in `usecase_planner_agent.py`, the explicit `break` loop was eviscerated and replaced with `answers = await asyncio.gather(*[async_query(q) for q in queries])`.

The agent now interrogates the Knowledge Base infinitely deeply in the exact same time envelope it previously took to test just one assumption.
