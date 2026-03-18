# Human-in-the-Loop (HITL) Checkpoints

## What is it?
Fully autonomous AI systems are dangerous. If an agent hallucinates a destructive choice (like running `DROP TABLE users` in a database, or sending a wildly offensive email to a thousand customers), the business impact is severe.

**Human-in-the-Loop (HITL)** architecture forces the agentic workflow to halt execution right before a high-risk action, serialize its current state (saving its memory), and wait for active human authorization (clicking "Approve" or typing modifications) before resuming logic execution. 

## How to Implement it

### Strategy 1: The Native LangGraph Checkpointer (`interrupt_before`)
LangGraph natively supports pausing state graphs using a persistent checkpointer (like SQLite or Postgres).
```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()

# Build the graph
workflow.add_node("draft_email", draft_node)
workflow.add_node("send_email", send_node)
workflow.add_edge("draft_email", "send_email")

# Compile with an explicit interrupt boundary
app = workflow.compile(
    checkpointer=memory,
    interrupt_before=["send_email"] # <-- HALTS BEFORE THIS NODE
)

# 1. Run the Graph (It will pause after drafting the email)
app.invoke({"messages": "Write an email to John"}, config={"thread_id": "1"})

# ... UI waits for human to simply click an "Approve" button ...

# 2. Resume the Graph (Pass `None` to just tell it to continue identically)
app.invoke(None, config={"thread_id": "1"}) 
```

### Strategy 2: The Hard-Break Architectural Approach
Instead of relying on internal graph memory pointers, you explicitly design the Multi-Agent system to hit an `END` node after generating a proposition. The frontend receives the JSON proposition, displays it in a custom UI block, and wait. The "Approval" button simply fires a brand new HTTP request directly into the *Execution* Graph logic.

## When to Use It
* **Destructive Tool Usage:** Before an agent executes irreversible API calls (Sending emails, deploying servers, deleting rows).
* **Lengthy/Expensive Executions:** Before an agent kicks off a 30-minute background job analyzing 10 gigabytes of Snowflake data, a human should quickly check if the agent's logic Plan is correct so compute isn't wasted.

## When NOT to Use It
* **Safe Exploratory Actions:** Do not interrupt the workflow before every single `select * from table` query or internal mathematical calculation. The system becomes a tedious "click-to-continue" simulator, destroying the value of AI autonomy. 

---

## Real-World Example in This Codebase

In this application, executing complex data-science pipelines (EDA → Recommendation → Modeling) can take massive amounts of time and backend compute. Therefore, the application strictly utilizes the **Hard-Break** strategy to enforce human approval.

In `backend/app/agents/supervisor_agent.py`, when a user makes a complex request, the Supervisor delegates to the `usecase_plan_generator_node`. When the Planner finishes drafting the execution plan, control returns to the Supervisor. 

Instead of automatically looping directly into the Executor node, the Supervisor intercepts the state:
```python
        has_generated_plan = False
        if any(key in outcome_data for key in ['plan_id', 'tasks', 'execution_plan']):
            has_generated_plan = True
```

The Supervisor modifies its prompt dynamically:
> *"IMPORTANT: The previous agent responses contains a generated execution plan. You MUST display the plan details clearly to the user and ask if they want to proceed. Set next_agent = 'END'."*

Because the Router hits `END`, the WebSocket SSE stream completely disconnects from the Python backend. The React frontend natively renders the JSON plan into a specialized UI Component exposing the AI's internal task breakdown. 

The pipeline remains entirely frozen indefinitely. It is only when the human clicks the **Run Plan** button that React hits the backend API again, packaging the approved state directly into `usecase_plan_executor`—safely crossing the execution boundary into a Background Worker subprocess.
