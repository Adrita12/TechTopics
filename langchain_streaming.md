# LangChain Streaming Limitations & Workarounds

## What is it?
LangGraph provides a native streaming method called `astream_events()`. It is designed to evaluate an entire state graph and automatically yield real-time logging events (`on_chain_start`, `on_tool_end`, `on_chat_model_stream`) for every node execution, allowing developers to stream the exact thoughts matching their AI architectures.

## The Limitations of Native `astream_events()`
While powerful, `astream_events()` suffers from strict rigidities when dealing with highly abstracted, complex, or non-standard architectures. 

1. **Nested Dynamic Agent Blindness:** If Agent A technically invokes Agent B via a direct function call inside its tool (e.g., `await create_sub_agent().ainvoke()`), the parent `astream_events()` loop **cannot "see"** into the sub-agent. The events do not bubble up, meaning the frontend UI freezes while the sub-agent does 5 minutes of work.
2. **Custom Code Execution Tools:** If you build a proprietary execution environment (like a custom `ToolNode` executing sandboxed Docker code), LangGraph has no native integration hook to broadcast what that code is doing internally unless you explicitly write complex native Python Callbacks via `BaseCallbackHandler`.
3. **Overly Verbose/Useless Events:** `astream_events` yields *thousands* of events regarding parser starts, dictionary formatters, and internal LLM HTTP pings. Filtering this to just the 3 or 4 things the user actually cares about becomes an engineering nightmare.

## How to Implement a Custom Queue Workaround

To fix this, we bypass the automated event bubbling of LangGraph entirely and inject a thread-safe Queue via `contextvars` directly into the deepest levels of the application.

### 1. Define the Global Context Variable
```python
from contextvars import ContextVar
import asyncio

# A special variable that behaves globally but is strictly isolated per-async-thread
_event_queue: ContextVar[asyncio.Queue] = ContextVar('_event_queue', default=None)

def push_event(event_dict):
    q = _event_queue.get()
    if q:
        q.put_nowait(event_dict)
```

### 2. Set the Queue at the Top Level Stream
```python
async def orchestrator_stream():
    queue = asyncio.Queue()
    _event_queue.set(queue) # Lock queue to this user's thread
    
    # Start graph (in background task or within the loop)
    async for langgraph_event in graph.astream_events(state):
        
        # 1. Drain our custom "deep" queue first
        while not queue.empty():
            yield queue.get_nowait()
            
        # 2. Yield whatever high level stuff LangGraph saw
        yield process_native_event(langgraph_event)
```

### 3. Push Events from Anywhere in the Codebase
Because `ContextVar` flows effortlessly down the async stack, *any* nested function, dynamic agent, or custom tool can push an event straight to the top level without needing it passed as a parameter.

```python
async def deeply_nested_custom_tool():
    push_event({"message": "I am modifying a local file!"})
    # Do work
    return True
```

## When to Use It
* When building **dynamic, multi-agent hierarchies** where sub-agents are instantiated on-the-fly and aren't natively wired into the parent LangGraph compile state.
* When executing **custom tools** (like a REPL prompt) where you want to stream exactly what code is being executed *before* the LLM evaluates the result.
* When standard `callbacks=[MyCallbackHandler()]` parameter dripping becomes overly burdensome to maintain.

## When NOT to Use It
* If you have a completely flat, single-agent LangChain workflow. In this case, simply running `for event in graph.astream_events():` and tracking `on_tool_start` is vastly simpler and recommended by LangChain directly.

---

## Real-World Example in This Codebase

In this application, the "Usecase Planner Agent" utilizes a `call_specialist()` function. Depending on the user's intent, it dynamically boots up a brand-new instance of an `ExploratoryDataAnalyser` or a `ModelingAgent` midway through execution. Because these sub-agents are invoked manually via `.ainvoke()`, the top-level Supervisor event stream went completely blank.

Additionally, the application uses a heavily modified `ToolNode` class to bind locally injected pandas Dataframes into a `jupyter_client` kernel or `PythonREPL`. 

To give the UI real-time visibility, `backend/app/services/event_streaming.py` introduces the `push_step_event()` function. Inside the `ToolNode` (`backend/app/agents/tools/tools.py`), the moment the agent attempts to run code, the system truncates the input code block and triggers `push_step_event("tool_start", "jupyter.exec", code, "running")`. The top-level SSE iterator constantly monitors the `ContextVar` queue, intercepting these custom execution milestones and forwarding them directly to the React `ThinkingProcess` component, completely bypassing LangGraph's internal streaming limitations.
