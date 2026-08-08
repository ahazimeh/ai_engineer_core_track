# LLM Engineering — Week 1 Summary

---

## 1. Environment Setup

- **Repo:** `github.com/ed-donner/llm_engineering`
- **Package manager:** `uv` — install it, run `uv self update`, then `uv sync` inside the repo to pull all dependencies
- **Editor:** Cursor
- **OpenAI key:** Generate at `platform.openai.com` → add balance → create key → save to `.env` file

---

## 2. Running Models Locally with Ollama

- **Site:** `ollama.com`
- Pull and run a model: `ollama run gemma3:270m` (small, good for testing)
- Other models tried: `phi3`, `gpt-oss` (very heavy)
- If Ollama is already running as a process: `taskkill /F /IM ollama.exe` then `ollama serve`
- Verify it's running: `requests.get("http://localhost:11434").content` → returns `b'Ollama is running'`
- Use the OpenAI SDK to talk to local Ollama models:

```python
ollama = OpenAI(base_url="http://localhost:11434/v1", api_key='ollama')  # api_key value doesn't matter locally
response = ollama.chat.completions.create(model="llama3.2", messages=[...])
```

---

## 3. System vs User Prompt

| Role | Purpose | Authority |
|---|---|---|
| `system` | Sets rules, persona, context for the whole conversation | High — model treats it as strict instructions |
| `user` | The actual message/question from the human | Lower — model responds to it, doesn't blindly obey |
| `assistant` | Previous AI responses — used to maintain conversation history | Replayed context |

**Key insight:** The model was trained to behave differently per role. Putting instructions in `user` instead of `system` weakens their authority — the model may partially ignore them or break character in follow-up turns.

---

## 4. Calling LLM APIs

### Via OpenAI SDK (recommended)
```python
response = openai.chat.completions.create(model="gpt-4.1-mini", messages=[...])
response.choices[0].message.content
```

### Via raw HTTP (what the SDK does under the hood)
```python
headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}
payload = {"model": "gpt-5-nano", "messages": [...]}
response = requests.post("https://api.openai.com/v1/chat/completions", headers=headers, json=payload)
response.json()["choices"][0]["message"]["content"]
```

### OpenAI SDK works for other providers too
```python
# Gemini
gemini = OpenAI(base_url=GEMINI_BASE_URL, api_key=google_api_key)

# Ollama (local)
ollama = OpenAI(base_url="http://localhost:11434/v1", api_key='ollama')
```

---

## 5. Streaming Responses

When `stream=True`, the API returns a generator instead of a full response. The `for` loop **blocks** on each `next()` call, waiting on the network socket until the next token arrives (SSE format). No polling — just blocking I/O.

```python
stream = openai.chat.completions.create(model="gpt-4.1-mini", messages=[...], stream=True)
response = ""
display_handle = display(Markdown(""), display_id=True)
for chunk in stream:
    response += chunk.choices[0].delta.content or ''
    update_display(Markdown(response), display_id=display_handle.display_id)
```

---

## 6. Tokens

- **What they are:** Chunks of text — not characters, not full words. A middle ground.
- **Rule of thumb:** 1 token ≈ 4 characters ≈ 0.75 words
- **Why not characters?** Too much work for the network
- **Why not full words?** Enormous vocab, rare words get dropped
- **Tokens** handle word stems elegantly and keep vocab manageable

```python
import tiktoken
encoding = tiktoken.encoding_for_model("gpt-4.1-mini")
tokens = encoding.encode("Hi my name is Ed")
# Decode back
for token_id in tokens:
    print(f"{token_id} = {encoding.decode([token_id])}")
```

Tokenizer playground: `platform.openai.com/tokenizer`

---

## 7. Context Window

The **maximum number of tokens** the model can consider when generating the next token. After each token is generated, the entire sequence (including the new token) is fed back in to generate the next one. This is why long conversations get expensive.

---

## 8. API Costs & Caching

- You pay per token — input + output
- **Reasoning tokens** (internal thinking) also cost
- **Caching:** If you send the same input multiple times in a short window, the provider may charge less due to prompt caching

---

## 9. The 3 Types of LLM

| Type | Description |
|---|---|
| **Base** | Raw pretrained model, just predicts next token |
| **Chat / Instruct** | Fine-tuned to follow instructions and have conversations |
| **Reasoning / Thinking** | Generates intermediate reasoning tokens before answering (inference-time scaling) |

---

## 10. Making Models Perform Better

### Training-time scaling
- Train on more data / larger model

### Inference-time scaling
- **Reasoning:** Let the model generate thinking tokens before the answer (e.g., use "wait", chain-of-thought)
- **RAG (Retrieval-Augmented Generation):** Inject relevant external information into the input context — more useful input = better output

---

## 11. Prompting Techniques

- **Zero-shot:** Just ask the question
- **One-shot:** Provide 1 example with the answer
- **Multi-shot:** Provide several examples with answers — model learns the pattern

---

## 12. Structured Output (JSON mode)

Force the model to always return valid JSON — it constrains token selection at inference time:

```python
response_format={"type": "json_object"}
```

---

## 13. Key Terminology

| Term | Meaning |
|---|---|
| **Frontier model** | Closed-source, state-of-the-art model (GPT-5, Claude, Gemini) |
| **Distillation** | Using a large model to generate synthetic training data for a smaller model |
| **Parameters** | The learned weights of the model — what gets updated during training |
| **Inference** | Running the model to generate output (as opposed to training) |
| **RAG** | Retrieval-Augmented Generation — supplying external info in the prompt |
| **SSE** | Server-Sent Events — the streaming format used by OpenAI's API |

---

## 14. The Paper That Started It All

**"Attention Is All You Need"** — Vaswani et al.
Introduced the **Transformer architecture**: relies entirely on self-attention instead of RNNs or convolutions. Foundation for BERT, GPT, and virtually every modern LLM.

---

## 15. Training Data for LLMs

Models are trained on:
- Natural language text
- Markdown
- JSON

This is why prompting in Markdown and requesting JSON output works so well — the model has seen enormous amounts of both.
