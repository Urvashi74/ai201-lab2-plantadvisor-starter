# Spec: `run_agent()`

**File:** `agent.py`
**Status:** Partially pre-filled — complete the two blank fields before implementing

---

## Purpose

Orchestrate a single conversational turn for the Plant Advisor agent. Given a user message and the conversation history, call the LLM with available tools, execute any tool calls the LLM requests, and return the final text response.

This is the core of what makes Plant Advisor an *agent* rather than a simple chatbot: the ability to decide which tools to call, use their results to inform its response, and loop until it has everything it needs.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's current message |
| `history` | `list` | Gradio conversation history — list of `[user_msg, assistant_msg]` pairs |

**Output:** `str`

The agent's final text response for this turn. Should never be empty — if something goes wrong, return a user-readable fallback message.

---

## Design Decisions

*Read `specs/system-design.md` (especially the "How the Groq Tool Calling API Works" section) before reviewing these. Complete the two blank fields before writing any code.*

---

### Messages list structure

The messages list must start with the system prompt, then replay the conversation
history, then add the new user message. Gradio history is a list of `[user, assistant]`
pairs — convert each pair to two API-format dicts:

```python
messages = [{"role": "system", "content": SYSTEM_PROMPT}]

for user_msg, assistant_msg in history:
    messages.append({"role": "user", "content": user_msg})
    if assistant_msg:
        messages.append({"role": "assistant", "content": assistant_msg})

messages.append({"role": "user", "content": user_message})
```

---

### Initial LLM call

Pass the model, the messages list, the tool definitions, and `tool_choice="auto"`
so the LLM can decide whether to call a tool or respond directly:

```python
response = client.chat.completions.create(
    model=LLM_MODEL,
    messages=messages,
    tools=TOOL_DEFINITIONS,
    tool_choice="auto",
)
```

---

### Detecting tool calls in the response

The response object has a `choices` list. Index 0 gives the assistant message.
Check its `tool_calls` attribute — if it's truthy, the LLM wants to call tools:

```python
assistant_message = response.choices[0].message

if not assistant_message.tool_calls:
    # No tool calls — LLM has a final answer
    ...
```

---

### Appending the assistant message

When there are tool calls, append the full assistant message object to `messages`
**before** appending any tool results. The API requires this ordering — a tool
result message must immediately follow the assistant message that requested it:

```python
messages.append(assistant_message)  # must come first
```

---

### Executing and appending tool results

For each tool call, extract the name and arguments, call `dispatch_tool()`, and
append the result as a `"tool"` role message. The `tool_call_id` links this result
back to the specific tool call that requested it:

```python
for tool_call in assistant_message.tool_calls:
    tool_name = tool_call.function.name
    tool_args = json.loads(tool_call.function.arguments)
    tool_result = dispatch_tool(tool_name, tool_args)

    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_result,
    })
```

---

### Loop termination conditions

*The loop should stop when: (a) the LLM returns a response with no tool calls, OR (b) the MAX_TOOL_ROUNDS limit is reached. Describe how you will detect each condition and what you will return in each case.*

```
(a) No tool calls: check if tool_calls on the response message is empty or None.
    This is more reliable than checking finish_reason, which can vary by model/provider.

  if not response.choices[0].message.tool_calls:
      break  # no tool calls → return response.choices[0].message.content

(b) MAX_TOOL_ROUNDS reached: track a counter incremented each iteration. If it
    reaches MAX_TOOL_ROUNDS before the model stops calling tools, exit the loop
    and return the last text response (or a fallback message if there is none).
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
response.choices[0].message.content
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: [tool name, args]
 → Tool call: lookup_plant({'plant_name': 'calathea'})
 ← Result: {"found": true, "plant": {"display_name": "Calathea", "scientific_name": "Calathea ornata", "aliases": ["prayer plant", ...
Round 2 tool call: [tool name, args] (if any)
→ Tool call: get_seasonal_conditions(None)
  ← Result: {"name": "Summer", "months": ["June", "July", "August"], "description": "Peak growing season. Most houseplants are at th...
Final response: According to the care data for your calathea, to keep your plant happy and healthy, make sure to water it every 1-2 weeks, keeping the soil consistently moist but not soggy. It's also important to use filtered, distilled, or rainwater to prevent brown edges on the leaves. 

Calatheas prefer low to medium indirect light, so it's best to avoid direct sunlight, which can bleach and damage the decorative leaf markings. They also require high humidity, so using a humidifier, misting frequently, or grouping with other plants can help.

In terms of temperature, keep your calathea in an area with a consistent temperature between 60-80°F (15-27°C), and avoid cold drafts or temperatures below 55°F.

As we are currently in summer, it's essential to increase humidity and water your calathea consistently. Keep it out of direct sun to prevent scorching, and consider fertilizing with a balanced fertilizer once a month. 

If you notice any issues like brown leaf edges, leaf curling, or yellowing, adjust your care routine accordingly. Remember, calatheas can be fussy, so it's normal for them to require some adjustment time in a new home.
```

**What happens when you ask about a plant that isn't in the database?**

```
How can I take care of my Basil plant?

Response: I couldn't find any specific information on a plant named 'Basil' in my database. However, I can provide you with some general guidance on how to care for a Basil plant. Basil is typically a low-maintenance herb that requires well-draining soil, plenty of sunlight, and regular watering. It prefers warmer temperatures, between 60-70°F (15-21°C), and high humidity. You can fertilize your Basil plant regularly to promote healthy growth. If you have any more specific questions or concerns about your Basil plant, I'll do my best to help.

```

**One thing about the tool call API that surprised you:**

```
The tool call API is not handling consecutive questions well, just throws "Error".

Fixed this issue with below change: 
for msg in history:
        if msg.get("content"):
            messages.append({"role": msg["role"], "content": msg["content"]})
```
