---
name: opengradient-sdk
description: >
  Use when the user wants to write code using the OpenGradient SDK, including
  LLM inference, chat completions, streaming, tool calling, on-chain model
  inference, LangChain agents, model hub operations, or digital twins.
  Also use when the user asks how to use a specific OpenGradient feature.
argument-hint: "[task description]"
allowed-tools: Read, Grep, Glob
---

You are an expert on the **OpenGradient Python SDK** (`opengradient`). Help the user write correct, idiomatic code using the SDK.

When the user describes what they want to build, generate working code that follows the patterns below. Always prefer the simplest approach that satisfies the requirements.

You can find more information about the OpenGradient infrastructure on https://docs.opengradient.ai.

## Installation

This guide was written for OpenGradient SDK version 0.9.4, make sure to install this version.

```bash
# Requires Python >=3.11
pip install opengradient==0.9.4
```

## SDK Overview

OpenGradient is a decentralized AI inference platform. The SDK provides:

- **Verified LLM inference** via TEE (Trusted Execution Environment)
- **x402 payment settlement** on Base Sepolia (on-chain receipts)
- **Multi-provider models** (OpenAI, Anthropic, Google, xAI) through a unified API
- **LangChain integration** for building agents
- **Digital twins** chat
- **On-chain ONNX model inference** (alpha features)

## Initialization

Each service is instantiated separately — there is no single `init()` function:

```python
import opengradient as og

# LLM inference (requires Base Sepolia private key with OPG tokens)
llm = og.LLM(private_key="0x...")

# On-chain ONNX inference (requires OpenGradient testnet private key)
alpha = og.Alpha(private_key="0x...")

# Model Hub (requires email/password auth)
hub = og.ModelHub(email="...", password="...")

# Digital twins (requires twins API key)
twins = og.Twins(api_key="...")
```

Before the first LLM call, approve OPG token spending (idempotent — skips if allowance is sufficient):
```python
llm.ensure_opg_approval(min_allowance=5)
```

The `ensure_opg_approval` method accepts:
- `min_allowance` (float): Minimum OPG allowance required.
- `approve_amount` (float, optional): Amount to approve. Defaults to `2 * min_allowance`.

Returns a `Permit2ApprovalResult` with `allowance_before`, `allowance_after`, and `tx_hash` (None if no approval was needed).

