# Tools

Comprehensive examples for wiring up ADK tools in agent configurations. The
sections below cover knowledge bases, filesystem access, and subagents with
code that you can copy directly into your projects.

## Knowledge bases

Use the `Knowledge` class to create a searchable corpus that agents can query
via a `KnowledgeTool`.

```python
from autonomy import Agent, Knowledge, KnowledgeTool, Model, Node


async def main(node):
  # Create searchable knowledge base
  knowledge = Knowledge(
    name="legal_statutes",
    searchable=True,
    model=Model("embed-english-v3"),  # Embedding model
    max_results=5,  # Top 5 most relevant chunks
    max_distance=0.3  # Similarity threshold (0.0 = exact match, 1.0 = very different)
  )

  # Add legal documents
  await knowledge.add_text(
    document_name="contract_law",
    text="""
    Contract Formation Requirements:
    1. Offer: Clear proposal of terms
    2. Acceptance: Unequivocal agreement to terms
    3. Consideration: Exchange of value between parties
    4. Capacity: Legal ability to enter contract
    5. Legality: Lawful purpose and terms
    """
  )

  await knowledge.add_text(
    document_name="tort_law",
    text="""
    Elements of Negligence:
    - Duty of care owed to plaintiff
    - Breach of that duty
    - Causation (actual and proximate)
    - Damages suffered by plaintiff
    """
  )

  # Give agent access to the knowledge base as a tool
  await Agent.start(
    node=node,
    name="henry",
    instructions="""
    You are Henry, an expert legal assistant.
    Use search_statutes tool to find relevant legal information.
    """,
    model=Model("claude-sonnet-4-v1"),
    tools=[
      KnowledgeTool(knowledge=knowledge, name="search_statutes")
    ]
  )


Node.start(main)
```

### Adding documents

Add text directly to the knowledge base:

```python
await knowledge.add_text(
  document_name="statute_330",
  text="# 15 USC 330: Weather Modification\n\nAny person engaging in weather modification activities..."
)
```

Load documents from the web:

```python
await knowledge.add_document(
  document_name="sec_330",
  document_url="https://raw.githubusercontent.com/AlextheYounga/us-federal-code/refs/heads/master/usc/title-15-commerce-and-trade/chapter-9a-weather-modification-activities-or-attempts%3B-reporting-requirement/sec-330.md",
  content_type="text/markdown"  # Optional, auto-detected if omitted
)
```

Supported formats (via text extraction): plain text, Markdown, HTML, PDF (if
`markitdown` is installed), Word documents (if `markitdown` is installed), and
many more formats via `markitdown`.

### Configuration options

```python
knowledge = Knowledge(
  name="docs",                     # Unique name for this knowledge base
  searchable=True,                 # Enable vector search
  model=Model("embed-english-v3"), # Model for generating embeddings
  max_results=10,                  # Max search results to return
  max_distance=0.2,                # Max cosine distance (0.0-1.0, lower = more similar)
  max_knowledge_size=4096,         # Max characters to include in context
  chunker=NaiveChunker(            # How to split documents
    max_characters=256,            # Chunk size
    overlap=16                     # Overlap between chunks
  )
)
```

Key parameters:

- `model`: Embedding model for vector search. Default: `embed-english-v3`.
- `max_results`: Number of top results to return. Default: `10`.
- `max_distance`: Similarity threshold (0.0 = identical, 1.0 = completely
  different). Default: `0.2`.
- `max_knowledge_size`: Maximum characters in the returned context. Default:
  `4096`.
- `chunker`: Strategy for splitting documents into chunks.

### Storage backends

**In-memory storage (default)**

Stores everything in memory. Simple but not persistent:

```python
from autonomy import InMemory, Knowledge

knowledge = Knowledge(name="docs", searchable=True, storage=InMemory())
```

**Database storage**

For persistent storage across restarts:

```python
from autonomy import Database, Knowledge

# Initialize database connection
db = await Database.from_environment()
knowledge = Knowledge(name="docs", searchable=True, storage=db)
```

### Use knowledge as a tool

Give agents explicit control over when to search:

```python
from autonomy import KnowledgeTool

await Agent.start(
  node=node,
  name="henry",
  instructions="""
  You are Henry, an expert legal assistant.
  Use search_statutes when you need to reference legal code.
  """,
  model=Model("claude-sonnet-4-v1"),
  tools=[KnowledgeTool(knowledge=knowledge, name="search_statutes")]
)
```

## Filesystem access

Filesystem tools give agents a dedicated filesystem to save plans and take
notes, with configurable isolation across users and conversations.

### Quick start

Add filesystem tools to your agent with default (conversation-level) isolation:

