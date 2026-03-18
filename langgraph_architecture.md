# LangGraph Architecture & State Design

## What is it?
LangGraph is an extension of LangChain designed natively for building stateful, multi-actor applications with LLMs. 

Traditional LangChain chains are "Directed Acyclic Graphs (DAGs)" — they execute Step A, then Step B, and finish. **LangGraph explicitly supports cyclic operations** (loops, conditionals, human-in-the-loop approvals), allowing you to build complex "Agentic" workflows where a generic AI acts as a sophisticated Supervisor, delegating specific coding, research, or plotting tasks to sub-agents, reviewing their work iteratively, and looping backward if a test fails.

## How to Implement it

### 1. Define the Immutable State (The TypedDict)
Everything in LangGraph revolves around the **State**. The state is a Python `TypedDict` or Pydantic `BaseModel` that is passed from node to node. When a node finishes, it doesn't overwrite the state—it returns an *update* to the state which LangGraph merges.

```python
from typing import TypedDict, Annotated, List
import operator

# State defines the exact variables every single node has access to.
class GraphState(TypedDict):
    messages: List[str]
    # Annotated with operator.add means nodes APPEND to the list, not overwrite.
    tool_results: Annotated[List[str], operator.add] 
```

### 2. Define the Nodes (The Actors)
A node is simply a Python function that takes the `State` as input, runs some LLM generation or tool logic, and returns a dictionary of state updates.

```python
def research_node(state: GraphState):
    llm = ChatOpenAI()
    answer = llm.invoke(f"Research this: {state['messages']}")
    return {"tool_results": [f"Research: {answer.content}"]}
```

### 3. Define the Edges (The Router Logic)
Edges dictate exactly which node executes next. "Conditional Edges" use the state to make dynamic choices.

```python
def router(state: GraphState):
    # Conditional logic
    if len(state["tool_results"]) > 3:
        return "END"
    return "research_node"
```

### 4. Compile the Graph
```python
from langgraph.graph import StateGraph, START, END

# Initialize
workflow = StateGraph(GraphState)

# Add nodes
workflow.add_node("research_node", research_node)
workflow.add_node("drafting_node", drafting_node)

# Add execution edges
workflow.add_edge(START, "research_node")
# Loop conditionally!
workflow.add_conditional_edges("research_node", router, {
    "research_node": "research_node", # Loop back
    "END": "drafting_node"            # Move forward
})
workflow.add_edge("drafting_node", END)

# Compile into executable
app = workflow.compile()
```

## When to Use It
* **Agentic Workflows:** When building complex intelligent systems that require "Thinking", "Acting", and "Reviewing" (loops).
* **Multi-Agent Topologies:** When building a Supervisor -> Worker hierarchy.

## When NOT to Use It
* **Simple Q&A Chatbots:** If you are just taking a prompt, stuffing it into an LLM window, and returning the output to the user, standard LangChain (LCEL `prompt | model | parser`) is infinitely simpler.
* **Zero loops:** If your logic is strictly sequential (Extract -> Transform -> Load) with zero dynamic LLM conditional routing, building a Graph is excessive overhead.

---

## Real-World Example in This Codebase

This application features an incredibly robust hierarchical Multi-Agent Topology (orchestrated via `backend/app/agents/supervisor_agent.py` and `usecase_planner_agent.py`):

1. **The Shared Schema:** Built on a unified payload object (`PlannerState`), guaranteeing every worker sees the same baseline history variables (`messages`, `tool_calls[]`, `scratchpad`).
2. **Supervisor Routing Node:** The entry point `supervisor_node` evaluates the human's input and invokes an LLM strictly bound with structured outputs to decide if it should generate a detailed execution Plan (`"usecase_plan_generator"`) or if it should execute a previously created plan (`"usecase_plan_executor"`).
3. **Execution Deep Nesting:** If routed to the Executor, the node (`usecase_plan_executor_node`) evaluates the Plan JSON. Inside this node, it determines which specialist is required. It instantiates fresh LangChain subclasses—like the `ExploratoryDataAnalyser` (EDA) or the `ModelBuildingAgent`—passing them highly constrained persona prompts and binding exactly customized Python sandbox toolsets directly to their `tool_node` instances.
4. **Infinite Loop Prevention:** LangGraph topologies are prone to catastrophic infinite retries. In legacy codebase history (`supervisor_infinite_loop_fix.md`), the routing logic lacked incrementing recursion depth bounds. If a Python execution tool natively threw a `SyntaxError`, the Agent would re-invoke the tool unconditionally in a tight circle. The architecture was upgraded to implement hard state breaks `tool_calls_needed = False` if errors exceeded threshold limits.
