# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: [Hoàng Anh Quyền]
- **Student ID**: [2A20200062]
- **Date**: 2026-04-06

---

## I. Technical Contribution (15 Points)

My main contribution in this lab was the **design and implementation of the stock tools layer** that allows the ReAct agent to ground its answers in stock data instead of responding only from model memory. I focused on defining tool interfaces that are simple for the LLM to call, useful for Vietnamese stock-analysis tasks, and compatible with the agent loop used in this project.

### Modules Implemented

- `src/tools/stock_tools.py`

### Code Highlights

#### 1. Designed the stock tool set for different task types
I helped build a small but practical tool inventory for common financial-agent workflows:

- `fetch_CafeF_stock(...)`: retrieve the latest 5 trading sessions for a ticker using a lightweight mock-data path.
- `fetch_FireAnt_stock(...)`: retrieve historical quotes over a date range from the FireAnt API.
- `compare_stocks(...)`: compare recent percentage performance between two tickers.
- `calculate(...)`: perform reusable numerical operations such as average, min, max, sum, and percentage change.

Relevant implementation:
- `src/tools/stock_tools.py:7-21`
- `src/tools/stock_tools.py:30-72`
- `src/tools/stock_tools.py:77-124`
- `src/tools/stock_tools.py:128-166`

I intentionally separated the tools into four roles:
- **data retrieval** (`fetch_CafeF_stock`, `fetch_FireAnt_stock`),
- **comparison logic** (`compare_stocks`),
- **numeric post-processing** (`calculate`).

This decomposition is useful because a ReAct agent works best when complex questions can be broken into smaller, explicit actions.

#### 2. Defined tool input schemas that are LLM-friendly
A major part of my contribution is designing **call formats** that the model could realistically follow.

Examples:
- single-string input for simple ticker lookup:
  - `fetch_CafeF_stock("FPT")`
- JSON-style input for structured tasks:
  - `fetch_FireAnt_stock({"ticker":"FPT","start_date":"2025-01-01","end_date":"2025-01-31"})`
  - `compare_stocks({"ticker1":"FPT","ticker2":"VNM"})`
  - `calculate({"operation":"average","values":[95000,93000,91000]})`

Relevant implementation:
- `src/tools/stock_tools.py:31-35`
- `src/tools/stock_tools.py:79-82`
- `src/tools/stock_tools.py:131-135`
- `src/tools/stock_tools.py:171-192`

I added explicit usage examples in both the docstrings and the registry descriptions because the ReAct loop depends on the model being able to produce machine-usable arguments. Better tool descriptions reduce the chance that the model generates vague or malformed actions.

#### 3. Added defensive parsing and error handling in the tool layer
Another important part of my work was making the tools more robust when the LLM produces imperfect arguments.

Examples of safeguards:
- JSON parsing guards with clear error messages when the input format is invalid.
- Required-key checks for fields such as `ticker`, `start_date`, and `end_date`.
- Type validation for numeric lists in `calculate(...)`.
- Timeout and request-exception handling for FireAnt API calls.
- Graceful handling of empty or missing historical data.

Relevant implementation:
- `src/tools/stock_tools.py:37-43`
- `src/tools/stock_tools.py:45-72`
- `src/tools/stock_tools.py:84-89`
- `src/tools/stock_tools.py:137-166`

This defensive design matters because, in an agent system, failures often do not come from Python syntax errors. They usually come from the mismatch between natural-language model output and strict tool-call requirements.

#### 4. Built a reusable tool registry for the agent runtime
I also contributed to exposing the stock tools through a registry structure that the agent can consume directly.

Relevant implementation:
- `src/tools/stock_tools.py:171-192`

The registry provides:
- tool `name`,
- tool `description`,
- bound Python `function`.

This is the bridge that lets the ReAct agent move from:
- **Thought** → choosing a tool,
- **Action** → calling the tool,
- **Observation** → receiving grounded data.

### Documentation

My contribution interacts with the ReAct loop in a very direct way. The agent in `src/agent/agent.py` parses an `Action: tool_name(args)` line and dispatches the call through `_execute_tool(...)`.

Relevant interaction points:
- `src/agent/agent.py:27-44`
- `src/agent/agent.py:85-105`
- `src/agent/agent.py:116-126`
- `src/tools/stock_tools.py:171-192`

From an architecture perspective:
1. the system prompt advertises available tools,
2. the model selects one tool name,
3. the agent parses the action,
4. my stock tool executes,
5. the returned result becomes the next `Observation`.

Because of this integration, the agent can answer questions like:
- “What is the recent price trend of FPT?”
- “Compare FPT and VNM over the latest sessions.”
- “Calculate the average closing price from these values.”

