# Server-Sent Events (SSE) & Real-Time Streaming

## What is it?
Server-Sent Events (SSE) is a standard web technology that allows a server to push real-time updates to a web client over a single, long-lived HTTP connection. Unlike WebSockets which provide bi-directional communication, SSE is strictly one-way (Server to Client), making it significantly simpler to implement, scale, and debug. 

In the context of AI applications, SSE is the industry standard for streaming LLM "typing" text (tokens) and background thinking steps progressively to the user's screen.

## How to Implement it

### 1. The Backend (FastAPI Example)
To implement SSE, you create an asynchronous generator function that yields strings formatted according to the SSE event stream spec (must end with `\n\n` or `\n`). You return this generator wrapped in a `StreamingResponse` with the `text/event-stream` mime type.

```python
from fastapi.responses import StreamingResponse
import asyncio
import json

async def event_generator():
    yield json.dumps({"type": "status", "message": "Thinking..."}) + "\n"
    await asyncio.sleep(1)
    
    words = ["Here ", "is ", "my ", "answer."]
    for word in words:
        yield json.dumps({"type": "token", "content": word}) + "\n"
        await asyncio.sleep(0.1)

@app.get("/stream")
async def stream_endpoint():
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### 2. The Frontend (React/TypeScript Example)
On the client side, use the native `fetch` API. Do not await the entire response. Instead, hook into the `response.body.getReader()` to parse chunks as they arrive over the wire.

```typescript
const streamResponse = async () => {
    const response = await fetch('/stream', { method: "GET" });
    const reader = response.body?.getReader();
    const decoder = new TextDecoder();
    
    while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value);
        const lines = chunk.split('\n');
        
        for (const line of lines) {
            if (!line.trim()) continue;
            const data = JSON.parse(line);
            if (data.type === 'token') {
                setAnswer(prev => prev + data.content);
            }
        }
    }
};
```

## When to Use It
* **LLM Token Streaming:** To provide instant user feedback while a large language model is generating a long response.
* **Progress Bars:** For multi-step synchronous processes (e.g., File parsing 20% -> 50% -> 100%).
* **Live Dashboards:** Emitting stock tickers or sensor data where the client only needs to read, not write.

## When NOT to Use It
* **Bi-directional communication:** If the client needs to rapidly send data *back* to the server simultaneously (use WebSockets).
* **Massive File Transfers:** Do not stream massive binary artifacts via SSE JSON encoding; use standard REST downloads.
* **Fire-and-Forget Long Jobs:** If a task takes longer than an HTTP load balancer's timeout (often 60 seconds), the connection will drop. Use Background Jobs for these.

---

## Real-World Example in This Codebase

In this application, standard LangChain streaming (`astream_events`) was insufficient because specialized tools and dynamically created sub-agents could not bubble their nested events up to the top-level supervisor. 

To solve this, the application utilizes a hybrid SSE approach:

1. **ContextVar Queue:** In `backend/app/services/event_streaming.py`, the system creates an `asyncio.Queue` locked into a thread-safe `ContextVar`.
2. **Deep Emission:** Deeply nested agents (like the EDA agent) and custom Python tools (like `ToolNode`) call `push_step_event()`, inserting JSON blocks containing their internal SQL queries, Python code execution, or thought processes into this Queue.
3. **Queue Draining:** Inside the overarching streaming loop in `backend/app/services/agent_service.py`, the server continuously checks `while not tool_event_queue.empty():` and instantly `yield`s any trapped tool events directly down the SSE HTTP socket to the frontend.
4. **React Aggregation:** In `frontend/services/geminiService.ts`, the frontend decodes the `text/event-stream` stream and maps `{type: "step"}` chunks to the `ThinkingProcess` UI component, and `{type: "token"}` to the standard chat bubble. This allows the user to see the agent literally writing code and analyzing data in real-time before the final answer arrives.
