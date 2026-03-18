# Supervisor Agent Pattern

## What is it?
In early LLM application design, developers often tried to create a single "God Agent"—one massive prompt that contained instructions for coding, querying databases, searching the web, evaluating finances, and writing poetry. This fails spectacularly at scale because the LLM gets confused, forgets rules, and hallucinates tools it shouldn't use.

The **Supervisor Agent Pattern** acts like a corporate hierarchy. You build one "Manager" (The Supervisor). The Supervisor's *only* job is to listen to the user, figure out what needs to be done, break the work down, and delegate it to narrow, highly constrained "Specialist Agents" (The Workers). It never does the heavy lifting itself.

## How to Implement it

### 1. Define the Specialists
Create individual agents that have very specific tools and extremely focused system prompts.
```python
# A worker that ONLY writes SQL
data_analyst = create_specialist(
    system_prompt="You are an analyst. You only write SQL.",
    tools=[sql_query_tool]
)

# A worker that ONLY writes Marketing text
copywriter = create_specialist(
    system_prompt="You are a marketer. You only write emails.",
    tools=[email_draft_tool]
)
```

### 2. Force the Supervisor to Choose (Structured Outputs)
The Supervisor LLM must not be allowed to chat openly. You constrain its output using Pydantic so it *must* return a valid JSON object indicating which worker to call next.

```python
from pydantic import BaseModel
from typing import Literal

class SupervisorDecision(BaseModel):
    next_worker: Literal["data_analyst", "copywriter", "FINISH"]
    reasoning: str

supervisor = llm.with_structured_output(SupervisorDecision)
```

### 3. The LangGraph Router
You compile this into a Graph. The Supervisor sits at the center, fanning out to the workers. After a worker finishes, control returns back to the Supervisor to review the work and decide if it needs to call a different worker.

```python
def supervisor_node(state):
    decision = supervisor.invoke(state["messages"])
    return {"next_step": decision.next_worker}

# Conditionally route based on the supervisor's strict JSON output
workflow.add_conditional_edges("Supervisor", lambda x: x["next_step"])
```

## When to Use It
* **Complex Multi-Step Goals:** Whenever a user request requires math, then code execution, then internet research, then summarization. 
* **Tool Overload:** If your application has more than 5 to 7 tools. An LLM's accuracy drops sharply if it has to choose from dozens of confusing tools simultaneously. Splitting them among specialists fixes this.

## When NOT to Use It
* **Latency Critical Systems:** Because the Supervisor has to "think", then pass the torch to a Worker, then "think" again to review the work, you inherently double or triple the LLM generation time. If the user needs an answer in 2 seconds, do not use a recursive multi-agent hierarchy.
* **Simple Q&A:** Pure RAG chatbots do not need supervisors.

---

## Real-World Example in This Codebase

The heartbeat of this entire application is defined in `backend/app/agents/supervisor_agent.py`. It is a pure embodiment of this pattern.

The user's prompt first hits the `supervisor_node`. The LLM evaluates the state and uses a rigid `SupervisorDecision` Pydantic model to choose from three distinct actions:
1. `generate_plan`: It decides to delegate to the `usecase_plan_generator` node.
2. `execute_plan`: It decides to delegate to the `usecase_plan_executor_node`, which internally spawns highly specialized agent subclasses (`ExploratoryDataAnalyser`, `ModelingAgent`, `SimulationAgent`, `RecommendationAgent`, etc.). Each specialist is strictly equipped with their own specific tools (`PythonREPL` with Dataframes vs mere text manipulation).
3. `FINAL_ANSWER`: It decides the job is done and replies directly.

By strictly separating the "Manager" logic (deciding *what* to do) from the "Worker" logic (running Pandas code to evaluate a CSV), the application completely avoids prompt-confusion. The EDA Agent doesn't have to worry about accidentally triggering an API simulation tool, because the Supervisor fundamentally guarantees the EDA Agent is only ever invoked inside isolated data-exploration environments.