In short, my main contribution was turning the stock-analysis domain into **toolable actions** that the ReAct loop could actually use.

---

## II. Debugging Case Study (10 Points)

### Problem Description
A concrete issue I encountered while working on the stock tools was that the LLM did not always produce **valid tool arguments**, especially for tools that expected JSON-like input such as:

- `fetch_FireAnt_stock(...)`
- `compare_stocks(...)`
- `calculate(...)`

In some runs, the model generated malformed arguments, for example:
- missing quotes around keys,
- missing required fields such as `start_date`,
- mixing natural language with JSON,
- sending a ticker comparison request in an incorrect schema.

When this happened, the tool layer could not parse the input correctly and the agent failed to complete the task with grounded data.

### Log Source
`Day03_2A202600062\logs\2026-04-06.log`

This pattern is consistent with the validation logic in:
- `src/tools/stock_tools.py:37-43`
- `src/agent/agent.py:92-98`

### Diagnosis
The root cause was not that the stock API itself was broken. The deeper issue was that **tool calling requires stricter interfaces than ordinary chat generation**.

The model can produce fluent natural language even when it does not follow a strict schema. That is acceptable in chatbot mode, but it is a problem in agent mode because the tool layer needs structured inputs.

I identified three main causes:
1. the model was sometimes not strict enough when formatting JSON-like arguments,
2. the input schema for some tools was more complex than a simple ticker string,
3. without strong examples in the tool descriptions, the agent had less guidance on how to call the tool correctly.

### Solution
To improve this, I made the stock tool layer more agent-friendly in several ways:

1. **Added explicit examples in docstrings and registry descriptions** so the LLM can imitate a concrete call pattern.
2. **Added strict parsing and validation** with informative error messages rather than silent crashes.
3. **Separated simple and complex tools** so the agent has an easier path for common tasks:
   - simple lookup through `fetch_CafeF_stock(...)`,
   - richer date-range retrieval through `fetch_FireAnt_stock(...)`.
4. **Provided a reusable calculation tool** so the model does not need to do all arithmetic from scratch in text.

After these changes, the tool layer became more robust and much easier to debug. Even when the model still produced imperfect actions, the failure mode became visible and interpretable instead of hidden.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

### 1. Reasoning
From my perspective as the person who designed the stock tools, the biggest difference between a chatbot and a ReAct agent is that the agent can convert a financial question into an explicit sequence of operations.

A chatbot usually answers directly from its internal knowledge, which may sound fluent but is often unreliable for market data. A ReAct agent, by contrast, can reason like this:
- first decide whether external data is needed,
- then call a retrieval tool,
- then compare or calculate,
- then answer from the observation.

The `Thought` step is valuable because it helps the model decide **which tool is needed next**, not just what sentence to write.

### 2. Reliability
The agent is not always better than the chatbot. In some situations, it can be worse:
- when the question is simple and does not actually need a tool,
- when the model formats the tool call incorrectly,
- when an external API times out,
- when the tool schema is too strict for the model to follow consistently.

So the agent is more capable, but also more fragile. Building tools taught me that reliability in an agent system depends not only on model quality, but also on interface design.

### 3. Observation
The most important advantage of the agent is the **Observation** step. Once a stock tool returns real values, the next reasoning step is no longer based on guesswork.

This changes the quality of the answer in three ways:
- the response becomes grounded in retrieved data,
- calculations become reproducible,
- debugging becomes easier because the full reasoning path is visible.

For stock analysis, this feedback loop is essential. A financial assistant without observation is mostly a language generator; a financial assistant with observation begins to behave like a true agent.

---

## IV. Future Improvements (5 Points)

### Scalability
If I continued this project, I would evolve the stock tool layer into a more modular architecture:
- split data providers into separate files,
- add a standardized schema per tool,
- support more providers beyond Cafef and FireAnt,
- add a router that chooses the right tool based on intent and time range.

### Safety
There are several safety and engineering improvements I would prioritize:
- move the FireAnt bearer token out of source code and into environment variables,
- validate ticker symbols against an allowlist,
- validate date ranges before sending API calls,
- add schema repair or argument normalization before tool execution.

### Performance
For better runtime quality, I would add:
- caching for repeated ticker requests,
- retries with backoff for transient API failures,
- stronger structured parsing than plain regex + raw string arguments,
- aggregated dashboards for tool success rate, latency, and input-parse errors.

These changes would make the stock tools more production-ready and reduce the gap between demo behavior and real-world financial-agent reliability.

---

> [!NOTE]
> Submit this report by renaming it to `REPORT_[YOUR_NAME].md` and placing it in this folder.
