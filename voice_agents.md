# Voice Agents

Give agents the ability to listen and speak with low-latency voice
interactions backed by a dedicated voice interface agent and a primary agent
that holds the full toolset.

## Architecture

- **Voice interface agent**: Uses a low-latency, real-time audio model to handle
  greetings, chitchat, and quick clarifications directly.
- **Primary agent**: Runs the full agent stack (tools, knowledge, actions) and
  handles complex requests delegated from the voice agent.
- **Delegation flow**: For complex requests, the voice agent uses a filler
  phrase such as “Let me check on that,” delegates to the primary agent, and
  then speaks the primary agent’s response verbatim.

## Quick start

```python
from autonomy import Agent, Model, Node


async def main(node):
  await Agent.start(
    node=node,
    name="assistant",
    instructions="You are a helpful customer service agent.",
    model=Model("claude-sonnet-4-v1"),
    voice={"voice": "nova"},
  )


Node.start(main)
```

Connect to the voice agent via WebSocket for audio or use the HTTP API for text
interactions.

## Voice configuration

The `voice` parameter accepts these options:

| Option | Description | Default |
| --- | --- | --- |
| `realtime_model` | Model for the voice agent (must support realtime API). | `gpt-4o-realtime-preview` |
| `voice` | TTS voice ID (`alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`). | `echo` |
| `instructions` | Custom voice agent instructions; auto-generated if unset. | `None` |
| `input_audio_format` | Audio format for input (`pcm16`, `g711_ulaw`, `g711_alaw`). | `pcm16` |
| `output_audio_format` | Audio format for output (`pcm16`, `g711_ulaw`, `g711_alaw`). | `pcm16` |
| `vad_threshold` | Voice activity detection sensitivity (0.0–1.0). | `0.5` |
| `vad_prefix_padding_ms` | Audio to include before speech detection. | `300` |
| `vad_silence_duration_ms` | Silence duration that marks end of speech. | `500` |

## Default behaviors

- **Handled directly**: Greetings, chitchat, collecting information, and quick
  clarifications.
- **Delegation filler phrases** (before sending a complex request to the
  primary agent): “Just a second.”, “Let me check.”, “One moment.”, “Let me
  look into that.”, “Give me a moment.”, “Let me see.”

## Tune VAD for responsiveness

```python
from autonomy import Agent, Model, Node


async def main(node):
  await Agent.start(
    node=node,
    name="assistant",
    instructions="You are a helpful assistant.",
    model=Model("claude-sonnet-4-v1"),
    voice={
      "voice": "alloy",
      "vad_threshold": 0.3,  # More sensitive detection
      "vad_silence_duration_ms": 700,  # Wait longer before ending speech
    },
  )


Node.start(main)
```

## Voice agents with tools

Voice agents delegate to the primary agent, which can call tools normally.

```python
from autonomy import Agent, Model, Node, Tool


async def lookup_order(order_id: str) -> dict:
  """Look up an order by ID."""
  return {"order_id": order_id, "status": "shipped", "eta": "Tomorrow"}


async def main(node):
  await Agent.start(
    node=node,
    name="support",
    instructions="""You are a customer support agent.
    Use the lookup_order tool to find order information.""",
    model=Model("claude-sonnet-4-v1"),
    tools=[Tool(lookup_order)],
    voice={"voice": "nova"},
  )


Node.start(main)
```

Flow for “Where is my order 12345?”:

1. Voice agent: “Let me look up your order.”
2. Voice agent delegates to the primary agent.
3. Primary agent calls `lookup_order("12345")`.
4. Voice agent speaks the primary agent’s response verbatim.

## Voice agents with knowledge

Combine voice with knowledge search for intelligent Q&A:

```python
from autonomy import Agent, Knowledge, KnowledgeTool, Model, NaiveChunker, Node


async def main(node):
  knowledge = Knowledge(
    name="product_docs",
    searchable=True,
    model=Model("embed-english-v3"),
    max_results=5,
    chunker=NaiveChunker(max_characters=1024),
  )

  await knowledge.add_document(
    document_name="user-guide",
    document_url="https://example.com/docs/user-guide.md",
    content_type="text/markdown",
  )

  await Agent.start(
    node=node,
    name="docs",
    instructions="""You are a product expert.
    Search the knowledge base to answer questions accurately.""",
    model=Model("claude-sonnet-4-v1"),
    tools=[KnowledgeTool(knowledge=knowledge, name="search_docs")],
    voice={"voice": "shimmer"},
  )


Node.start(main)
```

## Memory isolation

Voice sessions use the same memory isolation as text conversations. Include
`scope` and `conversation` query parameters when connecting over WebSocket:

```
ws://.../agents/assistant/voice?scope=user-123&conversation=session-456
```

## Complete example: software engineering interviewer

```python
from autonomy import Agent, Model, Node


INSTRUCTIONS = """
You are an experienced software engineering interviewer conducting first-round
screening interviews. Your goal is to assess candidates on technical
fundamentals, problem-solving ability, and communication skills.

Interview structure:
1. Brief introduction and put the candidate at ease
2. Ask about their background and experience (2-3 minutes)
3. Technical questions appropriate to their level (10-15 minutes)
4. Behavioral questions about teamwork and challenges (5 minutes)
5. Answer any questions they have about the role

Guidelines:
- Be warm and professional to help candidates perform their best
- Ask follow-up questions to understand their thought process
- Probe deeper if answers are surface-level
- Give hints if they're stuck, but note that you did
- Keep responses concise since this is a voice conversation
- Adapt difficulty based on their stated experience level

Technical topics to cover:
- Data structures and algorithms fundamentals
- System design basics (for senior candidates)
- Language-specific questions based on their background
- Problem-solving approach and debugging strategies

After the interview, provide a brief summary of strengths and areas for
improvement.
"""


async def main(node: Node):
  await Agent.start(
    node=node,
    name="interviewer",
    instructions=INSTRUCTIONS,
    model=Model("claude-sonnet-4-v1"),
    voice={"voice": "alloy"},
  )


Node.start(main)
```

Connect via WebSocket for voice:

```
ws://${CLUSTER}-${ZONE}.cluster.autonomy.computer/agents/interviewer/voice
```

Or use HTTP for text:

```bash
curl --request POST \
  --header "Content-Type: application/json" \
  --data '{"message": "Hi, I am ready to start the interview."}' \
  "https://${CLUSTER}-${ZONE}.cluster.autonomy.computer/agents/interviewer"
```
