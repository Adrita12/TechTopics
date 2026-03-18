# Caching Strategies (Disk-Based Chat History)

## What is it?
When interacting with complex Multi-Agent LLM architectures, conversations often generate massive payloads (e.g., executing a Pandas Dataframe might yield 50,000 tokens of raw `.info()` schema data).

If you feed the literal, full conversation chain (containing 50,000-token messages) into the LLM on every back-and-forth prompt, three catastrophic things happen:
1. **Latency:** The model takes 10+ seconds just to read its own context window.
2. **Cost:** You are paying for those 50k tokens again and again and again.
3. **Hallucination:** The model suffers from "lost in the middle" syndrome, getting completely distracted by the archaic raw noise and forgetting the user's actual question.

Caching allows you to truncate historical noise by selectively wiping heavy data from the AI's prompt string and replacing it with a tiny pointer reference, writing the actual massive payload to a temporary file locally.

## How to Implement it

### 1. Identify Heavy Payloads and Substitute
When an agent successfully completes a heavy task that isn't strictly necessary for surface-level conversation, intercept the output array before it gets permanently added to the `messages` history.

```python
import uuid
import json

def truncate_heavy_history(raw_output: str) -> str:
    if len(raw_output) > 2000:
        # Generate a unique cache ID
        cache_id = str(uuid.uuid4())
        
        # Save payload to local hard disk
        with open(f"/tmp/cache/{cache_id}.json", "w") as f:
            json.dump({"payload": raw_output}, f)
            
        # Replace the massive payload in the Chatbot's memory 
        # with a tiny 20-token "Sticky Note" pointer
        return f"[SYSTEM: Heavy data truncated. Full details in Cache ID: {cache_id}]"
        
    return raw_output
```

### 2. Equip the Supervisor with an Un-Truncate Tool
Because you wiped the data from the prompt, the AI literally no longer knows it. If the human asks a highly specific question referencing that old data, the AI *must* be given a dynamic Python tool to look up its own sticky notes.

```python
@tool
def retrieve_full_agent_details(cache_id: str) -> str:
    """If your conversation history says [SYSTEM: Full details in Cache ID: XYZ], 
    use this tool with XYZ to fetch the raw data."""
    try:
        with open(f"/tmp/cache/{cache_id}.json", "r") as f:
            data = json.load(f)
            return data["payload"]
    except Exception:
        return "Cache not found."
```

## When to Use It
* **Deep Code Iteration:** When logging full source code execution trails, stack traces, and data summaries that aren't critical to the immediate user conversational loop.
* **Cost Compression:** If your LLM charges per input token, keeping the active context window aggressively clean saves exponential dollars.

## When NOT to Use It
* **Short Tasks:** If the conversation history is naturally brief and doesn't contain massive tool outputs (less than 4,000 tokens total), the network latency of loading a JSON from disk is slower than just letting the LLM read the context naturally. 

---

## Real-World Example in This Codebase

As documented in `fix_doc/chat_history_caching.md`, when the user executed exploratory analytical sessions, specialist agents generated incredibly vast reporting blocks (JSON). When fed blindly back into the overarching Supervisor Agent's context window, it repeatedly blew past the LLM's absolute token limit.

To repair this, the architects implemented a completely dedicated local server filing cabinet located at `uploads/.memory_cache`.

Whenever an enormous dataset emerges, the application removes it from the LangGraph `messages:` context queue. Instead, it drops a simple "Sticky Note" (e.g., `[SYSTEM: Payload truncated. Details in cachexyz123]`) into the Supervisor's prompt memory to preserve the *fact* that execution happened, without forcing the Supervisor to memorize the *content* of that execution.

If the human user subsequently asks *"Wait, what was the exact mathematical variance 10 messages ago?"*, the Supervisor invokes the custom `retrieve_full_agent_details` tool. The Python execution intercepts this ID, pulls the heavy `.json` payload out of the local disk directory, injects it temporarily back into a hidden system prompt, allows the LLM to answer the user perfectly, and then unloads it back out of memory instantly.
