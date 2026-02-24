# OpenGradient Plugins for Claude Code

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that provides expert guidance for the [OpenGradient Python SDK](https://docs.opengradient.ai) - a decentralized AI inference platform with verified execution and on-chain settlement.

## What is OpenGradient?

[OpenGradient](https://opengradient.ai) is a decentralized AI inference network that runs models inside Trusted Execution Environments (TEEs). Every inference call produces a cryptographic proof of correct execution, and payment is settled on-chain via the [x402 protocol](https://docs.opengradient.ai) on Base Sepolia. This means you can call models from OpenAI, Anthropic, Google, and xAI through a single unified API — with verifiable, tamper-proof results.

## What this plugin does

When installed, this plugin adds the `/opengradient-sdk` skill to Claude Code. It gives Claude deep knowledge of the OpenGradient Python SDK so it can help you write correct, idiomatic code for:

- **Verified LLM inference** — Chat completions via TEE with cryptographic attestation
- **x402 payment settlement** — On-chain receipts on Base Sepolia (`SETTLE`, `SETTLE_METADATA`, `SETTLE_BATCH`)
- **Multi-provider models** — Access OpenAI, Anthropic, Google, and xAI models through one API
- **Streaming & tool calling** — Full multi-turn agent loops with function definitions
- **On-chain ONNX model inference** (alpha) — Run ONNX models directly on-chain
- **LangChain integration** — Build ReAct agents using the OpenGradient adapter
- **Digital twins chat** — Specialized chat API for digital twin interactions
- **Model Hub** — Upload, manage, and query models

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and running
- A Base Sepolia wallet funded with OPG tokens (required for LLM inference)

### Install the plugin

```bash
claude plugin add --marketplace https://github.com/OpenGradient/opengradient-claude-marketplace
```

This registers the OpenGradient marketplace and installs the plugin, making the `/opengradient-sdk` skill available in your Claude Code sessions.

## Usage

Invoke the skill directly:

```
/opengradient-sdk write a streaming chat completion with tool calling
```

Or just describe what you want to build with OpenGradient - Claude will automatically use the skill when relevant.

### Example prompts

```
/opengradient-sdk create a multi-turn agent that uses tool calling to fetch weather data
/opengradient-sdk build a LangChain ReAct agent with OpenGradient
/opengradient-sdk upload an ONNX model to the model hub
/opengradient-sdk show me how to use x402 settlement with metadata
```

## Links

- [OpenGradient Documentation](https://docs.opengradient.ai)
- [OpenGradient Website](https://opengradient.ai)
- [Claude Code Plugins Guide](https://docs.anthropic.com/en/docs/claude-code/plugins)
