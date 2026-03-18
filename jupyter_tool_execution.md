# Jupyter Tool Code Execution 

## What is it?
When an LLM (Language Model) is tasked with exploring data, parsing spreadsheets, or building predictive models, producing "theoretical" Python code is useless. The code must be actively executed against the local dataset natively to produce a true answer.

Traditional setups use a native `PythonREPL` (evaluating strings using `exec()`). However, this is dangerous, brittle, and lacks true state isolation. The modern standard is configuring your framework to execute strings natively through an isolated **Jupyter Kernel**, the exact same backend engine that powers analytical Notebooks.

## How to Implement it

### 1. Launch a Stateful Kernel
Using the `jupyter_client` python package, you initialize an independent IPython environment. This environment runs as a standalone process. If the AI writes a syntax error or a devastating infinite loop (`while True:`), your main FastAPI server does not crash.

```python
import jupyter_client

class StatefulJupyterTool:
    def __init__(self):
        # Starts a background python process
        self.km, self.kc = jupyter_client.manager.start_new_kernel(kernel_name='python3')
        
    def execute_ai_code(self, code_string: str) -> str:
        # Run code as if you hit Shift-Enter in a notebook
        self.kc.execute(code_string)
        # Fetch output from ZMQ socket
        reply = self.kc.get_iopub_msg(timeout=10)
        return reply['content']['text']
```

### 2. Injecting Local State (Dataframes)
One massive advantage of Jupyter kernels is state retention. You can serialize local data from your Web Server directly into the kernel's RAM memory so the AI can use them.

```python
    def inject_variable(self, var_name: str, df: pd.DataFrame):
        import pickle
        # Serialize the df heavily into bytes, write to /tmp, 
        # and tell the kernel to unpickle it
        self.kc.execute(f"import pickle\n{var_name} = pickle.load(open('/tmp/df.pkl', 'rb'))")
```

### 3. Capturing Plot Artifacts (Charts)
Because Jupyter integrates natively with visualization libraries (Plotly, Matplotlib), you can instruct the kernel execution tool to intercept Canvas renders and convert them into raw JSON schemas perfectly formatted for your React frontend to draw. 

## When to Use It
* **Data Science Platforms:** Whenever allowing agents to slice and dice user `.csv` records. A distinct Jupyter engine ensures variable states overlap properly between multi-tool invocations.
* **Code Sandboxing:** Running untrusted AI code through a restricted Jupyter kernel inside a Docker container vastly limits security footprint damage.

## When NOT to Use It
* **Basic Calculators:** If you just want an LLM to add large integers or do simple string manipulations, spinning up an entire ZMQ IPC socket kernel backend is ridiculously heavy overhead. Use lightweight `subprocess.run` or strict `numexpr` evaluators.

---

## Real-World Example in This Codebase

The architecture of this application specifically relies heavily on `USE_JUPYTER_KERNEL` configurations inside `backend/app/agents/tools/tools.py`. 

1. **Jupyter Fallback execution:** If enabled, the `ToolNode` creates a specialized `JupyterPythonTool` rather than employing the standard `PythonREPL`. 
2. **Path Sanitization Shield:** Because LLMs frequently hallucinate their own environment variables (assuming data lives inside generic `/path/to/file.csv`), the `tools.py` intercepts the code explicitly utilizing Regex: `sanitize_code_paths(code: str) -> str`. It guarantees absolute paths are surgically downgraded into universally functioning local basenames matching the Docker mount configurations.
3. **Dataframe Injection:** If the `ToolNode` initializes with an array of parsed Pandas objects, it immediately calls `executor.inject_variable("df_0", df)`. The Jupyter engine unpacks this implicitly, meaning the AI merely needs to natively write `print(df_0.head())` without ever worrying about File I/O loading sequences!
4. **Intelligent Artifact Capture:** Natively, when the `jupyter_client` runs code calling Plotly, it doesn't just print output logs to `stdout`. The tool hooks explicitly into the IPython Display Publisher sockets, searching for JSON-based graphics. It intercepts the Plotly objects, rips them out of the kernel as standard Javascript array configurations, and appends them to a generated `artifacts` array. This perfectly clean schema is seamlessly passed back through the API directly over to the `<LazyPlot />` React wrapper logic to be natively drawn.
