# Plot Rendering & UI Latency Optimization

## What is it?
In data science React applications, rendering heavy visualizations (like Plotly charts containing 10,000+ scatter points) forces the browser to engage in massive DOM/Canvas paint calculations. This aggressively hijacks the browser's "Main Thread." 

If an AI streams 5 complex charts simultaneously, the user's browser completely locks up (a "UI Freeze"). Buttons become unresponsive, text streaming stops, and the page crashes. Solving this requires strict Data Downsampling on the backend, and Deferred "Lazy" Rendering on the frontend.

## How to Implement it

### 1. Backend: Data Downsampling
You must never trust an AI / Python script to send safe payloads. You must intercept massive datasets (`>5000 points`) and randomly sample them down before sending JSON over the network.

```python
import random

def optimize_plot_for_streaming(fig_dict):
    """Intercepts a Plotly JSON schema and shrinks massive arrays."""
    max_safe_points = 5000
    
    for trace in fig_dict.get("data", []):
        if "x" in trace and len(trace["x"]) > max_safe_points:
            # Randomly sub-sample down to 5000 points
            indices = sorted(random.sample(range(len(trace["x"])), max_safe_points))
            trace["x"] = [trace["x"][i] for i in indices]
            trace["y"] = [trace["y"][i] for i in indices]
            
            # Warn the user
            trace["name"] = str(trace.get("name", "")) + " (sampled)"
            
    return fig_dict
```

### 2. Frontend: Deferred "Lazy" Rendering
When the heavy Javascript payload arrives, do not instantly push it into the `<Plot>` component. Instead, render a Skeleton component, and use `requestAnimationFrame` and `setTimeout` to push the heavy math to the absolute back of the browser's priority queue, rendering them one-by-one.

```tsx
import React, { useState, useEffect } from "react";
import Plot from "react-plotly.js";

const LazyPlot = ({ data, layout }) => {
  const [shouldRender, setShouldRender] = useState(false);

  useEffect(() => {
    // 1. Stagger executions so 5 plots don't render simultaneously.
    const delay = 50 + Math.random() * 200; 
    
    const timer = setTimeout(() => {
      // 2. Wait until the browser finishes all its current repaints.
      requestAnimationFrame(() => {
        setShouldRender(true);
      });
    }, delay);

    return () => clearTimeout(timer);
  }, [data]);

  if (!shouldRender) {
    return <div className="skeleton-graph">Loading visualization...</div>;
  }

  return <Plot data={data} layout={layout} />;
};

export default React.memo(LazyPlot);
```

## When to Use It
* **Data Exploration (EDA) AI Agents:** Whenever an AI agent utilizes pandas / seaborn / plotly to generate charts dynamically against unverified, massive user datasets.
* **Analytics Dashboards:** Where dozens of individual charts are rendered aggressively on one page view.

## When NOT to Use It
* **Server-side Rendering:** If using libraries that generate static `.png` or `.svg` files on the backend instead of shipping interactive Javascript objects. Static images do not block the main thread.
* **Bar Charts or small aggregates:** Rendering a 10-column aggregate bar chart takes 1ms; `requestAnimationFrame` is complete overkill.

---

## Real-World Example in This Codebase

In this application, standard operating procedure for the `eda_agent` was to consume user CSV files and print out Plotly datasets. When users uploaded 50 megabyte logs, the AI would enthusiastically return 5 distinct, 1-million-point Scatterplots.

The `agent_service.py` stream iterator blindly forwarded these 5-million point JSON blobs into the single React `Chat.tsx` context block. Because React rerenders the chat feed synchronously as new text tokens arrive, combining an ultra-heavy `ChartRenderer` mount loop simultaneously with rapid DOM stream updates instantly destroyed the Chrome browser tab.

To optimize UI latency (as heavily documented in `doc/05_PLOT_SYSTEM.md`):
1. **The Throttler:** The Backend specifically implements Stream Throttling (`await asyncio.sleep(0.5)`) between yielding sequential plot artifacts in SSE to prevent data flooding.
2. **The React Shield:** Plotly execution was completely detached from the chat component via the `LazyPlot` wrapping container. The `LazyPlot` catches the massive JSON payload, displays a loading skeleton, assigns an asynchronous `Math.random() * 150` millisecond delay via `setTimeout`, and forces a `requestAnimationFrame` queue so the browser prioritizes streaming the AI's textual "Thinking" explanation *before* processing the rigid geometric Canvas draw. `React.memo` caps it off to ensure the expensive operation never re-fires on component un-mounts.
