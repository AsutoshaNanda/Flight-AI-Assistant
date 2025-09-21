# Flight AI Assistant 

## Table of Contents
1. [Project Summary](#project-summary)  
2. [Why Use Function‑Calling Tools?](#why-use-function-calling-tools)  
   - 2.1 [Separation of Concerns](#separation-of-concerns)  
   - 2.2 [Accurate, Structured Data](#accurate-structured-data)  
   - 2.3 [Reduced Hallucinations](#reduced-hallucinations)  
   - 2.4 [Extensibility & Maintenance](#extensibility--maintenance)  
   - 2.5 [Security & Auditing](#security--auditing)  
3. [Architecture Overview](#architecture-overview)  
   - 3.1 [Core Components](#core-components)  
   - 3.2 [Data Flow Diagram](#data-flow-diagram)  
4. [Implementation Details](#implementation-details)  
   - 4.1 [System Prompt vs. Tools](#system-prompt-vs-tools)  
   - 4.2 [Tool Definition (OpenAI Function‑Calling)](#tool-definition-openai-function-calling)  
   - 4.3 `handle_tool_call` Logic  
   - 4.4 Gradio UI Integration  
5. [Running the Assistant Locally](#running-the-assistant-locally)  
6. [Extending the Assistant](#extending-the-assistant)  
   - 6.1 Adding New Functions  
   - 6.2 Multi‑Model Support  
   - 6.3 Deploying to Hugging Face Spaces  
7. [Best Practices & Gotchas](#best-practices--gotchas)  
8. [License](#license)  

---

## Project Summary
**Flight AI Assistant** is a lightweight chatbot built with **OpenAI’s GPT‑4o‑mini**, **Anthropic Claude**, and **Google Gemini** APIs. It demonstrates how to enrich a conversational AI with *function‑calling tools* that provide real‑time, authoritative data (e.g., ticket prices) rather than relying on the model’s internal knowledge.

The repository contains:
- `Flight AI Assistant.ipynb` – a Jupyter notebook that sets up the environment, defines the system prompt, registers tools, and launches a Gradio chat interface.
- `README.md` – this documentation.
- `LICENSE` – Apache 2.0 license.

---

## Why Use Function‑Calling Tools?

### 2.1 Separation of Concerns
- **System message**: Sets the *behaviour* and *tone* of the assistant (courteous, short answers, etc.).
- **Tools**: Encapsulate *domain‑specific logic* (price lookup, flight status, booking, etc.) that the LLM should **never** try to fabricate.

By keeping these responsibilities separate, you avoid cluttering the system prompt with procedural instructions and make the assistant’s behaviour easier to reason about.

### 2.2 Accurate, Structured Data
Function‑calling returns **JSON‑compatible** payloads. This guarantees:
- Correct data types (`string`, `number`, etc.).
- Predictable schema for downstream processing.
- No need for fragile text‑parsing heuristics.

### 2.3 Reduced Hallucinations
When the model is asked a factual question (e.g., “How much does a ticket to Tokyo cost?”) it might *hallucinate* a price based on its training data. By forcing the model to **call a tool** for that information, the answer is always sourced from the **authoritative `ticket_prices` dictionary** (or a live API in a production system).

### 2.4 Extensibility & Maintenance
Adding a new capability is as simple as:
1. Defining a new function (Python code).  
2. Declaring its JSON schema in a *tool* definition.  
3. Updating the `tools` list.  

No modifications to the system message are required, which means you can evolve the feature set without re‑tuning the model’s persona.

### 2.5 Security & Auditing
- **Audit trail**: Every tool call is logged (the model sends a `tool_calls` object, we handle it, and we can store the request/response).  
- **Least‑privilege**: Tools expose only the data they need. The model never gets direct DB or network access.

---

## Architecture Overview

### 3.1 Core Components
| Component | Responsibility |
|-----------|----------------|
| `system_message` | Sets conversational style and high‑level policies. |
| `price_function` (tool) | JSON schema describing the `get_ticket_price` function. |
| `tools` list | Passed to OpenAI’s `chat.completions.create` so the model can invoke them. |
| `chat()` | Orchestrates message history, decides whether a tool call is needed, and returns the final answer. |
| `handle_tool_call()` | Parses the tool call, runs the Python function, formats the tool response. |
| Gradio UI (`gr.ChatInterface`) | Front‑end for interactive testing. |

### 3.2 Data Flow Diagram
```
User → Gradio UI → chat() → OpenAI API
          ↘︎                     ↘︎
        (no tool)            (tool call)
          ↘︎                     ↘︎
   OpenAI response      handle_tool_call()
          ↘︎                     ↘︎
   Chat UI displays   tool response appended → second OpenAI call
```

---

## Implementation Details

### 4.1 System Prompt vs. Tools
The **system prompt** is static text that tells the model *how* to behave. It **does not** contain any data‑driven logic such as “price of a ticket to London is $799”.  
Instead, the **tool** `get_ticket_price` is the *single source of truth* for ticket pricing, and the model is instructed (via the `tools` parameter) to call it whenever the user asks about prices.

### 4.2 Tool Definition (OpenAI Function‑Calling)
```python
price_function = {
    'name': 'get_ticket_price',
    'description': "Get the price of a return ticket to the destination city. Call this whenever you need to know the ticket price, for example when a customer asks 'How much is a ticket to this city'",
    'parameters': {
        'type': 'object',
        'properties': {
            'destination_city': {
                'type': 'string',
                'description': "The city that the customer wants to travel to"
            }
        },
        'required': ['destination_city'],
        'additionalProperties': False
    }
}
tools = [{'type': 'function', 'function': price_function}]
```
- The schema ensures the model passes a *single* `destination_city` string.
- `additionalProperties: False` prevents unexpected fields.

### 4.3 `handle_tool_call` Logic
1. Extract the first `tool_calls` element.  
2. Parse its `function.arguments` JSON.  
3. Call the Python function `get_ticket_price(city)`.  
4. Build a **tool response** object (`role: "tool"`) that contains a JSON payload with both the city and price.  
5. Append the tool response and the original model message to the conversation, then issue a **second** OpenAI request to generate the final user‑facing reply.

### 4.4 Gradio UI Integration
Two launch configurations are shown:
- `gr.ChatInterface(fn=chat).launch(share=True)` – creates a public share link via Gradio Cloud.
- `gr.ChatInterface(fn=chat, type='messages').launch()` – uses the newer `type='messages'` API to avoid deprecation warnings.

Both provide an interactive web UI for testing the assistant.

---

## Running the Assistant Locally

1. **Clone the repo**  
   ```bash
   git clone https://github.com/AsutoshaNanda/Flight-AI-Assistant.git
   cd Flight-AI-Assistant
   ```

2. **Install dependencies** (preferably in a virtual environment)  
   ```bash
   pip install -r requirements.txt   # create this file if you wish
   # Minimal deps:
   pip install openai anthropic google-generativeai gradio python-dotenv beautifulsoup4
   ```

3. **Add your API keys** to a `.env` file:  
   ```
   OPENAI_API_KEY=sk-...
   ANTHROPIC_API_KEY=sk-ant-...
   GOOGLE_API_KEY=AIzaSy...
   ```

4. **Start the notebook** (`jupyter lab` or `jupyter notebook`) and run all cells, or copy the code into a `.py` script and execute it.

5. Open the displayed local URL (e.g., `http://127.0.0.1:7866`) to interact.

---

## Extending the Assistant

### 6.1 Adding New Functions
- Write a Python function (e.g., `def get_flight_status(flight_number): ...`).
- Create a matching **tool schema** with name, description, and JSON parameters.
- Append the new tool to the `tools` list.
- Update `handle_tool_call` to route the appropriate `tool_calls` to the new function.

### 6.2 Multi‑Model Support
The notebook already imports `anthropic` and `google.generativeai`. You can switch the model by:
```python
response = anthropic.Anthropic().messages.create(
    model="claude-3-5-sonnet-20240620",
    messages=messages,
    tools=tools
)
```
or similarly for Gemini. Keep the interface uniform by abstracting the request into a helper function.

### 6.3 Deploying to Hugging Face Spaces
```bash
gradio deploy
```
- This bundles the notebook (or script) into a Space.
- Enable GPU if you plan to use Gemini or large Claude models.

---

## Best Practices & Gotchas

| Issue | Recommendation |
|-------|----------------|
| **Tool call loops** | Guard against infinite recursion: limit the number of tool calls per turn (`max_tool_calls=2`). |
| **Schema mismatches** | Test the function schema with OpenAI’s Playground to ensure the model can generate the expected JSON. |
| **User‑supplied malicious input** | Validate arguments inside each Python function before using them (e.g., whitelist city names). |
| **Deprecation warnings** | Use `type='messages'` in `gr.ChatInterface` to avoid the “tuples format” warning. |
| **Rate limits** | Implement exponential back‑off if you hit API limits, especially when deploying publicly. |

---

## License
This project is licensed under the **Apache License 2.0** – see the `LICENSE` file for full details.

--- 

Please Contribute to this project and have a great day.
