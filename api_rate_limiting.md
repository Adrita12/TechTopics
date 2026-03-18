# API Rate Limiting for LLM Applications

## What is it?
Rate limiting is a defensive mechanism that restricts how many times a single user (usually identified by their IP address or API token) can hit a specific server endpoint within a given timeframe (e.g., 15 requests per minute).

In modern LLM applications, building a robust rate limiter is no longer an optional feature; it is mandatory. Without it, a malicious user or an accidental frontend `useEffect` infinite loop could trigger 5,000 text-generation requests a minute. This will instantly exhaust the LLM provider quotas (OpenAI/Anthropic/Google), driving up massive cloud compute bills, and permanently crashing your application to all other legitimate users.

## How to Implement it

### The SlowAPI Implementation (FastAPI Example)
SlowAPI is the native, ultrafast integration of the `limits` python package specifically engineered for FastAPI. It tracks the limitations directly in server-side memory (`MemoryStorage`) or Redis.

1. **Setup the Limiter Configuration**
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# Uses the user's IP address as the unique identifier key
limiter = Limiter(key_func=get_remote_address)
```

2. **Protecting Routes (Decorating)**
```python
from fastapi import Request

@app.post("/chat/generate")
@limiter.limit("15/minute") # Only 15 executions per IP, per 60 seconds
async def generate_response(request: Request, payload: dict):
    # If the user hits 16 requests, this internal code never executes. 
    # SlowAPI instantly bounces the request with an HTTP 429 response.
    return await heavy_llm_execution(payload)
```

## When to Use It
* **LLM Generation Endpoints:** Any route that directly incurs a cost per token.
* **Database and Search Indexes:** Routes executing massive `GROUP BY` aggregations or complex RAG cross-encoding matrix searches. 
* **File Uploads:** Upload endpoints should be aggressively throttled to prevent bad actors from saturating your server's Network Bandwidth and local Disk I/O.

## When NOT to Use It
* **Health Check Pings:** Endpoints like `GET /ping` or AWS Target Group health verification shouldn't be limited, as load balancers poll them constantly.
* **Internal Microservice Routes:** If your internal Kubernetes cluster communicates between worker pods using local network routes, do not limit them based on the `127.0.0.1` IP address.

---

## Real-World Example in This Codebase

In this application, the initial implementation scaled beautifully out to dozens of users asynchronously, until a single bug in a Javascript React hook caused a browser's polling interval to collapse to zero—sending thousands of requests a minute at the `/chat` route.

The resulting API crash prompted the creation of `fix_doc/api_rate_limiting.md`. 
The application fully decoupled the `Limiter` engine into a specialized configuration loop, attaching it statically to the FastAPI `app.state`. 

Granular, hyper-specific limits were drawn:
1. **Generative LLM Chat:** `15 requests / minute`. (1 request every 4 seconds is virtually impossible for a legitimate human reading responses, but perfectly prevents botting spam from destroying the Google GenAI quota).
2. **File Processing:** `5 requests / minute`. This strictly chokes excessive local Storage Space I/O when parsing 50-megabyte CSV uploads. 
3. **Global Sub-Routing:** `60 requests / minute` safety net.

Crucially, because SlowAPI is "In-Memory", checking an IP against a threshold takes microseconds. If the user violates the 15-request boundary, SlowAPI throws a `429 Too Many Requests` *before* the backend unpacks the JSON or contacts the GPU servers. This ultra-cheap gateway validation is what protects the expensive core logical operations from crashing under illegitimate traffic.