```python
from autonomy import Agent, FilesystemTools, Model, Node


async def main(node):
  await Agent.start(
    node=node,
    name="assistant",
    instructions="You are a helpful assistant with file access",
    model=Model("claude-sonnet-4-v1"),
    tools=[FilesystemTools()]  # Default: conversation-level isolation
  )


Node.start(main)
```

### Visibility levels

Visibility controls file isolation between users and conversations. Choose the
level that matches your security and sharing requirements.

**Conversation (default)**

Maximum isolation — each conversation has its own isolated filesystem:

```python
tools=[FilesystemTools(visibility="conversation")]
```

Path structure: `/tmp/agent-files/{agent_name}/{scope}/{conversation}/`.

**Scope**

Per-user or per-tenant isolation — files are shared across conversations for
the same user:

```python
tools=[FilesystemTools(visibility="scope")]
```

Path structure: `/tmp/agent-files/{agent_name}/{scope}/`.

**Agent**

Shared across all users for a single agent:

```python
tools=[FilesystemTools(visibility="agent")]
```

Path structure: `/tmp/agent-files/{agent_name}/`.

**All**

No isolation — files are shared across all agents and users:

```python
tools=[FilesystemTools(visibility="all")]
```

Path structure: `/tmp/agent-files/`.

### Available operations

Filesystem tools provide these operations:

- `list_directory`
- `read_file`
- `write_file`
- `edit_file`
- `find_files`
- `search_in_files`
- `remove_file`
- `remove_directory`
- `copy_file`
- `move_file`

### Factory mode vs direct mode

**Factory mode (recommended)** automatically creates the right instance with
proper isolation:

```python
from autonomy import Agent, FilesystemTools, Model, Node


async def main(node):
  await Agent.start(
    node=node,
    name="assistant",
    instructions="You are a helpful assistant",
    model=Model("claude-sonnet-4-v1"),
    tools=[FilesystemTools(visibility="conversation")]
  )


Node.start(main)
```

**Direct mode** specifies the full context upfront:

```python
from autonomy import FilesystemTools, Tool

# Create instance with specific context
fs = FilesystemTools(
  visibility="scope",
  agent_name="assistant",
  scope="user-alice",
  base_dir="/data/files"
)

# Use directly
tools = [
  Tool(fs.read_file),
  Tool(fs.write_file),
  Tool(fs.list_directory)
]
```

### Configuration examples

Change the base directory:

```python
tools=[FilesystemTools(
  visibility="conversation",
  base_dir="/data/agent-files"  # Custom location
)]
```

Provide multiple filesystem instances with different scopes:

```python
from autonomy import FilesystemTools

# Shared templates (all users can read)
templates_fs = FilesystemTools(visibility="agent")

# User-specific workspace
workspace_fs = FilesystemTools(visibility="scope")

await Agent.start(
  node=node,
  name="assistant",
  instructions="""
  You access two filesystems:
  - Shared templates in the agent filesystem
  - Your personal workspace in the scope filesystem
  """,
  model=Model("claude-sonnet-4-v1"),
  tools=[templates_fs, workspace_fs]
)
```

Build a document assistant with user-specific workspaces:

```python
from autonomy import Agent, FilesystemTools, Model, Node


async def main(node):
  await Agent.start(
    node=node,
    name="document_assistant",
    instructions="""
    You are a document assistant that helps users manage their files.

    Capabilities:
    - List files in their workspace
    - Read and write documents
    - Search for content across files
    - Organize files into directories

    Each user has their own isolated workspace. Files are persistent
    across conversations for the same user.
    """,
    model=Model("claude-sonnet-4-v1"),
    tools=[FilesystemTools(visibility="scope")]
  )


Node.start(main)
```

Example deployment configuration:

```yaml
autonomy.yaml
name: docu
pods:
  - name: main-pod
    public: true
    containers:
      - name: main
        image: main
```

Usage examples:

```bash
# User Alice creates a document
curl --request POST \
  --header "Content-Type: application/json" \
  --data '{"message":"Create a file called project-notes.md with my project ideas", "scope":"user-alice"}' \
  "https://CLUSTER-ZONE.cluster.autonomy.computer/agents/document_assistant"

# Later, in a different conversation, Alice can still access it
curl --request POST \
  --header "Content-Type: application/json" \
  --data '{"message":"Read my project-notes.md file", "scope":"user-alice", "conversation":"new-chat"}' \
  "https://CLUSTER-ZONE.cluster.autonomy.computer/agents/document_assistant"

# User Bob has a completely separate workspace
curl --request POST \
  --header "Content-Type: application/json" \
  --data '{"message":"List my files", "scope":"user-bob"}' \
  "https://CLUSTER-ZONE.cluster.autonomy.computer/agents/document_assistant"
```