Users must acquire $OPG tokens on Base Sepolia in their wallet in order to pay for inferences via x402.
If the user owns no $OPG, you they can request via our [faucet](https://faucet.opengradient.ai/).

## Available Models (`og.TEE_LLM`)

| Provider   | Models |
|------------|--------|
| OpenAI     | `GPT_4_1_2025_04_14`, `O4_MINI`, `GPT_5`, `GPT_5_MINI`, `GPT_5_2` |
| Anthropic  | `CLAUDE_SONNET_4_5`, `CLAUDE_SONNET_4_6`, `CLAUDE_HAIKU_4_5`, `CLAUDE_OPUS_4_5`, `CLAUDE_OPUS_4_6` |
| Google     | `GEMINI_2_5_FLASH`, `GEMINI_2_5_PRO`, `GEMINI_2_5_FLASH_LITE`, `GEMINI_3_PRO`, `GEMINI_3_FLASH` |
| xAI        | `GROK_4`, `GROK_4_FAST`, `GROK_4_1_FAST`, `GROK_4_1_FAST_NON_REASONING` |

## Settlement Modes (`og.x402SettlementMode`)

- `PRIVATE` — Payment only, no data on-chain (maximum privacy)
- `BATCH_HASHED` — Aggregated into Merkle tree (most cost-efficient, **default**)
- `INDIVIDUAL_FULL` — Full input/output recorded on-chain (maximum transparency)

## Core Patterns

**IMPORTANT**: `llm.chat()` and `llm.completion()` are **async** methods. Use `await` inside an async function or `asyncio.run()` for top-level calls.

### Basic Chat

```python
import asyncio
import opengradient as og

llm = og.LLM(private_key="0x...")
llm.ensure_opg_approval(min_allowance=5)

result = asyncio.run(llm.chat(
    model=og.TEE_LLM.GEMINI_2_5_FLASH,
    messages=[{"role": "user", "content": "Hello!"}],
    max_tokens=300,
    temperature=0.0,
))
print(result.chat_output["content"])
```

### Streaming

```python
import asyncio
import opengradient as og

async def stream_example():
    llm = og.LLM(private_key="0x...")
    llm.ensure_opg_approval(min_allowance=5)

    stream = await llm.chat(
        model=og.TEE_LLM.GPT_5,
        messages=[{"role": "user", "content": "Explain quantum computing"}],
        max_tokens=500,
        stream=True,
    )
    async for chunk in stream:
        if chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)

asyncio.run(stream_example())
```

### Tool Calling

```python
import asyncio
import opengradient as og

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get current weather",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    },
}]

async def tool_example():
    llm = og.LLM(private_key="0x...")
    llm.ensure_opg_approval(min_allowance=5)

    result = await llm.chat(
        model=og.TEE_LLM.GPT_5,
        messages=[{"role": "user", "content": "Weather in NYC?"}],
        tools=tools,
        max_tokens=200,
    )

    if result.finish_reason == "tool_calls":
        for tc in result.chat_output["tool_calls"]:
            print(f"Call: {tc['function']['name']}({tc['function']['arguments']})")

asyncio.run(tool_example())
```

### Multi-Turn Tool Agent Loop

```python
import asyncio
import opengradient as og

async def agent_loop(user_query, tools, max_iterations=5):
    llm = og.LLM(private_key="0x...")
    llm.ensure_opg_approval(min_allowance=5)

    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": user_query},
    ]

    for _ in range(max_iterations):
        result = await llm.chat(
            model=og.TEE_LLM.GPT_5,
            messages=messages,
            tools=tools,
            tool_choice="auto",
        )
        if result.finish_reason == "tool_calls":
            messages.append(result.chat_output)
            for tc in result.chat_output["tool_calls"]:
                tool_result = execute_tool(tc["function"]["name"], tc["function"]["arguments"])
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc["id"],
                    "content": tool_result,
                })
        else:
            return result.chat_output["content"]
```

### Text Completion

```python
import asyncio
import opengradient as og

llm = og.LLM(private_key="0x...")
llm.ensure_opg_approval(min_allowance=5)

result = asyncio.run(llm.completion(
    model=og.TEE_LLM.GPT_5,
    prompt="The capital of France is",
    max_tokens=50,
    temperature=0.0,
))
print(result.completion_output)
```

### LangChain ReAct Agent

```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
import opengradient as og

llm = og.agents.langchain_adapter(
    private_key="0x...",
    model_cid=og.TEE_LLM.GPT_5,
    max_tokens=300,
)

@tool
def lookup(query: str) -> str:
    """Look up information."""
    return "result"

agent = create_react_agent(llm, [lookup])
result = agent.invoke({"messages": [("user", "Find info about X")]})
print(result["messages"][-1].content)
```

### On-Chain ONNX Inference (Alpha)

```python
import opengradient as og

alpha = og.Alpha(private_key="0x...")

result = alpha.infer(
    model_cid="QmbUqS93oc4JTLMHwpVxsE39mhNxy6hpf6Py3r9oANr8aZ",
    inference_mode=og.InferenceMode.VANILLA,
    model_input={"input": [1.0, 2.0, 3.0]},
)
print(result.model_output)
print(result.transaction_hash)
```

### Digital Twins

Digital twins are digital clones of people. You can create your own digital twin or browse existing ones on https://twin.fun. In order to chat to a twin, you or the developer needs to get the twin's unique ID.

```python
import opengradient as og

twins = og.Twins(api_key="your-key")

result = twins.chat(
    twin_id="0x1abd463fd6244be4a1dc0f69e0b70cd5",
    model=og.TEE_LLM.GROK_4_1_FAST_NON_REASONING,
    messages=[{"role": "user", "content": "What do you think about AI?"}],
    max_tokens=1000,
)
print(result.chat_output["content"])
```

### Model Hub: Upload a Model

```python
import opengradient as og

hub = og.ModelHub(email="you@example.com", password="...")

repo = hub.create_model(
    model_name="my-model",
    model_desc="A prediction model",
    version="1.0.0",
)
upload = hub.upload(
    model_path="./model.onnx",
    model_name=repo.name,
    version=repo.initialVersion,
)
print(f"Model CID: {upload.modelCid}")
```

## Return Types

- **`TextGenerationOutput`**: `chat_output` (dict), `completion_output` (str), `finish_reason`, `transaction_hash`, `payment_hash`, `tee_signature`, `tee_timestamp`, `tee_id`, `tee_endpoint`, `tee_payment_address`
- **`TextGenerationStream`**: async iterable of `StreamChunk` objects (use `async for`)
- **`StreamChunk`**: `choices[0].delta.content`, `choices[0].delta.tool_calls`, `usage` (final chunk only), `is_final`, `tee_signature`, `tee_timestamp`
- **`InferenceResult`**: `model_output` (dict of np.ndarray), `transaction_hash`
- **`ModelRepository`**: `name`, `initialVersion`
- **`FileUploadResult`**: `modelCid`, `size`

## `llm.chat()` Full Signature

```python
async def chat(
    self,
    model: TEE_LLM,
    messages: List[Dict],
    max_tokens: int = 100,
    stop_sequence: Optional[List[str]] = None,
    temperature: float = 0.0,
    tools: Optional[List[Dict]] = None,
    tool_choice: Optional[str] = None,
    x402_settlement_mode: x402SettlementMode = x402SettlementMode.BATCH_HASHED,
    stream: bool = False,
) -> Union[TextGenerationOutput, AsyncGenerator[StreamChunk, None]]:
```

## Guidelines

1. Always call `llm.ensure_opg_approval(min_allowance=...)` before the first LLM inference.
2. `llm.chat()` and `llm.completion()` are **async** — use `await` or `asyncio.run()`.
3. Handle `finish_reason`: `"stop"` / `"length"` = text response, `"tool_calls"` = function calls.
4. For streaming, use `async for` and check `chunk.choices[0].delta.content is not None` before printing.
5. In tool-calling loops, append `result.chat_output` as the assistant message, then append each tool result with `role: "tool"` and matching `tool_call_id`.
6. Use environment variables or config files for private keys — never hardcode them.
7. If you are unsure about a specific API detail, read the source files in the SDK.
