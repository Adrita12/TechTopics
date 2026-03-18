# Agent Response Schemas & Structured Outputs

## What is it?
By default, large language models (LLMs) return raw, heavily conversational markdown strings (e.g., *"Sure, here are the top 3 items you asked for: 1. Apple, 2..."*). 

This is useless for programmatic backend code. If your backend router expects an array array elements to trigger an API lookup, it cannot parse *"Sure, here are..."*. 

**Structured Outputs** force the LLM to abandon conversational prose and strictly return valid, predictably typed JSON conforming to 100% rigid architectural specifications provided by the developer.

## How to Implement it

### 1. The Pydantic Schema Model
You must declare exactly what information you want the LLM to generate using standard Python Pydantic `BaseModel` classes. You should include deeply descriptive documentation (`Field(description=...)`), because modern LLMs natively read these field descriptions as implicit prompting instructions.

```python
from pydantic import BaseModel, Field
from typing import List

# Define the precise schema
class RecipeExtraction(BaseModel):
    title: str = Field(description="The formal title of the recipe.")
    ingredients: List[str] = Field(description="Strict list of raw ingredients only.")
    prep_time_minutes: int = Field(description="Total prep time mapped strictly to an integer.")
    difficulty: str = Field(description="Must be exactly: 'EASY', 'MEDIUM', or 'HARD'.")
```

### 2. LangChain `with_structured_output` bindings
LangChain provides an out-of-the-box wrapper for most modern LLMs (OpenAI, Gemini, Anthropic) utilizing their native Function Calling APIs to strictly enforce the provided Pydantic schema geometry.

```python
from langchain_openai import ChatOpenAI

# 1. Initialize empty model
llm = ChatOpenAI(model="gpt-4o")

# 2. Bind the schema natively into the LLM
structured_llm = llm.with_structured_output(RecipeExtraction)

# 3. Invoke it
raw_user_prompt = "I want to bake a quick 15 minute chocolate cake. Very hard."
result = structured_llm.invoke(raw_user_prompt)

# Returns a rigorously typed Python object, NOT a string!
print(result.ingredients)         # ["chocolate", "flour", ...]
print(result.prep_time_minutes)   # 15
print(result.difficulty)          # "HARD"
```

## When to Use It
* **Multi-Agent Routing:** When a Supervisor Agent needs to logically parse `{"next_node": "math_agent", "reasoning": "User asked for algebra"}` to determine exactly where to route a user's task.
* **Data Extraction AI:** When ingesting a generic block of text (like thousands of disparate resumes) and formatting it into a rigid database (Extracting `{name: "", email: "", years_experience: ""}`).

## When NOT to Use It
* **Final Conversational Synthesis:** The final node in your workflow that actually replies to the human user should *not* be structured. Users want friendly, markdown-rich chat.
* **Code Generation:** Forcing an AI to output code blocks into JSON string parameters (`{"code": "def run():\n..."}`) frequently causes nested quote escaping errors `\` `"` inside JSON parsers when running massive syntax models. Let code strings be code strings.

---

## Real-World Example in This Codebase

In this application, complex agentic behavior is impossible without deterministic JSON bindings.

In `backend/app/agents/helpers/pydantic_models.py`, there are massive schemas explicitly guiding the multi-agent router's thought process.

For instance, the Application uses the `SimulationAgent` to evaluate hypotheticals. How does the Supervisor definitively understand what hypothetical tests the human wants to trigger? Through the `SimulationScenario` payload.
```python
class SimulationScenario(BaseModel):
    name: str = Field(description="A short, descriptive name for the scenario, e.g., '10% Price Increase'")
    variables_to_change: List[Dict[str, float]] = Field(description="List of dicts specifying feature name and new value")
    rationale: str = Field(description="Explain WHY this simulation is being run.")
```

When building the overarching `supervisor_agent.py`, the system locks the model: `orchestrator = llm.with_structured_output(SupervisorDecision)`. This `SupervisorDecision` model rigidly enforces the `{"action": "execute", "target_agent": "EDA", "task": ""}` keys. 

Before this implementation, the Supervisor would hallucinate actions, occasionally outputting XML `<thought>` tags or dumping its rationale to standard out before picking a task. By locking it via `with_structured_output()`, LangGraph is entirely guaranteed to receive a perfectly formatted `action` and `target_agent` property to ingest in its edge routing logic, allowing Python logic loops to iterate securely without crashing.
