# Part 8: GenAI System Design

## Table of Contents

1. [Introduction to GenAI Systems](#introduction)
2. [API-based LLM Systems](#api-based-systems)
3. [Tool Calling & Function Calling](#tool-calling)
4. [AI Agents](#ai-agents)
5. [Memory Architectures](#memory-architectures)
6. [Multi-Agent Systems](#multi-agent-systems)
7. [Frameworks & Orchestration](#frameworks)
8. [System Design Patterns](#design-patterns)
9. [Production Considerations](#production-considerations)

---

## Introduction to GenAI Systems {#introduction}

### What are GenAI Systems?

**GenAI System** = Applications built using Large Language Models + tools + data + orchestration.

**Evolution:**

```
2020: ML Systems
  └─ Train model → Deploy API → Done

2023-2025: GenAI Systems
  └─ LLM + Tools + Memory + Agents + Retrieval + Evaluation → Complex orchestration
```

**Key difference:**

- ML systems: Train once, serve predictions
- GenAI systems: Prompt engineering, tool use, multi-step reasoning, continuous improvement

### System Complexity Levels

**Level 1: Simple prompt**

```
User → LLM → Response
```

**Level 2: RAG**

```
User → Query → Vector DB → Context → LLM → Response
```

**Level 3: Single agent with tools**

```
User → Agent (LLM) → Tools (APIs, search, calculator) → LLM synthesis → Response
```

**Level 4: Multi-agent system**

```
User → Orchestrator → Agent 1 (research) + Agent 2 (writing) + Agent 3 (review) → Response
```

**Most production systems: Level 2-3**

---

## API-based LLM Systems {#api-based-systems}

### Basic API Call

**Simplest GenAI system:**

```python
from openai import OpenAI

client = OpenAI(api_key="...")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is the capital of France?"}
    ],
    temperature=0.7,
    max_tokens=150,
)

print(response.choices[0].message.content)
```

**Parameters explained:**

```
Parameter      | Default | Description                           | When to adjust
---------------|---------|---------------------------------------|------------------
temperature    | 1.0     | Randomness (0=deterministic, 2=creative) | 0 for factual, 1+ for creative
top_p          | 1.0     | Nucleus sampling (alternative to temp) | 0.9 for focused responses
max_tokens     | inf     | Max response length                   | Always set (cost control)
frequency_pen  | 0       | Penalize repeated tokens              | >0 to reduce repetition
presence_pen   | 0       | Penalize used tokens                  | >0 to encourage diversity
stop           | None    | Stop sequences                        | Custom formats
```

### Streaming Responses

**For better UX (show tokens as they arrive):**

```python
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Write a story."}],
    stream=True,
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

**Benefits:**

- User sees progress immediately
- Can cancel long responses
- Better perceived latency

### Error Handling & Retries

**Production-grade API call:**

```python
import openai
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
    reraise=True,
)
def call_llm(messages, model="gpt-4o"):
    try:
        response = client.chat.completions.create(
            model=model,
            messages=messages,
            timeout=30,  # 30-second timeout
        )
        return response.choices[0].message.content

    except openai.RateLimitError as e:
        print(f"Rate limit hit: {e}")
        raise  # Retry

    except openai.APIConnectionError as e:
        print(f"Connection error: {e}")
        raise  # Retry

    except openai.APIError as e:
        print(f"API error: {e}")
        if e.status_code >= 500:
            raise  # Retry server errors
        else:
            return f"Error: {e}"  # Don't retry client errors
```

**Error types:**

- **RateLimitError:** Too many requests → Retry with backoff
- **APIConnectionError:** Network issue → Retry
- **APIError (5xx):** Server issue → Retry
- **APIError (4xx):** Bad request → Don't retry, log error
- **Timeout:** Request too slow → Retry with longer timeout

---

## Tool Calling & Function Calling {#tool-calling}

### What is Tool Calling?

**Tool calling** = LLM decides when and how to call external functions/APIs.

**Flow:**

```
1. User: "What's the weather in Paris?"
2. LLM recognizes it needs weather data
3. LLM calls: get_weather(location="Paris")
4. System executes function → Returns: {"temp": 18, "condition": "Sunny"}
5. LLM synthesizes: "It's 18°C and sunny in Paris."
```

**Key: LLM doesn't execute functions, it generates function calls.**

### Function Schema Definition

**Define tools for LLM:**

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather for a location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name, e.g., 'Paris' or 'New York'",
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit",
                    },
                },
                "required": ["location"],
            },
        },
    }
]
```

**Schema follows JSON Schema spec.**

### Tool Calling Example

**Full implementation:**

```python
import json
from openai import OpenAI

client = OpenAI()

def get_weather(location, unit="celsius"):
    """Simulated weather API"""
    # In reality: call weather API
    return {"location": location, "temperature": 18, "condition": "Sunny", "unit": unit}

def call_with_tools(messages, tools):
    # Call LLM with tools
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools,
        tool_choice="auto",  # Let LLM decide
    )

    message = response.choices[0].message

    # Check if LLM wants to call a tool
    if message.tool_calls:
        # Execute each tool call
        for tool_call in message.tool_calls:
            function_name = tool_call.function.name
            arguments = json.loads(tool_call.function.arguments)

            # Execute function
            if function_name == "get_weather":
                result = get_weather(**arguments)

            # Add function result to messages
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result),
            })

        # Call LLM again with function results
        return call_with_tools(messages, tools)

    else:
        # No tool calls, return final response
        return message.content

# Usage
messages = [
    {"role": "user", "content": "What's the weather in Paris?"}
]

result = call_with_tools(messages, tools)
print(result)
# Output: "The current weather in Paris is 18°C and sunny."
```

### Parallel Tool Calling

**LLMs can call multiple tools in parallel:**

```python
# User: "What's the weather in Paris and London?"

# LLM returns:
tool_calls = [
    {"function": {"name": "get_weather", "arguments": '{"location": "Paris"}'}},
    {"function": {"name": "get_weather", "arguments": '{"location": "London"}'}},
]

# Execute in parallel
import asyncio

async def execute_tools_parallel(tool_calls):
    tasks = [get_weather_async(**json.loads(tc.function.arguments)) for tc in tool_calls]
    results = await asyncio.gather(*tasks)
    return results
```

**Reduces latency from 2× serial calls to 1× parallel.**

### Tool Calling Patterns

**Pattern 1: Sequential (Chain)**

```
User query → Tool 1 → Result 1 → Tool 2 → Result 2 → Final answer
```

**Example:**

1. Search for company
2. Get company details
3. Calculate metrics
4. Format response

**Pattern 2: Parallel**

```
User query → [Tool 1, Tool 2, Tool 3] → [Results] → Synthesize → Final answer
```

**Example:**

1. Get weather in 5 cities simultaneously
2. Combine results

**Pattern 3: Conditional**

```
User query → Tool 1 → Result 1 → If X, call Tool 2; else call Tool 3
```

**Example:**

1. Check if user is authenticated
2. If yes, access database; else, return error

**Pattern 4: Iterative (Loop)**

```
User query → Tool 1 → Result → If not sufficient, call Tool 1 again (different params)
```

**Example:**

1. Search knowledge base
2. If no results, expand search query
3. Repeat up to 3 times

---

## AI Agents {#ai-agents}

### What is an AI Agent?

**AI Agent** = LLM that can:

- Use tools
- Make decisions
- Take actions
- Iterate until goal achieved

**Agent vs LLM:**

```
LLM: One-shot
  User → Prompt → Response (done)

Agent: Multi-step
  User → Goal → [Reasoning → Action → Observation] × N → Final answer
```

### ReAct Pattern (Reasoning + Acting)

**Most common agent architecture.**

**Loop:**

```
1. Thought: Reason about what to do
2. Action: Choose and execute tool
3. Observation: See result
4. Repeat until done
```

**Example:**

```
User: "What's the current stock price of Apple and how much has it changed today?"

Iteration 1:
  Thought: I need to get the current stock price of Apple.
  Action: get_stock_price(symbol="AAPL")
  Observation: {"price": 178.50, "change": +2.30, "change_percent": +1.3}

Iteration 2:
  Thought: I have the information needed.
  Action: None
  Final Answer: Apple (AAPL) is currently trading at $178.50, up $2.30 (+1.3%) today.
```

**Implementation:**

```python
def react_agent(question, tools, max_iterations=5):
    messages = [
        {"role": "system", "content": REACT_SYSTEM_PROMPT},
        {"role": "user", "content": question}
    ]

    for i in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=tools,
        )

        message = response.choices[0].message
        messages.append(message)

        # Check for tool calls
        if message.tool_calls:
            for tool_call in message.tool_calls:
                result = execute_tool(tool_call)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": json.dumps(result)
                })
        else:
            # No more tools, return answer
            return message.content

    return "Max iterations reached."

REACT_SYSTEM_PROMPT = """
You are a helpful assistant with access to tools.
For each step:
1. Think about what you need to do
2. Use a tool if needed
3. Synthesize the final answer when you have enough information
"""
```

### Planning Agents

**Concept:** Plan all steps upfront, then execute.

**vs ReAct:**

- ReAct: Plan one step at a time
- Planning: Plan all steps, then execute

**Example:**

```
User: "Book a flight to Paris and hotel for 3 nights."

Planning Agent:
  Plan:
    1. Search flights to Paris
    2. Select cheapest flight
    3. Search hotels near city center
    4. Select hotel with good reviews
    5. Book flight
    6. Book hotel

  Execute:
    Step 1: search_flights(destination="Paris") → ...
    Step 2: select_flight(id=12345) → ...
    (continue)
```

**Implementation:**

```python
def planning_agent(question, tools):
    # Step 1: Generate plan
    plan_prompt = f"Break down this task into steps: {question}"
    plan = llm_call(plan_prompt)

    # Step 2: Execute plan
    results = []
    for step in plan.steps:
        result = execute_step(step, tools)
        results.append(result)

    # Step 3: Synthesize
    synthesis = llm_call(f"Summarize results: {results}")
    return synthesis
```

**Pros:**

- More structured
- Easier to debug
- Can show plan to user

**Cons:**

- Inflexible (can't adapt mid-execution)
- Expensive (separate LLM call for planning)

### Reflection / Self-Correction Agents

**Concept:** Agent critiques its own output and improves.

**Loop:**

```
1. Generate initial response
2. Critique response (separate LLM call)
3. If good → Return
4. If bad → Regenerate with critique as feedback
5. Repeat up to N times
```

**Example:**

```
User: "Write a Python function to reverse a string."

Iteration 1:
  Output:
    def reverse(s):
        return s[::-1]

  Critique: "Good, but no docstring or type hints."

Iteration 2:
  Output:
    def reverse(s: str) -> str:
        """Reverses a string."""
        return s[::-1]

  Critique: "Perfect."

  Return: [Final output]
```

**Implementation:**

```python
def reflection_agent(task, max_iterations=3):
    output = None

    for i in range(max_iterations):
        # Generate
        if i == 0:
            output = llm_call(f"Task: {task}")
        else:
            output = llm_call(f"Task: {task}\nPrevious attempt: {output}\nCritique: {critique}\nImprove it.")

        # Critique
        critique = llm_call(f"Critique this output:\n{output}\nIs it correct and high-quality?")

        # Check if good enough
        if "perfect" in critique.lower() or "correct" in critique.lower():
            return output

    return output  # Return best attempt
```

**Use cases:**

- Code generation (check for bugs)
- Writing (improve clarity)
- Math problems (verify solution)

### Agent Comparison

```
Agent Type       | Complexity | Cost   | Accuracy | Latency | Use Case
-----------------|------------|--------|----------|---------|------------------
ReAct            | Medium     | Medium | Good     | Medium  | General tasks
Planning         | High       | High   | Great    | High    | Complex workflows
Reflection       | Medium     | High   | Great    | High    | Quality-critical
Simple prompt    | Low        | Low    | Okay     | Low     | Simple queries
```

**Rule of thumb:**

- Start simple (prompt engineering)
- Add tools (ReAct agent)
- Add planning if complex
- Add reflection if quality-critical

---

## Tool Calling & Function Calling {#tool-calling}

### Available Tools Types

**1. Search (Google, Bing, DuckDuckGo)**

```python
def search_web(query: str, num_results: int = 5) -> list:
    """Search the web and return top results."""
    # Use SerpAPI, Bing API, or scraping
    return [
        {"title": "...", "snippet": "...", "url": "..."},
        ...
    ]
```

**2. Calculator (Python REPL)**

```python
def calculate(expression: str) -> float:
    """Evaluate a math expression."""
    try:
        result = eval(expression)  # Unsafe! Use ast.literal_eval in production
        return result
    except Exception as e:
        return f"Error: {e}"
```

**3. Database Query**

```python
def query_database(sql: str) -> list:
    """Execute SQL query and return results."""
    # Sanitize SQL to prevent injection!
    conn = sqlite3.connect("db.sqlite")
    cursor = conn.execute(sql)
    return cursor.fetchall()
```

**4. API Calls**

```python
def get_user_info(user_id: int) -> dict:
    """Get user information from internal API."""
    response = requests.get(f"https://api.internal.com/users/{user_id}")
    return response.json()
```

**5. File Operations**

```python
def read_file(filepath: str) -> str:
    """Read file contents."""
    with open(filepath, 'r') as f:
        return f.read()

def write_file(filepath: str, content: str):
    """Write content to file."""
    with open(filepath, 'w') as f:
        f.write(content)
```

### Tool Safety

**Critical: Validate all tool inputs!**

**Example vulnerabilities:**

```python
# BAD: SQL injection vulnerability
def query_database(sql: str):
    return conn.execute(sql)  # User can inject: "DROP TABLE users"

# GOOD: Parameterized queries
def query_database(table: str, column: str, value: str):
    query = "SELECT * FROM ? WHERE ? = ?"
    return conn.execute(query, (table, column, value))
```

**Security checklist:**

- ✅ Validate inputs (type, range, whitelist)
- ✅ Sanitize SQL, shell commands
- ✅ Rate limit tool calls
- ✅ Require confirmation for destructive actions
- ✅ Log all tool executions
- ✅ Sandbox code execution

### Structured Output for Tools

**Force LLM to output valid JSON:**

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Extract: Name, Age, City from: 'John Doe is 25 and lives in Paris.'"}],
    response_format={"type": "json_object"},  # Force JSON
)

output = json.loads(response.choices[0].message.content)
# {"name": "John Doe", "age": 25, "city": "Paris"}
```

**Alternative: Use Pydantic for validation:**

```python
from pydantic import BaseModel
from openai import OpenAI

class Person(BaseModel):
    name: str
    age: int
    city: str

client = OpenAI()

completion = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Extract from: 'John Doe is 25 and lives in Paris.'"}],
    response_format=Person,
)

person = completion.choices[0].message.parsed
print(person.name)  # "John Doe"
print(person.age)   # 25
```

---

## Memory Architectures {#memory-architectures}

### Why Memory?

**Problem:** LLMs are stateless.

```
Conversation 1:
  User: "My name is Alice."
  LLM: "Nice to meet you, Alice."

Conversation 2 (new session):
  User: "What's my name?"
  LLM: "I don't know your name." ❌
```

**Solution:** Store conversation history and facts.

### Types of Memory

**1. Short-term Memory (Conversation buffer)**

**Store recent messages:**

```python
class ConversationMemory:
    def __init__(self, max_messages=10):
        self.messages = []
        self.max_messages = max_messages

    def add_message(self, role, content):
        self.messages.append({"role": role, "content": content})

        # Keep only recent messages
        if len(self.messages) > self.max_messages:
            self.messages = self.messages[-self.max_messages:]

    def get_messages(self):
        return self.messages

# Usage
memory = ConversationMemory()
memory.add_message("user", "My name is Alice.")
memory.add_message("assistant", "Nice to meet you, Alice.")
memory.add_message("user", "What's my name?")

# Pass full history to LLM
response = client.chat.completions.create(
    model="gpt-4o",
    messages=memory.get_messages()  # Includes context
)
```

**Problem:** Context length limit (128K tokens).

**2. Long-term Memory (Vector DB)**

**Store facts/entities permanently:**

```python
from chromadb import Client

class LongTermMemory:
    def __init__(self):
        self.db = Client()
        self.collection = self.db.create_collection("user_facts")

    def store_fact(self, fact: str):
        """Store fact in vector DB"""
        self.collection.add(
            documents=[fact],
            ids=[str(uuid.uuid4())]
        )

    def recall(self, query: str, n=3):
        """Retrieve relevant facts"""
        results = self.collection.query(
            query_texts=[query],
            n_results=n
        )
        return results['documents'][0]

# Usage
memory = LongTermMemory()
memory.store_fact("User's name is Alice")
memory.store_fact("User lives in Paris")

# Later conversation
relevant_facts = memory.recall("What's my name?")
# Returns: ["User's name is Alice"]

# Add to context
context = "\n".join(relevant_facts)
messages = [
    {"role": "system", "content": f"Known facts:\n{context}"},
    {"role": "user", "content": "What's my name?"}
]
```

**3. Working Memory (Scratchpad)**

**Agent's internal notes during reasoning:**

```
Thought: I need to find flights to Paris.
Working Memory: {"destination": "Paris", "task": "find_flights"}

Thought: I found 3 flights, need to pick the cheapest.
Working Memory: {"flights": [...], "criteria": "cheapest"}

Thought: Flight #2 is $450, book it.
Working Memory: {"selected_flight": 2, "price": 450}
```

**Implementation:**

```python
class WorkingMemory:
    def __init__(self):
        self.data = {}

    def store(self, key, value):
        self.data[key] = value

    def retrieve(self, key):
        return self.data.get(key)

    def clear(self):
        self.data = {}
```

### Memory Compression

**Problem:** Long conversations exceed context limit.

**Solution: Summarize old messages**

```python
def compress_memory(messages, max_tokens=4000):
    current_tokens = count_tokens(messages)

    if current_tokens < max_tokens:
        return messages

    # Summarize old messages
    old_messages = messages[:-10]  # Keep last 10
    summary_prompt = f"Summarize this conversation:\n{old_messages}"
    summary = llm_call(summary_prompt)

    # Replace with summary
    return [
        {"role": "system", "content": f"Previous conversation summary:\n{summary}"},
        *messages[-10:]  # Recent messages verbatim
    ]
```

### Memory Patterns Comparison

```
Memory Type      | Storage  | Retrieval | Use Case
-----------------|----------|-----------|------------------
Short-term       | List     | Recent N  | Conversation flow
Long-term        | Vector DB| Semantic  | User facts, knowledge
Working          | Dict     | Key       | Agent reasoning
Compressed       | Summary  | Summary   | Long conversations
```

---

## Multi-Agent Systems {#multi-agent-systems}

### Why Multi-Agent?

**Single agent limitations:**

- Jack of all trades, master of none
- Context length constraints
- Hard to specialize

**Multi-agent benefits:**

- Specialized agents (research, writing, coding, review)
- Parallel execution
- Modular (replace one agent without affecting others)

### Multi-Agent Patterns

**Pattern 1: Coordinator-Worker**

```
User → Coordinator (routes task) → Worker 1, Worker 2, Worker 3 → Coordinator (synthesize) → User
```

**Example: Content creation**

```
User: "Write a blog post about AI agents."

Coordinator:
  └─ Assigns:
      ├─ Research Agent: Gather info about AI agents
      ├─ Writing Agent: Write draft
      └─ Review Agent: Check quality

Flow:
  1. Research Agent → Returns: [Facts about agents]
  2. Writing Agent (uses research) → Returns: [Draft blog post]
  3. Review Agent → Returns: [Feedback: "Add examples"]
  4. Writing Agent (incorporates feedback) → Returns: [Final post]
  5. Coordinator → Returns to user
```

**Implementation:**

```python
class CoordinatorAgent:
    def __init__(self):
        self.research_agent = ResearchAgent()
        self.writing_agent = WritingAgent()
        self.review_agent = ReviewAgent()

    def run(self, task):
        # Decompose task
        plan = self.decompose(task)

        # Assign to workers
        research = self.research_agent.run(plan['research_query'])
        draft = self.writing_agent.run(plan['writing_task'], context=research)
        review = self.review_agent.run(draft)

        # Incorporate feedback
        if review['needs_improvement']:
            final = self.writing_agent.run(plan['writing_task'], context=research, feedback=review)
        else:
            final = draft

        return final
```

**Pattern 2: Debate / Consensus**

```
User → Agent 1 (propose) → Agent 2 (critique) → Agent 3 (judge) → Best answer
```

**Example: Code review**

```
Agent 1 (Generator): Writes code
Agent 2 (Critic): Reviews code, suggests improvements
Agent 1 (Generator): Revises code based on feedback
Agent 3 (Judge): Determines if code is good enough
```

**Use case:** Higher quality through multiple perspectives.

**Pattern 3: Specialized Agents (Pipeline)**

```
User → Agent A → Agent B → Agent C → Output
```

**Example: Data pipeline**

```
User uploads CSV →
  Agent 1 (Analyzer): Analyzes data, suggests transformations →
  Agent 2 (Transformer): Applies transformations →
  Agent 3 (Visualizer): Creates charts →
  User gets report
```

**Pattern 4: Hierarchical Agents**

```
Manager Agent
  ├─ Sub-agent 1 (handles subtask 1)
  ├─ Sub-agent 2 (handles subtask 2)
  └─ Sub-agent 3 (handles subtask 3)
```

**Example: Travel booking**

```
Travel Manager Agent
  ├─ Flight Agent (searches and books flights)
  ├─ Hotel Agent (searches and books hotels)
  └─ Activity Agent (suggests tourist activities)
```

### Multi-Agent Example: AutoGPT Style

**AutoGPT = Autonomous agent with memory and tools.**

**Architecture:**

```
User goal: "Research and write a report on AI trends."

Loop:
  1. Agent decides next action
  2. Execute action (search, write file, read file, etc.)
  3. Update memory
  4. Repeat until goal achieved
```

**Simplified implementation:**

```python
class AutoAgent:
    def __init__(self, goal):
        self.goal = goal
        self.memory = []
        self.completed = False

    def run(self, max_steps=10):
        for step in range(max_steps):
            if self.completed:
                break

            # Decide next action
            action = self.decide_action()

            # Execute
            result = self.execute(action)

            # Update memory
            self.memory.append({"action": action, "result": result})

            # Check if goal achieved
            self.completed = self.check_goal()

        return self.memory

    def decide_action(self):
        context = f"Goal: {self.goal}\nMemory: {self.memory}\nWhat's the next action?"
        response = llm_call(context, tools=AVAILABLE_TOOLS)
        return response
```

**Challenges:**

- Expensive (many LLM calls)
- Can get stuck in loops
- Hard to control

---

## Frameworks & Orchestration {#frameworks}

### LangChain

**Most popular GenAI framework.**

**Core concepts:**

- **Chains:** Sequence of LLM calls
- **Agents:** LLM with tools
- **Memory:** Conversation history
- **Retrieval:** Integration with vector DBs

**Simple chain:**

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser

llm = ChatOpenAI(model="gpt-4o")
prompt = ChatPromptTemplate.from_template("Tell me a joke about {topic}")

chain = prompt | llm | StrOutputParser()

result = chain.invoke({"topic": "AI"})
print(result)
```

**Agent with tools:**

```python
from langchain.agents import create_openai_functions_agent, AgentExecutor
from langchain.tools import Tool

def search(query: str) -> str:
    return f"Search results for: {query}"

tools = [
    Tool(name="Search", func=search, description="Search the web")
]

agent = create_openai_functions_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools)

result = agent_executor.invoke({"input": "What's the weather in Paris?"})
```

**Pros:**

- Rich ecosystem (100+ integrations)
- Easy to prototype

**Cons:**

- Abstractions can be leaky
- Complex for simple use cases
- Performance overhead

### LlamaIndex

**Focused on data ingestion and retrieval.**

**Core concepts:**

- **Loaders:** Ingest data (PDF, web, databases)
- **Indexes:** Structure data (vector, tree, list)
- **Query engines:** Retrieve and synthesize

**Simple RAG:**

```python
from llama_index import VectorStoreIndex, SimpleDirectoryReader

# Load documents
documents = SimpleDirectoryReader("./data").load_data()

# Create index
index = VectorStoreIndex.from_documents(documents)

# Query
query_engine = index.as_query_engine()
response = query_engine.query("What is the revenue for Q3?")
print(response)
```

**Advanced: Combine multiple indexes**

```python
from llama_index.tools import QueryEngineTool
from llama_index.query_engine import SubQuestionQueryEngine

# Create tools from indexes
tools = [
    QueryEngineTool(
        query_engine=financial_index.as_query_engine(),
        metadata={"description": "Financial data"},
    ),
    QueryEngineTool(
        query_engine=news_index.as_query_engine(),
        metadata={"description": "News articles"},
    ),
]

# Sub-question engine (breaks query into parts)
query_engine = SubQuestionQueryEngine.from_defaults(query_engine_tools=tools)

response = query_engine.query("Compare Q3 revenue to news sentiment.")
```

**Pros:**

- Best for RAG systems
- Flexible data connectors

**Cons:**

- Steeper learning curve
- Overlaps with LangChain (confusion)

### LangGraph

**State machine for agent workflows.**

**Key concept: Define agent as graph**

```
        ┌─────────────┐
        │   Start     │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │  Research   │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   Write     │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   Review    │◄─── Loop back if needed
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │     End     │
        └─────────────┘
```

**Code:**

```python
from langgraph.graph import StateGraph, END

# Define state
class State:
    research: str = ""
    draft: str = ""
    review: str = ""

# Define nodes (functions)
def research_node(state):
    state.research = research_agent.run(state.query)
    return state

def write_node(state):
    state.draft = writing_agent.run(state.research)
    return state

def review_node(state):
    state.review = review_agent.run(state.draft)
    return state

# Build graph
graph = StateGraph(State)
graph.add_node("research", research_node)
graph.add_node("write", write_node)
graph.add_node("review", review_node)

graph.add_edge("research", "write")
graph.add_edge("write", "review")

# Conditional edge (loop back if review suggests improvements)
def should_continue(state):
    if "improve" in state.review.lower():
        return "write"
    else:
        return END

graph.add_conditional_edges("review", should_continue)

# Compile and run
app = graph.compile()
result = app.invoke({"query": "Write about AI agents"})
```

**Pros:**

- Explicit control flow
- Easy to visualize
- Deterministic

**Cons:**

- More code than simple chains
- Overkill for simple tasks

### DSPy

**Framework for optimizing prompts automatically.**

**Key idea: Treat prompts as learnable parameters.**

**Example:**

```python
import dspy

# Configure LLM
lm = dspy.OpenAI(model="gpt-4o")
dspy.settings.configure(lm=lm)

# Define task
class QA(dspy.Signature):
    """Answer questions based on context."""
    context = dspy.InputField()
    question = dspy.InputField()
    answer = dspy.OutputField()

# Use Chain of Thought
qa = dspy.ChainOfThought(QA)

# Call
response = qa(context="Paris is the capital of France.", question="What is the capital of France?")
print(response.answer)
# "Paris"

# Optimize prompts on training data
optimizer = dspy.BootstrapFewShot(metric=accuracy)
optimized_qa = optimizer.compile(qa, trainset=train_data)
```

**Use case:**

- Automatically find best prompts
- Reduce manual prompt engineering

### Haystack

**Open-source NLP framework for production.**

**Focus:**

- Document processing pipelines
- RAG systems
- Agents

**Example:**

```python
from haystack import Pipeline
from haystack.components.retrievers import InMemoryBM25Retriever
from haystack.components.generators import OpenAIGenerator

pipeline = Pipeline()
pipeline.add_component("retriever", InMemoryBM25Retriever())
pipeline.add_component("generator", OpenAIGenerator(model="gpt-4o"))

pipeline.connect("retriever", "generator")

result = pipeline.run({"query": "What is RAG?"})
```

### Semantic Kernel (Microsoft)

**GenAI orchestration for .NET/Python/Java.**

**Features:**

- Planning
- Memory
- Plugins (tools)

**Example (Python):**

```python
import semantic_kernel as sk

kernel = sk.Kernel()
kernel.add_chat_service("chat", OpenAIChatCompletion("gpt-4o"))

# Define function (plugin)
@sk.kernel_function
def get_weather(location: str) -> str:
    return f"Weather in {location}: Sunny, 20°C"

kernel.import_skill({"get_weather": get_weather}, "WeatherPlugin")

# Use planner
planner = ActionPlanner(kernel)
plan = await planner.create_plan_async("What's the weather in Paris?")

result = await plan.invoke_async()
```

### Guardrails AI

**Add validation and safety to LLM outputs.**

**Features:**

- Validate output format
- Detect toxicity
- PII detection
- Custom validators

**Example:**

```python
from guardrails import Guard
from guardrails.validators import ValidLength, ToxicLanguage

guard = Guard.from_string(
    validators=[
        ValidLength(min=100, max=500),
        ToxicLanguage(threshold=0.5, on_fail="reask"),
    ]
)

# Validate LLM output
response = llm_call("Write a summary.")
validated = guard.validate(response)

if validated.validation_passed:
    print(validated.validated_output)
else:
    # Re-ask LLM with feedback
    print(validated.error_message)
```

### AutoGen (Microsoft)

**Multi-agent conversations.**

**Example:**

```python
from autogen import AssistantAgent, UserProxyAgent

assistant = AssistantAgent(name="Assistant", llm_config={"model": "gpt-4o"})
user_proxy = UserProxyAgent(name="User", human_input_mode="NEVER")

# Start conversation
user_proxy.initiate_chat(
    assistant,
    message="Write a Python function to calculate Fibonacci."
)

# Agents converse:
# Assistant: [Writes code]
# User: [Runs code, reports output]
# Assistant: [Fixes bugs if any]
```

### CrewAI

**Multi-agent framework with roles.**

**Concepts:**

- **Agents:** Team members with roles
- **Tasks:** Assignments
- **Crew:** Team working together

**Example:**

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Researcher",
    goal="Find accurate information",
    backstory="Expert researcher with access to web search",
    tools=[search_tool],
)

writer = Agent(
    role="Writer",
    goal="Write engaging content",
    backstory="Professional content writer",
)

research_task = Task(
    description="Research AI agents",
    agent=researcher,
)

writing_task = Task(
    description="Write a blog post using research",
    agent=writer,
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
)

result = crew.kickoff()
```

### Framework Comparison

```
Framework        | Focus           | Ease of Use | Flexibility | Best For
-----------------|-----------------|-------------|-------------|------------------
LangChain        | General         | Easy        | High        | Prototyping
LlamaIndex       | RAG             | Medium      | High        | Data-heavy apps
LangGraph        | State machines  | Medium      | High        | Complex workflows
DSPy             | Prompt optimization | Hard    | Medium      | Research
Haystack         | NLP pipelines   | Medium      | High        | Production RAG
Semantic Kernel  | .NET/Enterprise | Medium      | Medium      | Enterprise
Guardrails AI    | Validation      | Easy        | Medium      | Safety-critical
AutoGen          | Multi-agent     | Easy        | Medium      | Agent conversations
CrewAI           | Team agents     | Easy        | Low         | Role-based agents
```

**Decision tree:**

```
Need RAG? → LlamaIndex
Need complex workflow? → LangGraph
Need validation? → Guardrails AI
Need multi-agent? → AutoGen or CrewAI
General purpose? → LangChain
```

---

## System Design Patterns {#design-patterns}

### Pattern 1: Sequential Chain

**Use case:** Multi-step transformation.

**Example: Translation pipeline**

```
English text → LLM (translate to French) → French text → LLM (check grammar) → Final output
```

**Code:**

```python
def sequential_chain(text):
    # Step 1: Translate
    translation = llm_call(f"Translate to French: {text}")

    # Step 2: Check grammar
    corrected = llm_call(f"Check grammar: {translation}")

    return corrected
```

**Pros:**

- Simple
- Easy to debug

**Cons:**

- Serial (slow)
- Error propagates

### Pattern 2: Map-Reduce

**Use case:** Process large documents.

**Example: Summarize 100-page doc**

```
Document → Split into chunks → Summarize each chunk (parallel) → Combine summaries → Final summary
```

**Code:**

```python
def map_reduce_summarize(document):
    # Map: Summarize chunks in parallel
    chunks = split_document(document, chunk_size=4000)

    import concurrent.futures
    with concurrent.futures.ThreadPoolExecutor() as executor:
        chunk_summaries = list(executor.map(lambda c: llm_call(f"Summarize: {c}"), chunks))

    # Reduce: Combine summaries
    combined = "\n\n".join(chunk_summaries)
    final_summary = llm_call(f"Summarize these summaries:\n{combined}")

    return final_summary
```

**Pros:**

- Handles large inputs
- Parallelizable

**Cons:**

- May lose context across chunks

### Pattern 3: Router

**Use case:** Route queries to specialized models/prompts.

**Example: Customer support**

```
User query → Router (classify intent) →
  ├─ Billing question → Billing agent
  ├─ Technical question → Technical agent
  └─ General question → General agent
```

**Code:**

```python
def router_agent(query):
    # Classify intent
    classification = llm_call(f"Classify this query: {query}\nCategories: billing, technical, general")

    # Route
    if "billing" in classification:
        return billing_agent(query)
    elif "technical" in classification:
        return technical_agent(query)
    else:
        return general_agent(query)
```

**Pros:**

- Specialized handling
- Can use different models (cheaper for simple queries)

**Cons:**

- Routing LLM call adds latency

### Pattern 4: Iterative Refinement

**Use case:** Improve output through multiple iterations.

**Example: Code generation**

```
Iteration 1: Generate code
Iteration 2: Check for bugs → Fix
Iteration 3: Check for style → Improve
Iteration 4: Add tests → Final code
```

**Code:**

```python
def iterative_refinement(task, max_iterations=3):
    output = llm_call(f"Task: {task}")

    for i in range(max_iterations):
        critique = llm_call(f"Critique this output and suggest improvements:\n{output}")

        if "no improvements" in critique.lower():
            break

        output = llm_call(f"Improve based on critique:\nOutput: {output}\nCritique: {critique}")

    return output
```

### Pattern 5: Human-in-the-Loop

**Use case:** Critical decisions require human approval.

**Example: Email generation**

```
User request → LLM generates draft → Show to user → User approves/edits → Send email
```

**Code:**

```python
def human_in_loop_email(request):
    # Generate draft
    draft = llm_call(f"Write email: {request}")

    # Show to user
    print(f"Draft:\n{draft}\n")
    approval = input("Approve? (yes/no/edit): ")

    if approval == "yes":
        send_email(draft)
    elif approval == "edit":
        edited = input("Enter edited version: ")
        send_email(edited)
    else:
        print("Email cancelled.")
```

**Use for:**

- Financial transactions
- Sending communications
- Deleting data
- Any high-stakes action

---

## Production Considerations {#production-considerations}

### Latency Optimization

**GenAI systems are slow (200ms - 10s per request).**

**Optimization strategies:**

**1. Streaming**

```python
# Show tokens as they arrive
for chunk in client.chat.completions.create(stream=True, ...):
    print(chunk, end="")
```

**2. Parallel tool calls**

```python
# Execute multiple tools simultaneously
results = await asyncio.gather(*[tool1(), tool2(), tool3()])
```

**3. Caching**

```python
# Cache identical requests
@lru_cache(maxsize=1000)
def llm_call_cached(prompt):
    return llm_call(prompt)
```

**4. Smaller models for simple tasks**

```python
# Use GPT-4o mini for classification, GPT-4o for complex reasoning
if task_complexity == "simple":
    model = "gpt-4o-mini"  # 100× cheaper, 3× faster
else:
    model = "gpt-4o"
```

**5. Prompt caching (Anthropic Claude)**

```python
# Cache system prompt (99% discount)
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    system=[{
        "type": "text",
        "text": "Long system prompt...",
        "cache_control": {"type": "ephemeral"}
    }],
    messages=[...],
)
```

### Cost Optimization

**Production costs can spiral (10K users × $0.01/query = $100/day).**

**Strategies:**

**1. Model selection**

```
Task                  | Model            | Cost/1M tokens
----------------------|------------------|----------------
Simple classification | GPT-4o mini      | $0.15
General chat          | GPT-4o           | $2.50
Complex reasoning     | o1-preview       | $15
```

**2. Prompt compression**

```python
# Long prompt: 1000 tokens
"You are an expert assistant with extensive knowledge in..."

# Compressed: 50 tokens
"You are an expert assistant."

# Savings: 95%
```

**3. Caching (exact match)**

```python
import hashlib
from functools import lru_cache

@lru_cache(maxsize=10000)
def llm_call_cached(prompt_hash):
    return llm_call(prompt)

def call_with_cache(prompt):
    prompt_hash = hashlib.md5(prompt.encode()).hexdigest()
    return llm_call_cached(prompt_hash)
```

**4. Semantic caching (similar prompts)**

```python
# Use embedding similarity to check if prompt is similar to cached ones
def semantic_cache_lookup(prompt, threshold=0.95):
    prompt_embedding = get_embedding(prompt)

    # Search cache
    similar = cache_db.similarity_search(prompt_embedding, k=1)

    if similar[0].score > threshold:
        return similar[0].response  # Cache hit
    else:
        response = llm_call(prompt)
        cache_db.store(prompt_embedding, response)
        return response
```

**5. Batching**

```python
# Process 100 queries in one batch instead of 100 individual calls
# Reduces overhead and may get bulk discounts
```

### Failure Modes

**Common failure patterns:**

**1. Infinite loops**

```
Agent: "I need to search."
Action: search("X")
Observation: [No results]
Agent: "I need to search."
Action: search("X")  # Same query again!
...
```

**Fix:**

- Set max iterations (e.g., 5)
- Detect repeated actions
- Add memory of failed attempts

**2. Tool calling errors**

```
LLM calls: get_weather(location="Paris123")  # Invalid location
System crashes
```

**Fix:**

- Validate tool inputs
- Return error messages to LLM
- Let LLM retry with corrected input

**3. Context overflow**

```
Conversation: 50 messages × 500 tokens = 25K tokens
Tools: 20 tools × 200 tokens = 4K tokens
Total: 29K tokens → Exceeds context limit
```

**Fix:**

- Compress old messages
- Summarize conversation
- Remove unused tool definitions

**4. Hallucinated tool calls**

```
LLM invents tools that don't exist:
  Action: get_bitcoin_price(currency="USD")
System: Tool not found!
```

**Fix:**

- Explicitly list available tools
- Return error: "Tool not available"
- Add fallback response

### Observability

**Production systems need monitoring.**

**What to track:**

**1. Input/Output logging**

```python
import logging

def llm_call_logged(prompt, user_id=None):
    logging.info(f"User {user_id} | Prompt: {prompt[:100]}")
    response = llm_call(prompt)
    logging.info(f"User {user_id} | Response: {response[:100]}")
    return response
```

**2. Token usage**

```python
response = client.chat.completions.create(...)
tokens_used = response.usage.total_tokens
cost = tokens_used * COST_PER_TOKEN

# Log to database
db.log_usage(user_id=user_id, tokens=tokens_used, cost=cost)
```

**3. Latency**

```python
import time

start = time.time()
response = llm_call(prompt)
latency = time.time() - start

metrics.record("llm_latency_ms", latency * 1000)
```

**4. Tool usage**

```python
# Track which tools are called most
def execute_tool(tool_name, args):
    metrics.increment(f"tool_calls.{tool_name}")
    return actual_tool_execution(tool_name, args)
```

**5. Error rates**

```python
try:
    response = llm_call(prompt)
    metrics.increment("llm_success")
except Exception as e:
    metrics.increment("llm_error")
    logging.error(f"LLM error: {e}")
    raise
```

**Observability platforms:**

- **LangSmith:** LangChain observability
- **LangFuse:** Open-source tracing
- **Helicone:** OpenAI proxy with logging
- **Weights & Biases:** Experiment tracking

---

## System Design Interview Examples

### Example 1: AI-Powered Customer Support

**Requirements:**

- Handle 10K concurrent users
- 24/7 availability
- Respond within 2 seconds
- Support ticket creation
- Escalate to human if needed

**Architecture:**

```
┌──────────┐
│  User    │
└────┬─────┘
     │
┌────▼─────────────────────────────────┐
│  Load Balancer                       │
└────┬─────────────────────────────────┘
     │
┌────▼─────────────────────────────────┐
│  Intent Classifier (GPT-4o mini)     │ ← Fast, cheap
│  Output: billing, technical, general  │
└────┬─────────────────────────────────┘
     │
     ├───────────┬───────────┐
     │           │           │
┌────▼──────┐ ┌─▼──────┐ ┌─▼────────┐
│  Billing  │ │Technical│ │  General │
│  Agent    │ │ Agent   │ │  Agent   │
└────┬──────┘ └─┬──────┘ └─┬────────┘
     │          │          │
     └──────────┴──────────┘
                │
         ┌──────▼───────┐
         │  RAG System  │ ← Knowledge base
         │  (Vector DB) │
         └──────┬───────┘
                │
         ┌──────▼───────┐
         │  Response    │
         │  + Metadata  │
         └──────────────┘
```

**Components:**

1. **Intent classifier:**
   - Model: GPT-4o mini (cheap, fast)
   - Purpose: Route to correct agent
   - Latency: ~100ms

2. **Specialized agents:**
   - Billing Agent: Access to billing DB, can create refund tickets
   - Technical Agent: Access to docs, can create support tickets
   - General Agent: FAQs, routing

3. **RAG system:**
   - Vector DB: Pinecone
   - Documents: Product docs, FAQs, past tickets (10K docs)
   - Chunking: Recursive, 1000 tokens, 200 overlap

4. **Escalation logic:**

   ```python
   if confidence < 0.7 or user_requests_human:
       create_ticket(priority="high")
       return "I've created a ticket. A human agent will help you shortly."
   ```

5. **Tools:**
   - `search_knowledge_base(query)`: RAG search
   - `create_ticket(category, description)`: Create support ticket
   - `get_order_status(order_id)`: Fetch from DB
   - `process_refund(order_id)`: Requires human approval

**Performance targets:**

- 90% queries answered without human
- <2s latency (p95)
- <$0.01 per query

**Cost estimate:**

- 10K queries/day
- Avg 500 tokens in + 300 tokens out = 800 tokens
- GPT-4o mini: $0.15/1M input, $0.60/1M output
- Cost: 10K × (500 × $0.15 + 300 × $0.60) / 1M = **$2.55/day**

### Example 2: Code Review Assistant

**Requirements:**

- Review GitHub PRs automatically
- Check for bugs, style issues, security
- Suggest improvements
- Run on every PR

**Architecture:**

```
GitHub PR opened
     │
     ▼
┌─────────────────┐
│  Webhook        │
│  Receiver       │
└────┬────────────┘
     │
┌────▼────────────┐
│  PR Analyzer    │
│  - Get diff     │
│  - Get context  │
└────┬────────────┘
     │
     ├─────────┬─────────┬─────────┐
     │         │         │         │
┌────▼──────┐ │         │         │
│  Bug      │ │         │         │
│  Detector │ │         │         │
└────┬──────┘ │         │         │
     │    ┌───▼───────┐ │         │
     │    │  Style    │ │         │
     │    │  Checker  │ │         │
     │    └───┬───────┘ │         │
     │        │   ┌─────▼──────┐  │
     │        │   │  Security  │  │
     │        │   │  Scanner   │  │
     │        │   └─────┬──────┘  │
     │        │         │  ┌──────▼──────┐
     │        │         │  │ Performance │
     │        │         │  │ Analyzer    │
     │        │         │  └──────┬──────┘
     └────────┴─────────┴─────────┘
                  │
         ┌────────▼─────────┐
         │  Aggregator      │
         │  (GPT-4o)        │
         │  Synthesize      │
         │  feedback        │
         └────────┬─────────┘
                  │
         ┌────────▼─────────┐
         │  Post comment    │
         │  on GitHub PR    │
         └──────────────────┘
```

**Tools:**

```python
tools = [
    {
        "name": "get_pr_diff",
        "description": "Get code changes in PR",
    },
    {
        "name": "get_file_content",
        "description": "Read full file for context",
    },
    {
        "name": "run_linter",
        "description": "Run ESLint/Pylint on code",
    },
    {
        "name": "check_tests",
        "description": "Check if tests exist and pass",
    },
]
```

**Review prompt:**

```python
REVIEW_PROMPT = """
You are an expert code reviewer. Analyze this PR and provide feedback.

Code diff:
{diff}

Focus on:
1. Correctness: Are there bugs or logic errors?
2. Security: Are there vulnerabilities (SQL injection, XSS, etc.)?
3. Performance: Are there inefficiencies (N+1 queries, unnecessary loops)?
4. Style: Does it follow best practices?
5. Tests: Are there tests for new code?

Provide feedback in this format:
### Critical Issues
- [Issue 1]

### Suggestions
- [Suggestion 1]

### Positive
- [What was done well]
"""
```

**Cost:**

- Avg PR: 500 lines changed
- Diff tokens: ~2000
- Review output: ~1000 tokens
- GPT-4o: $2.50/1M input, $10/1M output
- **Cost per PR: $0.02**
- 100 PRs/day: **$2/day** (negligible)

### Example 3: Multi-Agent Travel Planner

**Requirements:**

- User provides: destination, dates, budget, preferences
- System books: flights, hotels, activities
- Optimize for cost and user preferences

**Architecture:**

```
        ┌────────────────┐
        │ User Input     │
        └───────┬────────┘
                │
        ┌───────▼────────┐
        │ Orchestrator   │
        │ (GPT-4o)       │
        └───────┬────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
┌───▼─────┐ ┌──▼──────┐ ┌─▼─────────┐
│ Flight  │ │ Hotel   │ │ Activity  │
│ Agent   │ │ Agent   │ │ Agent     │
└───┬─────┘ └──┬──────┘ └─┬─────────┘
    │          │          │
    └──────────┴──────────┘
               │
        ┌──────▼──────┐
        │  Budget     │
        │  Optimizer  │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │ Booking     │
        │ Confirmation│
        └─────────────┘
```

**Agent responsibilities:**

**Flight Agent:**

```python
class FlightAgent:
    def search_flights(self, origin, destination, dates):
        results = flight_api.search(origin, destination, dates)
        return self.rank_by_price_and_duration(results)

    def book_flight(self, flight_id):
        return flight_api.book(flight_id)
```

**Hotel Agent:**

```python
class HotelAgent:
    def search_hotels(self, location, dates, budget):
        results = hotel_api.search(location, dates)
        filtered = [h for h in results if h.price <= budget]
        return self.rank_by_rating(filtered)

    def book_hotel(self, hotel_id):
        return hotel_api.book(hotel_id)
```

**Activity Agent:**

```python
class ActivityAgent:
    def suggest_activities(self, location, preferences):
        # Use RAG to find activities
        activities = rag_search(f"activities in {location} for {preferences}")
        return activities
```

**Orchestrator:**

```python
class TravelOrchestrator:
    def plan_trip(self, user_input):
        # Parse input with LLM
        parsed = self.parse_input(user_input)

        # Coordinate agents
        flights = self.flight_agent.search_flights(...)
        hotels = self.hotel_agent.search_hotels(...)
        activities = self.activity_agent.suggest_activities(...)

        # Optimize
        plan = self.optimize(flights, hotels, activities, budget=parsed.budget)

        # Get user confirmation
        confirmed = self.get_user_confirmation(plan)

        if confirmed:
            # Book everything
            self.book_all(plan)

        return plan
```

**Challenges:**

- Coordinating multiple agents
- Handling booking failures (rollback needed)
- User confirmation for financial transactions

---

## Practical Takeaways

### For Beginners (0-2 years)

**Learn:**

- Use OpenAI/Anthropic API directly (no framework at first)
- Implement simple tool calling (calculator, search)
- Build ReAct agent from scratch
- Add conversation memory (buffer)

**Build:**

- Chatbot with memory
- Agent with 3-5 tools
- Simple RAG system

**Avoid:**

- Complex multi-agent systems
- Over-engineering with frameworks
- Agents when simple prompts work

### For Mid-Level (2-5 years)

**Learn:**

- Master LangChain or LlamaIndex
- Build multi-agent systems (2-3 agents)
- Implement different memory types
- Add observability (logging, tracing)
- Optimize for latency and cost

**Build:**

- Production RAG system
- Customer support agent
- Code review agent

**Focus:**

- System design patterns (router, map-reduce, iterative)
- Error handling and retries
- Production concerns (cost, latency, reliability)

### For Senior (5+ years)

**Learn:**

- Design complex multi-agent architectures
- LangGraph for state machines
- Custom orchestration (no framework)
- Scaling to 100K+ users
- Security and safety

**Build:**

- Enterprise GenAI platform
- Multi-agent workflows with 5+ agents
- Custom frameworks for company needs

**Focus:**

- Architecture decisions (when to use agents vs simple prompts)
- Cost-performance-quality trade-offs
- Production incidents and debugging
- Team enablement (internal tools, SDKs)

---

## Interview Questions

### Junior Level

**Q: What is tool calling in LLMs?**

**A:**
Tool calling allows LLMs to decide when and how to call external functions (APIs, search, calculators). The LLM generates a function call with parameters, the system executes it, and the result is fed back to the LLM to synthesize a final response.

**Q: Explain the ReAct pattern.**

**A:**
ReAct (Reasoning + Acting) is an agent pattern with 3 steps repeated in a loop:

1. Thought: Reason about what to do next
2. Action: Execute a tool
3. Observation: See the result
   Continue until goal is achieved or max iterations reached.

**Q: What is the difference between short-term and long-term memory in agents?**

**A:**

- Short-term memory: Recent conversation messages (last 10-20 turns) in a buffer. Provides conversation flow.
- Long-term memory: Facts stored in vector DB, retrieved semantically. Provides persistent knowledge.

### Mid Level

**Q: Design a code review agent for a team of 20 engineers.**

**A:**
**Architecture:**

1. GitHub webhook triggers on PR
2. Fetch PR diff and file contents
3. Run parallel checks:
   - Linter (ESLint/Pylint)
   - Security scan (Bandit, Semgrep)
   - LLM review (GPT-4o): bugs, logic, style
4. Aggregate feedback
5. Post comment on GitHub
6. If critical issues, request changes; else approve

**Tools:** `get_pr_diff`, `get_file`, `run_linter`, `run_security_scan`
**Model:** GPT-4o (500 lines × 4 tokens/line = 2K tokens in, 1K out = $0.01/review)
**Cost:** 100 PRs/month = $1/month

**Q: How would you prevent infinite loops in an agent?**

**A:**

1. Set max iterations (5-10)
2. Track action history, detect repeated actions (same tool + args)
3. If repeated 2×, intervene:
   - Force different tool
   - Ask user for help
   - Return partial results
4. Add timeout (30s total execution)
5. Monitor via observability (alert if loops detected)

**Q: How do you optimize latency in a multi-agent system?**

**A:**

1. Parallel agent execution (not serial)
2. Use faster models for simple agents (GPT-4o mini)
3. Streaming responses (show progress)
4. Cache frequent queries (semantic caching)
5. Preload tools and models
6. Reduce token usage (compress prompts)
7. Use prompt caching (Anthropic Claude)
8. Set short timeouts to fail fast

### Senior Level

**Q: Design an autonomous AI agent that can browse the web, run code, and write reports.**

**A:**
**Architecture:**

```
┌────────────────────────────────────┐
│  User Goal                         │
│  "Research AI trends and create    │
│   investment report"               │
└────────┬───────────────────────────┘
         │
┌────────▼───────────────────────────┐
│  Planning Agent (GPT-4o)           │
│  - Decompose goal into tasks       │
│  - Create execution plan           │
└────────┬───────────────────────────┘
         │
┌────────▼───────────────────────────┐
│  Task Queue                        │
│  1. Search for AI trends           │
│  2. Analyze data                   │
│  3. Generate report                │
└────────┬───────────────────────────┘
         │
┌────────▼───────────────────────────┐
│  Execution Agent (ReAct)           │
│  - Execute each task               │
│  - Use tools                       │
│  - Store results                   │
└────────┬───────────────────────────┘
         │
    ┌────┴────┬─────────┬──────────┐
    │         │         │          │
┌───▼──┐ ┌───▼───┐ ┌───▼───┐ ┌───▼────┐
│Search│ │Browser│ │ Code  │ │ File   │
│ API  │ │ Tool  │ │ Exec  │ │ System │
└───┬──┘ └───┬───┘ └───┬───┘ └───┬────┘
    │        │         │         │
    └────────┴─────────┴─────────┘
                 │
        ┌────────▼─────────┐
        │  Memory System   │
        │  - Task history  │
        │  - Results       │
        │  - Insights      │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │  Report Generator│
        │  (GPT-4o)        │
        └────────┬─────────┘
                 │
        ┌────────▼─────────┐
        │  Final Report    │
        │  (Markdown/PDF)  │
        └──────────────────┘
```

**Key components:**

1. **Planning Agent:**
   - Breaks goal into subtasks
   - Creates execution plan
   - Estimates time and cost

2. **Execution Agent (ReAct):**
   - Iteratively executes tasks
   - Uses tools based on need
   - Stores intermediate results

3. **Tools:**
   - `search_web(query)`: Google/Bing API
   - `browse_url(url)`: Fetch and parse webpage
   - `execute_code(code)`: Run Python in sandbox
   - `read_file(path)`, `write_file(path, content)`
   - `create_chart(data)`: Generate visualizations

4. **Memory:**
   - Short-term: Current task context
   - Long-term: All task results and insights (vector DB)
   - Working: Scratchpad for reasoning

5. **Safety:**
   - Sandbox code execution (Docker container)
   - Rate limit web requests (10/min)
   - Require approval for file writes
   - Timeout per task (5 min)

**Challenges:**

1. **Infinite loops:**
   - Solution: Max 20 steps, detect repeated actions

2. **Unreliable tools:**
   - Solution: Retry with exponential backoff, fallback to alternative tools

3. **Context overflow:**
   - Solution: Compress memory after each task, keep only summaries

4. **Cost:**
   - Solution: Use GPT-4o mini for simple tasks, GPT-4o for complex reasoning

5. **Security:**
   - Solution: Sandbox all code execution, whitelist allowed tools, audit logs

**Performance:**

- Total latency: 2-5 minutes (acceptable for complex research)
- Cost: ~$1-2 per report (50K tokens usage)
- Success rate: 80% (needs human review 20% of time)

**Scaling:**

- Run agents in parallel for multiple users
- Queue system for task management
- Async execution

---

## Summary

**Core concepts:**

1. **GenAI systems:** LLM + tools + memory + orchestration
2. **Tool calling:** LLM generates function calls, system executes
3. **Agents:** Multi-step reasoning with tools (ReAct, planning, reflection)
4. **Memory:** Short-term (buffer), long-term (vector DB), working (scratchpad)
5. **Multi-agent:** Coordinator-worker, debate, pipeline, hierarchical

**Design patterns:**

- Sequential chain (multi-step transformation)
- Map-reduce (large documents)
- Router (specialized handling)
- Iterative refinement (improve quality)
- Human-in-the-loop (critical decisions)

**Frameworks:**

- **LangChain:** General-purpose, easy to prototype
- **LlamaIndex:** Best for RAG
- **LangGraph:** State machines for complex workflows
- **DSPy:** Prompt optimization
- **Guardrails AI:** Output validation

**Production concerns:**

- Latency: Streaming, parallel calls, caching, smaller models
- Cost: Model selection, prompt compression, caching
- Reliability: Retries, error handling, max iterations
- Observability: Logging, tracing, metrics

**Interview focus:**

- System design (architecture diagrams, trade-offs)
- Tool safety (validation, sandboxing)
- Failure modes (loops, context overflow, errors)
- Cost-performance trade-offs

**Next:** Part 9 - Production GenAI (deployment, monitoring, security)
