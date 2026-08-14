# LLM Engineering — Week 2 Summary

## 1. Inference-Time vs Training-Time
- **Training-time scaling:** more data, bigger model
- **Inference-time scaling:** reasoning effort control at call time via `reasoning_effort` param (e.g., `"minimal"`)

## 2. Routers & Abstraction Layers
Two ways to work across multiple LLM providers without rewriting your code:
- **Router** (e.g., OpenRouter): a remote process you call, it decides which provider to forward to
- **Abstraction Layer** (e.g., LangChain, LiteLLM): a local framework with a unified API that handles provider differences itself — LiteLLM also exposes token usage and cost out of the box

**Caching tip:** OpenAI caches automatically if the start of the prompt matches — put static content first. Anthropic requires explicit cache priming (costs 25% more but saves 10x on cache hits).

**Multi-model conversations tip:** If more than 2 models are chatting, use a single system prompt with the full conversation history and ask the AI to respond as the next participant.

## 3. Tool Calling
- The LLM only generates tokens — **your code** actually calls the tool and feeds the result back
- Common use cases: fetch data, take actions, run calculations, modify UI
- Flow: LLM responds with `finish_reason == "tool_calls"` → you call the tool → append result with `tool_call_id` → call LLM again

**Key evolutions to get right:**
- Handle **multiple tool calls** in one response by looping over `message.tool_calls`
- Support **sequential tool calling** by changing `if` to `while` — the LLM keeps calling tools until it's done, then stops naturally (you can add a cap if cost is a concern)
- Pass `tools=tools` again on every re-call inside the loop

## 4. Agents
An agent is an LLM that **controls the workflow** — it runs tools in a loop autonomously to achieve a goal.

**Core features:** memory/persistence, planning, autonomy, orchestration via tools

**Multimodal agent capabilities covered:**
- **Image generation:** `openai.images.generate(...)` → decode base64 → PIL Image
- **Audio/TTS:** `openai.audio.speech.create(...)` → returns audio bytes
- Combine both into a single chat loop: LLM replies → TTS the reply → generate image if a city was mentioned

## 5. One-Liners Worth Remembering
- `yield from result` == `for chunk in result: yield chunk`
- In Python you can resolve a function by name from a string and call it dynamically — useful for tool dispatch
- Streaming + tool calling works but requires significantly more code
