---
layout: post
title: "One request for every model your agent calls"
date: 2026-10-29 09:00:00 +0800
categories: [AAIF, Engineering]
projects: [open-responses]
image: "/assets/images/og/open-responses-for-agent-developers.png"
description: "Open Responses is now an AAIF project. Write the agent against items and /v1/responses, then point that same client at a gateway or a vLLM pool behind llm-d."
---

On 22 October 2026 the Agentic AI Foundation added four projects and reached ten in its first year. One of them is [Open Responses](https://www.openresponses.org/), contributed by OpenAI. The [announcement](https://aaif.io/news/agentic-ai-foundation-announces-four-new-projects) puts it next to MCP, A2A, agentgateway, AGENTS.md, and goose, under the same Linux Foundation governance. Code in the project is Apache-2.0. Documentation is CC BY 4.0.

For an agent developer the date matters less than the contract. Take the example of a simple coding agent that needs to find a failing test which requires four API calls: it sends the user turn and its tools, it receives a request to read a file, it runs that tool, and it sends the result back. Anthropic's Messages API puts that prompt in `messages`. Google's `generateContent` API uses `contents` and `parts`. Each new provider, and each new feature on an old provider, becomes another branch in the client.

[Open Responses](https://aaif.io/blog/what-open-responses-changes-for-ai-developers) aims at standardising this kind of interaction on OpenAI's Responses API. The endpoint and the model name become configuration. A provider, a gateway, or a local server implements the specification and, where it must, translates behind that endpoint. The model weights stay as they are. The agent still executes tools and still decides what those tools are allowed to touch.

In this article, I will cover the part you actually need to maintain as an Agent Developer: the items on the wire, a client that can target more than one endpoint, and what changes when that endpoint is a vLLM model serving process behind [llm-d](https://llm-d.ai/).

## What the specification asks your client to handle

The [specification](https://www.openresponses.org/specification) is a wire protocol for `POST /v1/responses`. There are four important design rules that cover the typical interactions of an agentic client using the Open Responses-compatible endpoint.

**Items are the unit of context.** A turn is a sequence of typed objects, discriminated on `type`, and the same objects travel in both directions. A user message goes in. A `function_call` comes out. A `function_call_output` goes back. A `reasoning` item may come out beside the answer. Every item carries an `id`, a `type`, and a `status`. That gives every turn an explicit shape. You stop flattening the transcript into an untyped list of role strings, and you stop guessing which blob was a tool result.

**Items and responses are small state machines.** An item moves through `in_progress` while the model is still emitting it, then ends `completed` or `incomplete`. `incomplete` means the token budget ran out; it is terminal, it is the last item, and the response that contains it is `incomplete` too. That lifecycle describes the object inside this response. Persistence is a separate flag, `store`.

**Streams are semantic events.** A streaming response is `text/event-stream`. The `event` name matches the JSON `type`, events carry a `sequence_number`, and the stream ends with `response.completed`, `response.failed`, or `response.incomplete`. Branch on lifecycle events. `response.output_item.added` fires when a new item appears, so you already know a tool name before its arguments finish. `response.output_text.delta` grows the text. `response.function_call_arguments.delta` grows the arguments. Matching `done` events close each part. Assemble the tool call from those events.

**Extensions stay namespaced.** A provider may add an item or a stream event. The type must carry that provider's slug, as in `acme:trace_event` or `openai:web_search_call`. Clients that understand the slug can show cache timing or a hosted-tool phase. Everyone else ignores the event and keeps reading the standard stream. If an extension spreads, the project has a path to pull it into the spec. Until then it is not portable, and a transcript that contains it should be treated as belonging to that implementation.

Three request fields change how you write the loop.

`previous_response_id` continues a stored response. The follow-up body carries the new user text or the `function_call_output`, plus the id, and omits the history the server already has. That only works when the server stored the prior response. With `store: true`, a later HTTP call can rehydrate it. With `store: false`, there is no persisted fallback. On a WebSocket the server may keep that last response in connection-local memory, which is how a zero-retention session continues on the same socket. If the id is gone, the error code is `previous_response_not_found` and the client sends the full input again.

`truncation` decides what happens when the rehydrated input no longer fits. `auto` allows the server to drop older context. `disabled` requires a hard error instead of a silent cut. Use `disabled` when a missing tool result would be worse than a failed turn. Use `auto` only where you have defined what "older" is allowed to mean.

`allowed_tools` is the cache-friendly permission switch. The full `tools` list stays in the prompt, so a stable system prompt and a stable schema prefix can stay cached, while the model may call only the subset you name on this turn.

Reasoning is managed with three optional fields on a reasoning item. `content` is the raw trace, when the provider is willing to emit it. `encrypted_content` is provider-protected state the client can round-trip and usually cannot read. `summary` is the sanitized text you can show. Open-weight model servers often stream raw reasoning as `response.reasoning.delta`. A client written for summary-only APIs should accept a missing `content` and keep going. A client that needs the trace for an audit log should treat a summary-only endpoint as a different retention decision, and record that decision next to the response id.

Provider-hosted tools and client-hosted tools share the loop and differ in who runs it. A function tool yields a `function_call` and waits. Your process runs it and posts `function_call_output`. A hosted tool, such as a provider search index, runs inside the endpoint. `max_tool_calls` caps the loop. `tool_choice` constrains which tools are eligible. The intermediate items come back on the same response, so a trace can show the call, the result, and the final text without a private side log.

The same events ride a WebSocket at `/v1/responses`. The client opens a turn with `response.create` and leaves HTTP-only fields such as `stream` off that frame. One socket carries one in-flight response. Connections are limited to 60 minutes (`websocket_connection_limit_reached`). After `POST /v1/responses/compact`, the next turn starts from the compacted input with `previous_response_id` cleared. Those rules landed in the [2026-04-24 specification](https://www.openresponses.org/changelog).

| | Chat Completions | Provider-native messages | Open Responses |
| --- | --- | --- | --- |
| Context | Role and content objects | Provider blocks (`messages`, or `contents` and `parts`) | Typed items (`message`, `function_call`, `function_call_output`, `reasoning`) |
| Next turn | Client resends the transcript | Client resends the transcript, or uses that provider's cache headers | `previous_response_id` when `store` is true; otherwise resend items |
| Stream | Token deltas on `delta.content` | Provider block events | Item, text, argument, and lifecycle events with a terminal response event |
| Tools | Client parses a tool call and loops | Client posts tool results in that provider's shape | `function_call` out, `function_call_output` back; hosted tools stay inside the endpoint |
| Reasoning | Often absent, or mixed into the answer | Provider-specific | Optional `content`, `encrypted_content`, and `summary` |
| Change control | One vendor | One vendor | AAIF project, with a [conformance suite](https://www.openresponses.org/compliance) |

Run that suite against every endpoint you depend on, including your own. Unknown item types and unknown slug events are part of the contract: ignore them, and keep the standard items.

## The client you maintain

If you already speak the OpenAI Responses API, the migration is simply a change in the base URL. The official Python client targets any server that implements the specification.

```python
import json
import os
from openai import OpenAI

client = OpenAI(
    base_url=os.environ.get("OPEN_RESPONSES_BASE_URL", "http://127.0.0.1:9000/v1"),
    api_key=os.environ.get("OPEN_RESPONSES_API_KEY", "none"),
)
model = os.environ.get("MODEL", "Qwen/Qwen3-32B")

tools = [{
    "type": "function",
    "name": "read_file",
    "description": "Read a UTF-8 file the agent is allowed to open.",
    "parameters": {
        "type": "object",
        "properties": {"path": {"type": "string"}},
        "required": ["path"],
        "additionalProperties": False,
    },
}]

def read_file(path: str) -> str:
    # Permissions stay in the harness. The specification does not move them.
    with open(path, encoding="utf-8") as handle:
        return handle.read()

first = client.responses.create(
    model=model,
    store=True,
    instructions="You are a coding agent. Read a file before you explain a failure.",
    tools=tools,
    input="The parse_config unit test is failing. Name the file I should open.",
)

call = next(item for item in first.output if item.type == "function_call")
follow_up = client.responses.create(
    model=model,
    store=True,
    previous_response_id=first.id,
    tools=tools,
    input=[{
        "type": "function_call_output",
        "call_id": call.call_id,
        "output": read_file(json.loads(call.arguments)["path"]),
    }],
)
print(follow_up.output_text)
```

If `output` has no `function_call`, the model answered in text and there is nothing to execute. That second call is the whole portability claim. Swap `OPEN_RESPONSES_BASE_URL` and `MODEL`. The tool loop stays.

Several endpoints already accept this body. [Databricks Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/query-open-responses-models) exposes it for Claude, Gemini, and Databricks-hosted open models, and translates to each provider. [Vercel AI Gateway](https://vercel.com/docs/ai-gateway/sdks-and-apis/openresponses) exposes an Open Responses API in front of its model catalog. [Amazon Bedrock](https://aws.amazon.com/blogs/opensource/open-responses-joins-aaif-with-aws-support/) already serves the Responses API across many models. Hugging Face published an early router path in [January 2026](https://huggingface.co/blog/open-responses). In each case the gateway owns the translation. Your process owns `read_file`.

Two types of failures will typically show up as soon as you point the same code snippet at a raw engine.

The first is state. `previous_response_id` is a server feature. A stateless engine will not rehydrate an id it never stored. On those servers, append the model's `function_call` item and your `function_call_output` to `input` and send the list again. vLLM's own [Responses tool example](https://github.com/vllm-project/vllm/blob/main/examples/tool_calling/openai_responses_client_with_tools.py) does exactly that against `http://localhost:8000/v1`. Keep one helper with two strategies: continue by id when the endpoint stored the turn, resend items when it did not. `store: false` is the explicit form of the second strategy, including zero-retention deployments.

The second is tool text that never becomes a `function_call`. The model emitted a call in its own markup, and the server treated it as assistant text. Fix that on the server process, before you debug the agent. vLLM only structures the call when it is started with `--enable-auto-tool-choice` and a `--tool-call-parser` that matches the model's chat template. A reasoning model also needs `--reasoning-parser`, so chain-of-thought tokens become a reasoning item and the parser looks for tools in the answer. The llm-d guide's validated NVIDIA path is `Qwen/Qwen3-32B` with `hermes` and `qwen3`:

```bash
vllm serve Qwen/Qwen3-32B \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --reasoning-parser qwen3
```

A wrong parser is worse than none. The server starts, accepts `POST /v1/responses`, and then fails the tool loop. Other models in that guide use different pairs (`openai` for `gpt-oss-120b`, `qwen3_xml` for Qwen3.5 templates, `deepseek_v4` for DeepSeek V4). So it is important you make sure to match the checkpoint you loaded.

vLLM's Responses routes are `/v1/responses`, `/v1/responses/{response_id}`, cancel, and [`/v1/responses/render`](https://docs.vllm.ai/en/latest/serving/online_serving/openai_compatible_server/). `render` expands the chat template and returns prompt token ids without a GPU forward pass, which is the right preflight when you need a token count, a context-window check, or a cache key before you spend a decode. Extra sampling fields such as `top_k`, `min_p`, and `repetition_penalty` pass through to the engine. They are server extensions. Leave them out of the portable client, or send them only on the base URL you know is vLLM.

The engine should stay stateless. Continuous batching and prefix cache want a complete request, a forward pass, and no session table. Multi-turn state belongs in front of it. That front door is [vLLM Agentic API](https://github.com/vllm-project/agentic-api), a Rust service. It listens on port 9000, writes responses to PostgreSQL 17, and on `previous_response_id` loads the prior items, prepends them, and forwards one stateless Responses request to the engine. Chat Completions and embeddings are proxied through with no session and no tool loop. Responses requests with `store: false` and no state id are also forwarded as a full history, without a write.

You can let that gateway run the tool loop. It accepts MCP server declarations on the request (up to 64 servers and 128 discovered tools in current builds), calls the tool, and submits `function_call_output` itself. That is convenient for a coding harness pointed at one URL. For an agent you already govern, keep the loop in the harness, as in the snippet above. The gateway then only stores and rehydrates. Hosted tools and gateway-executed tools are an explicit choice (`tool_choice`, `max_tool_calls`, and which process holds the credentials), recorded where you review permissions.

One vLLM limit belongs in the client. Agentic API prepares plaintext reasoning for vLLM and rejects encrypted-only reasoning the engine cannot replay. If your trace depends on `encrypted_content` round-trips, a vLLM backend will not satisfy it. Ask for `content` or `summary`, or terminate that session on an endpoint that can replay the encrypted state.

## The same client, pointed at a cluster

A single vLLM process is sufficient for a development server. But with a fleet of agent sessions, you will soon face a routing problem: long tool outputs and long prefixes want prefill capacity, token generation wants memory bandwidth, and the next turn is cheaper on the replica that already holds the prefix. [llm-d](https://llm-d.ai/) 1.0, released 20 October 2026, is the cloud-native stack for that. It is currently a CNCF sandbox project, co-founded by Google, Red Hat, IBM, and others, and it sits on the Kubernetes Gateway API Inference Extension.

With llm-d, from the perspective of the agent, nothing in the snippet has to change. You still `POST /v1/responses` to one base URL. Continuation is still `previous_response_id` or a full item list. Tool results are still `function_call_output`. Pod choice, KV placement, and prefill versus decode are properties of the request the gateway forwards, which is why a stable `instructions` block and a stable `tools` list are serving configuration as much as prompt design. `allowed_tools` is how you narrow a turn without rewriting the prefix the router is trying to stick.

Note that there is a [well-lit path](https://llm-d.ai/docs/well-lit-paths/foundations/serve-responses-api) for this which reuses the Optimized Baseline router and model servers, then adds two deployments:

| Piece | Role |
| --- | --- |
| `agentic-api` on port 9000 | The base URL your agent uses. Stores responses in PostgreSQL and rehydrates `previous_response_id`. |
| PostgreSQL 17 | The state store. The guide generates the password into a Secret. |
| llm-d router on port 80 | Places each stateless inference round. `agentic-api` dials it by DNS. |
| vLLM replicas | Must expose `/v1/responses`. The NVIDIA reference is two `Qwen/Qwen3-32B` replicas, tensor parallel 2, with the tool-calling flags above. |

SGLang and TensorRT-LLM are outside this path. Agentic API delegates each round to vLLM's Responses implementation. The guide's checkout is the `release-1.0` branch, guide name `agentic-api`, namespace `llm-d-agentic-api`. After the model servers are Available, the developer loop is a port-forward and a check that state and tools actually work:

```bash
kubectl port-forward --address 127.0.0.1 -n llm-d-agentic-api svc/agentic-api 9000:9000
export OPEN_RESPONSES_BASE_URL=http://127.0.0.1:9000/v1
export MODEL=Qwen/Qwen3-32B

curl -sS -X POST http://127.0.0.1:9000/v1/responses \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen/Qwen3-32B","input":"How are you today?"}'

python3 guides/agentic-api/verify.py --base-url http://127.0.0.1:9000
```

`verify.py` is the contract worth copying into your own smoke test. It checks health and model discovery. It stores a secret with `store: true` and continues with only `previous_response_id`, so a pass means PostgreSQL rehydration worked. It runs a tool loop through a local webhook, and that test fails with a clear message when the model server is missing the parser flags. It opens a WebSocket to `/v1/responses`, sends `response.create`, and waits for a terminal event. All four run from the client side. The cluster does not dial back to your laptop.

So the same gateway can sit in front of a prefill/decode guide or a wide-expert-parallel guide you already deployed, as long as the model server is vLLM and the parser flags are on that server's `vllm serve` line. There is no environment-variable equivalent for those flags.

What the router does with the forwarded request is the part worth understanding so your timeouts and your prefixes match the cluster.

Envoy hands the request to an Endpoint Picker over Envoy external processing. The picker scores replicas from live engine telemetry: queue depth, KV-cache block use, and which replica already holds the longest matching prefix. It balances that affinity against predicted time-to-first-token and time-per-output-token, then routes to one pod. A round-robin Service will recompute a shared system prompt on every replica. Prefix-aware placement is how `previous_response_id` stays cheap after the gateway has expanded it back into a long prompt.

Prefill and decode interfere when they share a GPU. Long tool outputs and code blocks fill the prefill, and token generation for other sessions waits. llm-d's DisaggregatedSet, built on LeaderWorkerSet, runs those as separate pools, moves KV cache between them (NIXL and Mooncake are the usual paths), rolls them forward on matching versions, and scales them on different signals. The agent still addresses one base URL. The effect shows up in latency: a huge `function_call_output` is a prefill problem, and a chatty decode is a memory-bandwidth problem. `truncation: disabled` will fail the turn when that output blows the window. `truncation: auto` will fit it by dropping something. Pick the one your agent can explain.

KV cache is tiered because multi-turn agents exhaust accelerator memory. Active blocks stay in GPU or TPU memory. vLLM's OffloadingConnector, or SGLang's HiCache on stacks that use it, can move colder blocks to host DRAM, then to NVMe or a shared filesystem such as Google Cloud Managed Lustre. Peers can also copy a prefix from another node's host memory. The agent-facing consequence is session stickiness. A follow-up that lands on a replica with the prefix in host RAM is a different latency from a follow-up that recomputes 50,000 tokens. The picker is what makes that choice. Your client sends the same body either way.

Interactive turns and background agent work share that fleet. llm-d 1.0's flow control keeps priority lanes, and `llm-d-async-processor` pulls lower-priority jobs from Redis or Pub/Sub only when the pool has slack. If your product runs a user-facing coding turn and a batch of eval rollouts against the same endpoint, mark them on the router’s priority lanes. Open Responses leaves that split to the serving tier.

Published numbers measure those placement choices, on stated hardware, against a stated baseline. Treat them as evidence from those platforms. The client in front of them stays the same.

| What was measured | Baseline | Result | Source |
| --- | --- | --- | --- |
| Agentic coding on TPU 7x, prefix-aware routing plus host-memory offload | Plain Kubernetes Service | Median time-to-first-token down more than 85%, throughput 2.4× | [Google Cloud, 20 Oct 2026](https://cloud.google.com/blog/topics/ai-infrastructure/llm-d-1-0-an-open-source-stack-for-distributed-inference) |
| Gemini production traffic through GKE Inference Gateway | Prior gateway behavior | Prefix-cache hit rate from 35% to 70%; coding-agent TTFT down more than 35%; bursty-chat P95 TTFT down more than 50% | Same |
| 70B model, 50,000-token prompts, KV offload to host RAM and Managed Lustre | Host RAM only | About 40% better TTFT, 30% lower end-to-end latency, nearly 60% fewer GPU-hours | Same |
| 17,000-node multi-region GKE deployment, mixture-of-experts | Routing across three clusters | 2.9× throughput, 99.9% success, under 1% routing overhead | Same |
| Disaggregated serving on SageMaker HyperPod, prefix routing plus NIXL | Standard vLLM | Up to 70% more output tokens per second | [llm-d 1.0](https://llm-d.ai/blog/llm-d-v1.0-state-of-the-art-inference-ready-for-day-2) |

JetBrains runs this router in production behind Junie, using predicted-latency scheduling across long coding sessions and short requests. The agent binary stays a Responses client.

llm-d's [observability docs](https://llm-d.ai/docs/operations/observability) cover Prometheus and OpenTelemetry on the router and the model servers. Join them to the response id your client already logs. Cache hit rate, queue depth, and time-to-first-token explain a slow turn. The semantic event log explains what the model asked the tools to do.

## What to change in the agent you already have

1. Make `POST /v1/responses` the model call. Keep one request builder. Put the endpoint and the model name in configuration, and run the [conformance suite](https://www.openresponses.org/compliance) on every base URL you ship against.
2. Represent the loop as items. Append `function_call` and `function_call_output`. Keep permissions in the harness that produces the output.
3. Consume semantic events. Read `sequence_number`. Ignore slug-prefixed events you do not implement. Require a terminal response event before you treat the turn as finished.
4. Choose storage per session. `store: true` and `previous_response_id` when the endpoint is agentic-api or a provider that persists. Resend items when the engine is raw vLLM or the session is `store: false`.
5. Set `truncation: disabled` until you have a written policy for what may be dropped. Set `allowed_tools` when you want a tighter tool menu without invalidating the shared tool prefix.
6. Treat reasoning as optional fields. Show `summary` by default. Persist `content` only where policy allows the raw trace. For `encrypted_content` round-trips, terminate on an endpoint that can replay them. vLLM and Agentic API accept plaintext reasoning.
7. Start vLLM with the parser pair for that checkpoint, then prove it with a tool-loop smoke test that executes a `function_call` and posts `function_call_output`.
8. When one process is no longer enough, put llm-d and agentic-api behind the same base URL. Follow the [experimental guide](https://llm-d.ai/docs/well-lit-paths/foundations/serve-responses-api) on `release-1.0`, and keep `verify.py`'s four checks. Change the deployment. Leave the agent loop alone.
9. When a provider extension helps, implement that slug on purpose. File the gap, the lost field, or the adapter you were able to delete on the [Open Responses issue tracker](https://github.com/openresponses/openresponses/issues). That is the feedback AAIF asked implementers to send.

The specification, the reference OpenAPI document, and the compliance tests live in [openresponses/openresponses](https://github.com/openresponses/openresponses). The serving path in this article is [vllm](https://github.com/vllm-project/vllm), [vllm-project/agentic-api](https://github.com/vllm-project/agentic-api), and [llm-d/llm-d](https://github.com/llm-d/llm-d).
