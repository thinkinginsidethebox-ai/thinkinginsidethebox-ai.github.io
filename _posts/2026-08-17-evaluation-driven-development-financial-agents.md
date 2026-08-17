---
layout: post
title: "Stop Trusting the Black Box: Evaluation-Driven Development for Enterprise Financial Agents"
date: 2026-08-17 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [evaluation-driven-development, agentic-safety]
projects: [mcp]
image: "/assets/images/og/evaluation-driven-development-financial-agents.png"
description: "From vibe checks to Evaluation-Driven Development: MLflow tracing, MCP tool-call evaluation, and MemAlign CLHF for enterprise financial agents."
external_repo: "https://github.com/caldeirav/mlflow-edd-financial-agent"
---

Deploying autonomous AI agents into enterprise workflows — financial analysis, market research, automated portfolio briefing — demands a fundamental transition from subjective "vibe checks" to **Evaluation-Driven Development (EDD)**. As demonstrated in the [caldeirav/mlflow-edd-financial-agent](https://github.com/caldeirav/mlflow-edd-financial-agent) reference architecture, non-deterministic LLM outputs, multi-step planning, and distributed tool invocations present severe operational risks around data precision, tool routing errors, and ungrounded synthesis. EDD addresses these failure modes by embedding quantitative testing, distributed tracing, and domain-calibrated evaluation directly into the software development lifecycle.

On this blog, we call that discipline **thinking inside the box**. EDD is how the box becomes inspectable: every tool call is traced, every answer is scored, and every automated judge is calibrated against expert judgment.

---

## The Case for Evaluation-Driven Development in Agentic Systems

Traditional software testing relies on deterministic assertions, while classic machine learning relies on static evaluation datasets. Neither approach is sufficient for agentic architectures, where an LLM dynamically plans execution steps, invokes external tools via standards like the [Model Context Protocol (MCP)](https://modelcontextprotocol.io), and iteratively refines its reasoning based on tool outputs.

EDD structures agent quality assurance around two continuous evaluation loops:

* **Inner Loop (CI/CD Pre-Deployment):** Developers evaluate prompt templates, tool selection, and answer groundedness against a versioned golden dataset before merging code. By running automated evaluation suites during pull requests, engineering teams catch regressions in tool routing or reasoning logic before deployment.
* **Outer Loop (Production Observability):** Live production monitoring continuously traces real-world user interactions with financial agents. Outer-loop evaluation measures hallucination rates on live market queries, tracks tool-call failure rates, and detects semantic drift over time.

---

## Pillar 1: Persistent Tracing and Protocol-Level Observability

Agentic architectures are distributed state machines. When a financial agent processes a request like *"Provide a financial analysis of AAPL — cover recent price context, relevant news, key financial-statement signals, and risks,"* the system generates a chain of intermediate thoughts, tool calls, and payload responses. Without granular, persistent tracing, debugging why an agent selected an incorrect tool or produced an erroneous financial figure becomes impossible.

As implemented in the [mlflow-edd-financial-agent](https://github.com/caldeirav/mlflow-edd-financial-agent) repository, persistent state management relies on tracking every decision point, LLM call, and tool execution inside a persistent backend like SQLite via MLflow Tracing. Storing full execution graphs enables developers to replay agent state, isolate intermediate failures, and audit the exact lineage of every generated financial metric.

### Overcoming the MCP Observability Challenge with MLflow Tracing

When agents interact with external tools over MCP, tool execution is decoupled across independent server processes. Standard agent logs fail at the MCP boundary, creating a critical blind spot where engineers cannot distinguish between an LLM tool-selection error, a JSON payload serialization failure, or a downstream data timeout.

The reference architecture resolves this with OpenTelemetry-style spans captured by `mlflow.langchain.autolog()`. A FastMCP server (`mcp_server.py`) exposes live Yahoo Finance tools over stdio. LangGraph binds those tools through `langchain-mcp-adapters`. Autolog then records typed `AGENT` / `TOOL` / `LLM` spans — including arguments and results — into `sqlite:///mlflow.db` for inspection in the MLflow UI.

The MCP server itself is deliberately small: three market-data tools, no mock tool list.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("yahoo-finance")

@mcp.tool()
def get_stock_price(ticker: str) -> str:
    """Return the latest stock price and short context for a ticker."""
    t = yf.Ticker(ticker.strip().upper())
    info = t.fast_info.__dict__ if hasattr(t, "fast_info") else {}
    price = info.get("last_price") or info.get("lastPrice")
    return json.dumps({
        "ticker": ticker.upper(),
        "price": float(price) if price is not None else None,
        "currency": info.get("currency"),
    }, default=str)

@mcp.tool()
def get_financial_news(ticker: str, limit: int = 5) -> str:
    """Return recent financial news headlines for a ticker."""
    ...

@mcp.tool()
def get_financial_statements(
    ticker: str,
    statement_type: Literal["income", "balance", "cashflow"] = "income",
) -> str:
    """Return key financial statement rows for a ticker (truncated)."""
    ...
```

On the agent side, tracing is enabled before the ReAct loop starts. The MCP client launches the FastMCP process, loads the tool schemas, and every subsequent tool invocation lands as a durable `TOOL` span:

```python
mlflow.langchain.autolog()
llm = build_llm()
client = MultiServerMCPClient(_mcp_server_command())
tools = await client.get_tools()
agent = create_react_agent(llm, tools, prompt=SYSTEM_PROMPT)
result_state = await agent.ainvoke(
    {"messages": [{"role": "user", "content": question}]},
)
```

That is the audit trail. Open the experiment **edd-financial-assistant** in the MLflow UI, inspect a trace, and you can see which MCP tool was called, with which ticker, and what observation the model used when it wrote the Markdown report.

---

## Pillar 2: Automated Evaluation Pipelines and Metric Standardization

Scalable agent engineering requires replacing manual inspection with programmatic, reproducible evaluation pipelines. Automated EDD pipelines utilize LLM-as-a-Judge evaluators combined with deterministic heuristic metrics to assess agent outputs across multiple performance dimensions.

In the mlflow-edd-financial-agent framework, evaluation pipelines leverage `mlflow.genai.evaluate()` to execute standardized evaluation suites against agent runs. Rather than scoring overall output text in isolation, the evaluation pipeline breaks down agent execution into distinct quantitative metrics:

| Metric | Type | Target Evaluation Focus |
| :--- | :--- | :--- |
| **RequiredMarkdownSections** | Code scorer | Required `##` headings present (Price context, News, Financial statements, Risks/limitations). |
| **RequiredToolsUsed** | Code scorer | Required market tools actually called (`get_stock_price`, `get_financial_news`, `get_financial_statements`). |
| **ToolCallCorrectness** | Gemini judge | Right tools for the request — selection accuracy against expectations. |
| **ToolCallEfficiency** | Gemini judge | Lean tool use — needed calls without redundant thrash. |
| **Groundedness** | Gemini judge | Numeric and factual claims supported by live tool observations, not invented. |

Together, the tool scorers target **calling accuracy**; Groundedness targets whether tool **execution results** were used faithfully in the report. Judges score inputs, outputs, and expectations (not raw traces). Tool evidence is compacted into outputs so qualitative judges see the same facts as the UI.

A baseline pass over the versioned golden dataset looks like this:

```python
scorers = [
    RequiredMarkdownSections,
    RequiredToolsUsed,
    *build_uncalibrated_judges(),
]
with mlflow.start_run(run_name=f"baseline-eval-{dataset_version}"):
    mlflow.set_tags(tags)
    results = mlflow.genai.evaluate(
        data=data,
        predict_fn=_predict_fn,
        scorers=scorers,
    )
```

The Groundedness judge is explicit about live tool payloads — not frozen snapshots — as the source of truth:

```python
groundedness = make_judge(
    name="Groundedness",
    instructions=(
        "You are evaluating a financial-analysis agent.\n"
        "Inputs: {{ inputs }}\n"
        "Outputs include the Markdown report and tool_observations "
        "(live tool results from this run — not frozen snapshots): {{ outputs }}\n"
        "Return 'grounded' if numeric/factual claims in the report are supported "
        "by tool_observations; 'ungrounded' if the answer invents or contradicts "
        "tool data."
    ),
    model="gemini:/gemini-2.5-pro",
    feedback_value_type=Literal["grounded", "ungrounded"],
)
```

Automating these evaluations within CI/CD pipelines ensures that changes to system prompts, underlying foundation models, or tool schemas do not silently degrade agent performance in production.

---

## Pillar 3: Expert Judge Alignment and Continuous Learning via Human Feedback (CLHF)

Automated LLM-as-a-Judge evaluators provide rapid feedback, but uncalibrated LLM judges often suffer from self-preference bias, verbosity bias, and an inability to detect subtle domain-specific errors. In financial analysis, an automated judge might rate an answer as correct because it sounds coherent, while completely missing an invented price, a contradicted income-statement row, or a news claim that never appeared in the tool payload.

The third pillar of EDD focuses on calibrating automated evaluators against human Subject Matter Experts (SMEs), such as senior financial analysts and auditors. By establishing a blind double-scoring protocol — where both the LLM judge and human experts evaluate identical agent runs — teams can measure agreement and systematically refine scoring rubrics until the automated judge closely mirrors expert human judgment.

### Solving MCP Tool Selection Accuracy with CLHF

In enterprise agent deployments, incorrect tool selection is one of the most common failure modes. Agents operating over MCP tool registries frequently pick sub-optimal tools, hallucinate non-existent parameter names, or enter redundant tool-execution loops when presented with ambiguous user prompts.

Continuous Learning via Human Feedback (CLHF) resolves this by connecting production observability directly to expert calibration through a closed-loop engineering workflow. In the reference architecture, that loop is implemented on MLflow traces with MemAlign:

1. **Trace Capture & Span Filtering:** Agent runs captured via autolog and stored in the SQLite/MLflow backend are reviewed in the UI. Engineers screen traces for anomalies — low evaluator confidence, failed tool calls, ungrounded figures, or empty reports.
2. **SME Annotation:** Flagged traces receive a **HUMAN** assessment using the **same name** as the judge being calibrated (`ToolCallCorrectness`, `ToolCallEfficiency`, or `Groundedness`), plus a short rationale. MemAlign uses both the label and the rationale.
3. **Adaptive Feedback Loop:** `MemAlignOptimizer` embeds those expert examples into an episodic memory attached to the judge. Alignment does not fine-tune Gemini weights. At eval time, the aligned judge retrieves nearby human-labeled cases as few-shot context. A second eval run is tagged `eval_phase=aligned`; the uncalibrated baseline is retained for comparison.

This continuous feedback loop turns tool-selection and grounding failures into high-value training and evaluation data, systematically driving calling accuracy and faithfulness toward production thresholds.

```python
optimizer = MemAlignOptimizer(
    reflection_lm="gemini:/gemini-2.5-flash",
    embedding_model="gemini:/gemini-embedding-001",
)
feedback_traces = _traces_with_human_feedback("Groundedness")
aligned_judge = base_judge.align(traces=feedback_traces, optimizer=optimizer)
aligned_judge.register(experiment_id=experiment_id)
```

---

## Operationalizing EDD: From Demo to Production

Transitioning agentic AI from an experimental prototype to an enterprise-grade asset requires abandoning the belief that foundation models can be trusted out of the box. The [caldeirav/mlflow-edd-financial-agent](https://github.com/caldeirav/mlflow-edd-financial-agent) project demonstrates that combining persistent MLflow tracing, an OpenTelemetry-style view of MCP tool calls, automated evaluation suites, and expert-aligned CLHF loops creates a robust software engineering harness for autonomous agents.

The operational loop is always the same: **instrumented run → evaluate tool use and groundedness → CLHF in the UI → align → compare**. Which judge you annotate depends on whether you are tuning **tool selection** or **answer quality from tool results** (or both). Further rounds add more HUMAN feedback; baselines remain for regression comparison.

By building on these three pillars of EDD, engineering organizations gain complete visibility into agent decision-making, ensure strict tool-calling accuracy across distributed protocols, and deliver trustworthy financial AI systems ready for production deployment.

---

*Demo repository: [github.com/caldeirav/mlflow-edd-financial-agent](https://github.com/caldeirav/mlflow-edd-financial-agent).*
