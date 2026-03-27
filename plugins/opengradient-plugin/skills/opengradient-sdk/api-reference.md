# OpenGradient SDK — API Reference

Detailed reference for the OpenGradient Python SDK. Use this alongside
the main SKILL.md when you need specifics about parameters, return types,
or less common features.

---

## Installation

```bash
# Requires Python >=3.11
pip install opengradient==0.9.4
```

---

## Client Initialization

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

### `og.LLM`

```python
og.LLM(
    private_key: str,                    # Required — Base Sepolia wallet (holds OPG tokens)
    rpc_url: str = DEFAULT_RPC_URL,      # Optional — blockchain RPC endpoint
    tee_registry_address: str = DEFAULT_TEE_REGISTRY_ADDRESS,
)
```

Also available: `og.LLM.from_url(private_key, llm_server_url)` for development/self-hosted TEE servers.

### `og.Alpha`

```python
og.Alpha(
    private_key: str,                    # Required — OpenGradient testnet key
    rpc_url: str = DEFAULT_RPC_URL,
    inference_contract_address: str = DEFAULT_INFERENCE_CONTRACT_ADDRESS,
    api_url: str = DEFAULT_API_URL,
)
```

### `og.ModelHub`

```python
og.ModelHub(
    email: str = None,                   # Optional — Model Hub email
    password: str = None,                # Optional — Model Hub password
)
```

### `og.Twins`

```python
og.Twins(
    api_key: str,                        # Required — Digital twins API key
)
```

---

## LLM API — `og.LLM`

### `ensure_opg_approval()`

```python
llm.ensure_opg_approval(
    min_allowance: float,                # Minimum OPG allowance required
    approve_amount: float = None,        # Amount to approve (defaults to 2 * min_allowance)
) -> Permit2ApprovalResult
```

Approve OPG token spending for x402 payments. Idempotent — only sends a
transaction when the current allowance drops below `min_allowance`.
Must be called before the first `chat()` call.

**Returns:** `Permit2ApprovalResult` with fields:
- `allowance_before: int`
- `allowance_after: int`
- `tx_hash: Optional[str]` (None if no approval was needed)

### `chat()` (async)

```python
await llm.chat(
    model: og.TEE_LLM,
    messages: list[dict],                # OpenAI-style message dicts
    max_tokens: int = 100,
    temperature: float = 0.0,
    stop_sequence: list[str] = None,
    stream: bool = False,
    tools: list[dict] = None,            # Function definitions
    tool_choice: str = None,             # "auto", "none", or specific
    x402_settlement_mode: og.x402SettlementMode = og.x402SettlementMode.BATCH_HASHED,
) -> TextGenerationOutput | AsyncGenerator[StreamChunk, None]
```

**IMPORTANT**: This is an **async** method. Use `await` inside an async function or `asyncio.run()` for top-level calls.

**Messages format:**
```python
[
    {"role": "system", "content": "System prompt"},
    {"role": "user", "content": "User message"},
    {"role": "assistant", "content": "Previous response"},
    {"role": "tool", "tool_call_id": "call_123", "content": "Tool output"},
]
```

### `completion()` (async)

```python
await llm.completion(
    model: og.TEE_LLM,
    prompt: str,
    max_tokens: int = 100,
    stop_sequence: list[str] = None,
    temperature: float = 0.0,
    x402_settlement_mode: og.x402SettlementMode = og.x402SettlementMode.BATCH_HASHED,
) -> TextGenerationOutput
```

Non-chat text completion with a raw prompt string.

---

## Return Types

### `TextGenerationOutput`

Returned by `llm.chat()` when `stream=False`, and by `llm.completion()`.

| Field | Type | Description |
|-------|------|-------------|
| `chat_output` | `dict` | `{"role": "assistant", "content": "...", "tool_calls": [...]}` |
| `completion_output` | `str` | Text (completions API only) |
| `finish_reason` | `str` | `"stop"`, `"length"`, or `"tool_calls"` |
| `transaction_hash` | `str` | Blockchain tx hash |
| `payment_hash` | `str` | x402 settlement proof |
| `tee_signature` | `str` | TEE attestation signature |
| `tee_timestamp` | `str` | TEE execution timestamp |
| `tee_id` | `str` | TEE node identifier |
| `tee_endpoint` | `str` | TEE endpoint URL |
| `tee_payment_address` | `str` | TEE payment address |

### `TextGenerationStream`

Returned by `llm.chat()` when `stream=True`. **Async** iterable of `StreamChunk`. Use `async for`.

### `StreamChunk`

| Field | Type | Description |
|-------|------|-------------|
| `choices` | `list[StreamChoice]` | Delta updates |
| `choices[0].delta.content` | `str or None` | Incremental text |
| `choices[0].delta.role` | `str or None` | Role (first chunk only) |
| `choices[0].delta.tool_calls` | `list or None` | Incremental tool calls |
| `model` | `str` | Model identifier |
| `usage` | `StreamUsage or None` | Token counts (final chunk only) |
| `is_final` | `bool` | `True` on last chunk |
| `tee_signature` | `str or None` | TEE attestation signature |
| `tee_timestamp` | `str or None` | TEE execution timestamp |
| `tee_id` | `str or None` | TEE node identifier |
| `tee_endpoint` | `str or None` | TEE endpoint URL |
| `tee_payment_address` | `str or None` | TEE payment address |

