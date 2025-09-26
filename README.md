
# 🚀 Flight AI Assistant  

## Table of Contents
1. [What is this repo?](#what-is-this-repo)  
2. [Why function‑calling tools are the secret sauce?](#why-function-calling-tools-are-the-secret-sauce)  
3. [Core Architecture – A Quick Tour](#core-architecture-a-quick-tour)  
4. [New Feature Spotlight: **Discount‑For‑Sales** Tool ✨](#new-feature-spotlight-discount‑for‑sales-tool)  
5. [Getting Started – Run It Locally in 5 Minutes](#getting-started--run-it-locally-in-5-minutes)  
6. [Extending the Assistant – Add Your Own Magic](#extending-the-assistant--add-your-own-magic)  
7. [Best‑Practices, Gotchas & Tips](#best‑practices-gotchas--tips)  
8. [License & Contributing](#license--contributing)  

---

## What is this repo?

`Flight AI Assistant` is a **playful yet powerful** chatbot that demonstrates how to blend large‑language‑model chat (OpenAI GPT‑4o‑mini, Anthropic Claude, Google Gemini) with **function‑calling tools** to fetch **real‑time, reliable data**.  
Think of it as a virtual airline agent that never makes up ticket prices—because the prices live in a tiny, well‑structured Python dictionary (or a future live API).

> **TL;DR:** *Chat with a model, let it call Python functions for you, and get spot‑on answers.*  

---

## Why function‑calling tools are the secret sauce?

| ✅ Benefit | 🎯 What it means for you |
|-----------|--------------------------|
| **Separation of concerns** | The system prompt handles tone (courteous, one‑sentence answers). The tools handle the hard facts (prices, discounts). |
| **Zero hallucinations** | The model never guesses a price; it *must* call `get_ticket_price` or `get_discounted_price`. |
| **Structured, JSON‑friendly data** | Responses are machine‑readable, no fragile regex parsing required. |
| **Easy extensibility** | Add a new function → update its JSON schema → append to `tools`. No system‑prompt rewrites! |
| **Auditability** | Every tool call can be logged for compliance or debugging. |
| **Security by design** | The model never gets direct DB or network access—only the curated functions you expose. |

---

## Core Architecture – A Quick Tour

`````
User → Gradio UI → chat() → OpenAI API
           ↘ (no tool)               ↘ (tool call)
             ↘                         ↘
        OpenAI response          handle_tool_call()
             ↘                         ↘
   UI displays answer   tool response → second OpenAI call → final answer
`````

* **`system_message`** – sets the personality: short, courteous, always accurate.  
* **Tool definitions** – JSON schemas (`price_function`, `discount_function`) describing the callable Python functions.  
* **`chat()`** – orchestrates the conversation, decides when a tool call is needed, and stitches everything together.  
* **`handle_tool_calls()`** – parses the model’s tool request, runs the real Python function, and builds the tool‑response payload.  
* **Gradio UI** – a sleek web front‑end (`gr.ChatInterface`) for instant testing.

---

## New Feature Spotlight: **Discount‑For‑Sales** Tool ✨

### What’s new?
We’ve added a **second tool** that returns **discounted ticket prices** for special promotions (Black Friday, Christmas sales, etc.).  

### How it works
1. **User asks** “What’s the sale price for a Tokyo ticket?”  
2. The model sees the `discount_function` schema, decides to call it.  
3. `handle_tool_calls()` runs `get_discounted_price(city)` against the `discount_prices` dict.  
4. The model receives the JSON result and crafts a short, friendly reply like:  

> “A Tokyo round‑trip is currently on sale for **$999** 🎉”

### Tool definition (for nerds)

```python
discount_function = {
    "name": "get_discounted_price",
    "description": "Get the discounted price of a return ticket to the destination city. Use this during sales periods (e.g., Black Friday, Christmas).",
    "parameters": {
        "type": "object",
        "properties": {
            "destination_city": {
                "type": "string",
                "description": "The city the customer wants to travel to."
            }
        },
        "required": ["destination_city"],
        "additionalProperties": False,
    },
}
tools += [{"type": "function", "function": discount_function}]
```

Now the assistant can **smoothly switch** between regular and discounted pricing without any extra prompt gymnastics.

---

## Getting Started – Run It Locally in 5 Minutes

1. **Clone the repo**  
   ```bash
   git clone https://github.com/AsutoshaNanda/Flight-AI-Assistant.git
   cd Flight-AI-Assistant
   ```

2. **Create a virtual environment** (optional but recommended)  
   ```bash
   python -m venv venv && source venv/bin/activate   # macOS/Linux
   .\venv\Scripts\activate                          # Windows
   ```

3. **Install dependencies**  
   ```bash
   pip install openai anthropic google-generativeai gradio python-dotenv beautifulsoup4
   ```

4. **Add your API keys** in a `.env` file at the project root:  
   ```
   OPENAI_API_KEY=sk-...
   ANTHROPIC_API_KEY=sk-ant-...
   GOOGLE_API_KEY=AIzaSy...
   ```

5. **Launch the notebook or script**  
   *Notebook*: `jupyter lab` → open `Flight AI Assistant.ipynb` → run all cells.  
   *Script*: copy the notebook code into a `app.py` and run `python app.py`.

6. **Open the Gradio UI** (usually at `http://127.0.0.1:7860`) and start chatting!  

---

## Extending the Assistant – Add Your Own Magic

1. **Write a Python function** – e.g., `def get_flight_status(flight_number): ...`  
2. **Create a matching tool schema** (name, description, JSON parameters).  
3. **Append it to the `tools` list**.  
4. **Update `handle_tool_calls()`** to route the new `function.name` to your Python function.  

*Tip:* Keep the function **pure** (no side‑effects) and **validate inputs** (whitelist city names, regex flight numbers) for security.

---

## Best‑Practices, Gotchas & Tips

| 🎯 Area | ✅ Recommendation |
|--------|-------------------|
| **Infinite loops** | Limit `max_tool_calls` per turn (e.g., 2) to avoid recursion. |
| **Schema mismatches** | Test tool schemas in OpenAI Playground before coding. |
| **User‑supplied data** | Sanitize/whitelist all arguments inside your functions. |
| **Deprecation warnings** | Use `type='messages'` in `gr.ChatInterface` (as shown). |
| **Rate limits** | Implement exponential back‑off & exponential jitter for API calls. |
| **Debugging** | Print/log the raw `tool_calls` payload to see exactly what the model sent. |
| **Fun factor** | Sprinkle emojis, friendly language, and short sentences (the system prompt already enforces it!). |

---

## License & Contributing

This project is licensed under the **Apache License 2.0** – see the `LICENSE` file for full details.  

💡 **Contributions welcome!** Feel free to fork, add new tools (e.g., flight status, baggage allowance), improve the UI, or write tests. Open an issue or submit a pull request, and let’s make this little airline agent even smarter together.

---

### Happy coding ✈️💸

Please contribute to this and hope this repo would be some help of yours.
