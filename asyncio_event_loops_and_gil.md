# Asyncio, Event Loops, and the Global Interpreter Lock (GIL)

## What is it?
In Python, true parallelism (running multiple computations simultaneously on separate CPU cores) is severely hindered by the **Global Interpreter Lock (GIL)**. The GIL physically restricts a Python process to executing only one line of bytecode at a time, entirely preventing multi-threading from increasing raw math execution speed.

To serve 1,000 concurrent web users without spawning 1,000 OS processes, modern Python frameworks like **FastAPI** employ **`asyncio`**.

### The Event Loop
`asyncio` operates on an **Event Loop**. A single thread manages a loop that constantly checks tasks. When a task hits an "I/O bound" operation (e.g., waiting 3 seconds for a database query to return, or waiting 5 seconds for an LLM API to respond), the Event Loop hits an `await` keyword. It pauses that task, saves its state, and instantaneously switches to serving the next user whose network request just arrived. 

Because 95% of web application time is spent "waiting" for network or disk operations, this single-threaded Event Loop can juggle vast amounts of traffic blazingly fast.

## The Asyncio Lock Collision Danger
While powerful, scaling an `asyncio` ecosystem comes with rigid safety mechanisms. 

If multiple users are sharing a resource pool (like SDK connections), `asyncio` uses `asyncio.Lock()` to prevent them from corrupting the pool simultaneously. However, **Locks are permanently bound to the distinct Event Loop that created them.**

If your FastAPI application spins up a background thread utilizing `asyncio.run()` (which creates a brand *new*, isolated Event Loop), and that isolated loop attempts to acquire a Lock that was created in the *main* web server's Event Loop at boot time, Python will immediately throw a catastrophic error:
`RuntimeError: <asyncio.locks.Lock object> is bound to a different event loop`

## How to Implement Safety (Lazy Instantiation)

If you declare a third-party SDK connection or a database pool globally at the top of your python file, it binds to the startup Event Loop. You must delay the creation of these objects until *inside* the execution of the specific request so it binds to the correct loop.

### Bad Implementation (Global Binding):
```python
import genai
# EVIL: Bound to the startup loop forever.
global_llm_client = genai.Client()

async def background_worker():
    # CRASH: if this is running in a different thread/loop
    await global_llm_client.generate("Hello") 
```

### Good Implementation (Lazy Instantiation):
```python
import genai

def get_llm_client():
    # Only created EXACTLY when asked for, safely binding 
    # itself to the active user's current isolated event loop.
    return genai.Client()

async def background_worker():
    client = get_llm_client()
    await client.generate("Hello") 
```

## When to Use `asyncio`
* Whenever building API backends that handle heavy I/O (Database queries, external HTTP API calls like LLMs, File streaming).
* Whenever using FastAPI, as it is fundamentally designed around ASGI async compliance.

## When NOT to Use `asyncio`
* Heavy CPU-bound tasks (e.g., parsing massive pandas dataframes, encrypting files, image rendering). Processing CPU operations in an `async def` function will completely block the Event Loop, freezing the server for all other users. For these, you must use `ThreadPoolExecutor` or `ProcessPoolExecutor`.

---

## Real-World Example in This Codebase

In this application's RAG agent (`rag_agent/query.py`), the codebase initially suffered from massive lock crashes under load.

The Gemini AI Python SDK (`genai.Client()`) utilizes `asyncio.Lock()` internally for network connection pooling. The legacy code defined `client = genai.Client()` at the global module level.

During intense UI scaling, the main FastAPI thread would accept user tasks, but orchestrate the complex LangGraph execution inside isolated `ThreadPoolExecutor` background event loops. When the Planner Agent triggered the RAG search, the isolated background loop attempted to utilize the global `genai.Client()`. Because the client's internal `asyncio.Lock` belonged to the FastAPI startup loop, Python aggressively terminated the RAG agent with `RuntimeError is bound to a different event loop`. 

To fix this concurrency bottleneck, the application refactored to use dynamic instantiation: `def get_client() -> genai.Client: return genai.Client()`. Calling this lazily within the execution context of the RAG query directly guaranteed the SDK and its locks were born on the correct, isolated worker loop, eliminating crashes entirely.