## Subagents

Subagents are specialized agents that handle delegated work from a parent
agent. Use them to parallelize complex tasks or separate responsibilities.

### Quick start

```python
from autonomy import Agent, Model, Node


async def main(node):
  await Agent.start(
    node=node,
    name="manager",
    instructions="""
    You are a project manager who coordinates research and writing tasks.

    When given a topic:
    1. Delegate research to the researcher subagent
    2. Delegate writing to the writer subagent
    3. Combine their work into a final response
    """,
    model=Model("claude-sonnet-4-v1"),
    subagents={
      "researcher": {
        "instructions": "You are a researcher who finds accurate information on topics.",
        "model": Model("claude-sonnet-4-v1"),
        "auto_start": True
      },
      "writer": {
        "instructions": "You are a writer who creates clear, engaging content.",
        "model": Model("nova-pro-v1"),
        "auto_start": True
      }
    }
  )


Node.start(main)
```

### Delegation tools

When you configure subagents, the parent agent automatically receives tools
for lifecycle and delegation:

- `start_subagent`
- `delegate_to_subagent`
- `delegate_to_subagents_parallel`
- `list_subagents`
- `stop_subagent`

### Delegation patterns

**Sequential delegation** handles steps one at a time:

```python
await Agent.start(
  node=node,
  name="sequential_manager",
  instructions="""
  When given a topic to analyze:
  1. First, delegate to 'researcher' to gather information
  2. Then, delegate to 'analyst' to analyze the research
  3. Finally, delegate to 'writer' to create a report

  Use delegate_to_subagent for each step. Wait for results before proceeding.
  """,
  model=Model("claude-sonnet-4-v1"),
  subagents={
    "researcher": {
      "instructions": "Research topics thoroughly and provide factual information",
      "auto_start": True
    },
    "analyst": {
      "instructions": "Analyze research data and identify key insights",
      "auto_start": True
    },
    "writer": {
      "instructions": "Write clear, engaging reports from analysis",
      "auto_start": True
    }
  }
)
```

**Parallel delegation** runs independent tasks concurrently:

```python
await Agent.start(
  node=node,
  name="parallel_manager",
  instructions="""
  When given a list of topics to research:
  1. Use delegate_to_subagents_parallel to research all topics at once
  2. Each topic will be processed by a separate researcher instance
  3. Combine the results into a comprehensive summary
  """,
  model=Model("claude-sonnet-4-v1"),
  subagents={
    "researcher": {
      "instructions": "Research a single topic and provide detailed findings",
      "model": Model("claude-sonnet-4-v1"),
      "auto_start": True
    }
  }
)
```

**Role-based delegation** assigns specialized roles with tailored tools:

```python
await Agent.start(
  node=node,
  name="content_manager",
  instructions="""
  You coordinate content creation:
  - Use 'researcher' for fact-finding
  - Use 'writer' for drafting content
  - Use 'editor' for reviewing and improving content
  - Use 'fact_checker' to verify claims

  Choose the right subagent based on the task at hand.
  """,
  model=Model("claude-sonnet-4-v1"),
  subagents={
    "researcher": {
      "instructions": "Research topics and gather accurate information",
      "tools": [McpTool("brave_search", "brave_web_search")],
      "auto_start": True
    },
    "writer": {
      "instructions": "Write engaging, clear content",
      "model": Model("claude-sonnet-4-v1"),
      "auto_start": True
    },
    "editor": {
      "instructions": "Review content for clarity, grammar, and style",
      "model": Model("claude-sonnet-4-v1"),
      "auto_start": True
    },
    "fact_checker": {
      "instructions": "Verify factual claims and identify unsupported statements",
      "tools": [McpTool("brave_search", "brave_web_search")],
      "auto_start": True
    }
  }
)
```

### Configuration options

Subagents support several options beyond `instructions` and `model`:

- `auto_start`: Start automatically on first use.
- `tools`: Tools available to the subagent.
- `max_iterations`: Maximum reasoning loops (default `10`).
- `max_execution_time`: Timeout in seconds (default `60`).
- `enable_long_term_memory`: Enable persistent memory.
- `runner_filter`: Direct subagents to specific runner nodes.

### Memory behavior

Each subagent instance maintains its own conversation history. Parallel
instances do not share history with each other or with the parent agent.

### Best practices

- Choose models appropriate to each role.
- Set timeouts that fit the task (for example, longer for web search).
- Prefer `auto_start` for simpler orchestration.
- Provide clear, focused instructions for each role.
- Include failure-handling guidance in the parent agent instructions.
