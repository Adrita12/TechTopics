# Background Jobs & Polling Mechanisms

## What is it?
A Background Job is an asynchronous computing pattern where heavy, time-consuming tasks are offloaded from the main web server thread to a specialized background worker process. The web server immediately returns a "Job ID" to the user, and the frontend routinely "polls" (asks the server) about the status of that Job ID until it finishes.

This prevents the web server from freezing or dropping HTTP connections due to load balancer timeouts (e.g., 504 Gateway Timeout).

## How to Implement it

### 1. The Database Schema
You need an external state store (usually Postgres or Redis) to act as the single source of truth for the job's progress.
```sql
CREATE TABLE background_jobs (
    id UUID PRIMARY KEY,
    status VARCHAR(50) DEFAULT 'running', -- running, completed, failed
    created_at TIMESTAMP
);
CREATE TABLE job_steps (
    job_id UUID,
    step_description TEXT,
    status VARCHAR(50)
);
```

### 2. The Worker Backend (Subprocess or Celery)
When the API receives a request, it inserts a new job into the database and spawns a background thread or a completely different Python process to do the actual work.
```python
# The API Endpoint
@app.post("/run-heavy-task")
async def start_task(background_tasks: BackgroundTasks):
    job_id = create_job_in_db()
    # Pass execution to background
    background_tasks.add_task(run_analytics, job_id)
    return {"job_id": job_id}

# The Background Worker
def run_analytics(job_id: str):
    db.update_status(job_id, "running")
    # ... do 10 minutes of heavy processing ...
    db.insert_step(job_id, "Data cleaned")
    db.update_status(job_id, "completed")
```

### 3. The Frontend Polling (React Component)
The frontend uses a `setInterval` or recursive `setTimeout` to constantly query the backend for updates on the `job_id`.
```typescript
useEffect(() => {
    if (!activeJobId) return;

    const poll = async () => {
        const res = await fetch(`/api/jobs/${activeJobId}`);
        const data = await res.json();
        
        setSteps(data.steps);
        
        if (data.status === 'completed') {
            setActiveJobId(null); // Stop polling
        } else {
            setTimeout(poll, 3000); // Check again in 3 seconds
        }
    };
    
    poll();
}, [activeJobId]);
```

## When to Use It
* **Long Running AI Workflows:** Any task taking longer than 30-60 seconds (like running extensive EDA analysis, rendering 50 charts, or scraping websites).
* **Heavy Data Processing:** Uploading parsing gigabytes of CSV data.
* **Email Campaigns:** Sending thousands of notifications where blocking the user's browser isn't acceptable.

## When NOT to Use It
* **Instantaneous queries:** Standard database lookups or fast (1-2 second) LLM text generations.
* **High-frequency updates:** If you need millisecond-level precision updates, SSE or WebSockets are better suited than an HTTP polling mechanism which adds constant overhead.

---

## Real-World Example in This Codebase

In this application, executing complex analytical plans via LangGraph can take anywhere from 1 to 5 minutes. An HTTP request would time out. 

The application utilizes a **Dual-Mode Executor Node** design:
1. When a user approves an AI plan, the `supervisor_agent.py` kicks into action. Instead of running the entire workflow locally, the Supervisor returns a "Signal Payload": `{"background_job_signal": {"job_id": "123"}}`.
2. The FastAPI endpoint intercepts this signal, immediately closes the HTTP connection to the frontend by emitting a `background_job_start` SSE event, and uses `subprocess.Popen("python backend/jobs/run_executor.py")` to launch an entirely isolated worker process.
3. The newly spawned Python script (`run_executor.py`) physically spins up LangGraph in a distinct process, utilizing a `JobStepLogger` to write every thought, Python REPL evaluation, and pandas error directly into PostgreSQL using `psycopg2`.
4. Over on the React frontend (`Chat.tsx`), upon receiving the `background_job_start` event signal, a `useEffect` hook transitions from listening to an SSE stream into active polling mode. It hits `GET /jobs/{job_id}/steps` every 5 seconds, reading the Postgres rows that the worker is actively writing, and animating the `ThinkingProcess` UI components until the job row reads `completed`.
