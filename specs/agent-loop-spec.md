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
Structure: a bounded loop that runs at most MAX_TOOL_ROUNDS (5) iterations,
using a counter so the tool-calling can never spin forever.

    for _ in range(MAX_TOOL_ROUNDS):
        response = client.chat.completions.create(... messages, tools, "auto")
        assistant_message = response.choices[0].message

        # Condition (a): no tool calls -> LLM has a final answer
        if not assistant_message.tool_calls:
            return assistant_message.content

        # Otherwise: append assistant msg, run each tool, append results, loop again
        messages.append(assistant_message)
        for tool_call in assistant_message.tool_calls:
            ... dispatch_tool() ... append {"role": "tool", ...}

    # Condition (b): fell out of the loop -> hit MAX_TOOL_ROUNDS still wanting tools

Detecting each condition:
  (a) No tool calls: after each LLM call, check `assistant_message.tool_calls`.
      If it's falsy (None/empty), the model is done deciding and has produced a
      text answer. Return `assistant_message.content` immediately.
  (b) Round limit reached: the `for _ in range(MAX_TOOL_ROUNDS)` exhausts without
      ever hitting the `return` in (a). Detected by reaching the code *after* the loop body.

What to return in each case:
  (a) The assistant's text content for that turn.
  (b) A user-readable message like "I wasn't able to finish looking that up — could you rephrase or narrow your question?"
```

---

### Extracting the final text response

*Once the loop exits because there are no more tool calls, how do you extract the text content from the response object? What field holds the string you should return?*

```
The string lives at: response.choices[0].message.content

Access path, field by field:
  - response.choices        -> a list of completion choices; we always use [0]
  - .choices[0].message     -> the assistant message object (same one whose
                               .tool_calls we checked to decide whether to loop)
  - .message.content        -> the plain-text answer string to return

On the no-tool-calls path, .tool_calls is None and .content holds the final text, so:

    if not assistant_message.tool_calls:
        return assistant_message.content

Note: .content can be None when the message instead carries tool_calls (the
model chose to call a tool rather than speak). That case never reaches this
return, because we only get here once .tool_calls is falsy. As a defensive
guard against an empty/None content (per the "never empty" contract), fall back:

    return assistant_message.content or "Sorry, I couldn't generate a response."
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Trace of a working agent turn (what tools were called and in what order):**

```
Query: "How should I care for my calathea?"
Round 1 tool call: [lookup_plant, {'plant_name': 'calathea'}]
Round 2 tool call: [get_seasonal_conditions, {}] (if any)
Final response: The agent references the care data for calatheas including watering instructions, light conditions, humidity, and fertilizations with some specific instructions for care in the current season.
```

**What happens when you ask about a plant that isn't in the database?**

```
The agent will acknowledge that the plant isn't in the database and then try to offer general advice. This behavior follows graceful degradation and is enforced by the not-found message in lookup_plant() given back to the LLM.
```

**One thing about the tool call API that surprised you:**

```
When the model calls a tool with no args, the tool_call.function.arguments string comes back as "null"/"" rather than "{}", so you can't assume json.loads(arguments) returns a dict.
```