### `InferenceResult`

Returned by `alpha.infer()`.

| Field | Type | Description |
|-------|------|-------------|
| `model_output` | `dict[str, np.ndarray]` | Model outputs |
| `transaction_hash` | `str` | On-chain tx hash |

### `ModelRepository`

Returned by `hub.create_model()`.

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Repository name |
| `initialVersion` | `str` | Initial version string |

### `FileUploadResult`

Returned by `hub.upload()`.

| Field | Type | Description |
|-------|------|-------------|
| `modelCid` | `str` | Content-addressed model identifier |
| `size` | `int` | Upload size in bytes |

---

## Models — `og.TEE_LLM`

### OpenAI
- `GPT_4_1_2025_04_14`
- `O4_MINI`
- `GPT_5`
- `GPT_5_MINI`
- `GPT_5_2`

### Anthropic
- `CLAUDE_SONNET_4_5`
- `CLAUDE_SONNET_4_6`
- `CLAUDE_HAIKU_4_5`
- `CLAUDE_OPUS_4_5`
- `CLAUDE_OPUS_4_6`

### Google
- `GEMINI_2_5_FLASH`
- `GEMINI_2_5_PRO`
- `GEMINI_2_5_FLASH_LITE`
- `GEMINI_3_PRO`
- `GEMINI_3_FLASH`

### xAI (Grok)
- `GROK_4`
- `GROK_4_FAST`
- `GROK_4_1_FAST`
- `GROK_4_1_FAST_NON_REASONING`

---

## Settlement Modes — `og.x402SettlementMode`

| Mode | Value | Description |
|------|-------|-------------|
| `PRIVATE` | `"private"` | Payment only, no data on-chain — maximum privacy |
| `BATCH_HASHED` | `"batch"` | Aggregated into Merkle tree — most cost-efficient (**default**) |
| `INDIVIDUAL_FULL` | `"individual"` | Full input/output recorded on-chain — maximum transparency |

---

## Inference Modes — `og.InferenceMode` (Alpha)

| Mode | Description |
|------|-------------|
| `VANILLA` | Standard execution |
| `TEE` | Trusted Execution Environment |
| `ZKML` | Zero-Knowledge ML |

---

## Alpha API — `og.Alpha`

### `infer()`

```python
alpha.infer(
    model_cid: str,
    inference_mode: og.InferenceMode,
    model_input: dict,
    max_retries: int = None,
) -> InferenceResult
```

### `new_workflow()`

```python
alpha.new_workflow(
    model_cid: str,
    input_query: og.HistoricalInputQuery,
    input_tensor_name: str,
    scheduler_params: og.SchedulerParams = None,
) -> str  # contract address
```

### `run_workflow(contract_address: str) -> ModelOutput`

### `read_workflow_result(contract_address: str) -> ModelOutput`

### `read_workflow_history(contract_address: str, num_results: int) -> list[ModelOutput]`

---

## Workflow Input Queries

### `og.HistoricalInputQuery`

```python
og.HistoricalInputQuery(
    base="ETH",
    quote="USD",
    total_candles=10,
    candle_duration_in_mins=60,
    order=og.CandleOrder.DESCENDING,
    candle_types=[og.CandleType.CLOSE],
)
```

### `og.SchedulerParams`

```python
og.SchedulerParams(
    frequency=3600,       # Seconds between runs
    duration_hours=24,    # Total duration
)
```

---

## Model Hub — `og.ModelHub`

### `create_model(model_name, model_desc, version="1.00") -> ModelRepository`

### `create_version(model_name, notes="", is_major=False) -> dict`

### `upload(model_path, model_name, version) -> FileUploadResult`

### `list_files(model_name, version) -> list[dict]`

---

## Digital Twins — `og.Twins`

### `chat()`

```python
twins.chat(
    twin_id: str,
    model: og.TEE_LLM,
    messages: list[dict],
    temperature: float = None,
    max_tokens: int = None,
) -> TextGenerationOutput
```

Browse twins at https://twin.fun.

---

## LangChain Integration

```python
llm = og.agents.langchain_adapter(
    private_key: str,
    model_cid: og.TEE_LLM,
    max_tokens: int = 300,
    x402_settlement_mode: og.x402SettlementMode = og.x402SettlementMode.BATCH_HASHED,
)
```

Returns a LangChain-compatible `BaseChatModel` that can be used with
`create_react_agent()`, chains, or any LangChain component expecting an LLM.

Also available via direct class import:
```python
from opengradient.agents import OpenGradientChatModel

llm = OpenGradientChatModel(
    private_key="0x...",
    model_cid=og.TEE_LLM.CLAUDE_SONNET_4_6,
    max_tokens=300,
)
```

Supports `bind_tools()` for function calling with LangChain agents.

---

## Tools Format (for Tool Calling)

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "function_name",
            "description": "What this function does",
            "parameters": {
                "type": "object",
                "properties": {
                    "param1": {"type": "string", "description": "..."},
                    "param2": {"type": "number"},
                },
                "required": ["param1"],
            },
        },
    }
]
```

---

## CLI Commands

```bash
opengradient config init       # Interactive setup
opengradient config show       # Display config
opengradient config clear      # Reset config
opengradient create-account    # Generate wallet
opengradient infer -m <CID> --input '<json>'
opengradient chat --model <model> --messages '<json>' --max-tokens 100
```
