# LightRAG Expert Agent System Prompt Guide
## Based on Internal Architecture & Implementation

> **Purpose**: This guide enables AI agents to expertly query and retrieve data from LightRAG using MCP, optimized for cross-source conceptual connections and cross-domain synthesis.

---

## 1. Understanding LightRAG's Core Architecture

### What LightRAG Really Is
LightRAG is a **three-layer semantic memory system** that creates a knowledge graph from your reference library:

1. **Vector Layer (Entities)**: Embeddings of all extracted entities (people, concepts, frameworks, patterns)
2. **Vector Layer (Relationships)**: Embeddings of all connections between entities
3. **Graph Layer**: Structural connections showing how entities relate across documents

**Critical Insight**: Unlike simple vector search, LightRAG enables **multi-hop reasoning** by combining:
- Entity similarity (finding relevant concepts)
- Relationship traversal (connecting ideas across sources)
- Document chunk retrieval (preserving original context)

### Knowledge Graph Structure

```
Document → Chunks → LLM Extraction → Entities + Relationships

Example from your use case:
"Event-Driven Architecture" PDF
    ↓
Chunk 1: "Martin Fowler discusses schema evolution..."
    ↓
Entities: [Schema Registry, Event Schema, Martin Fowler]
Relationships: [
    (Schema Registry, Event Schema, "manages versions of"),
    (Martin Fowler, Schema Registry, "advocates for")
]
    ↓
Knowledge Graph Node: Schema Registry
    - Type: Pattern
    - Description: "Centralized service for managing event schemas"
    - Source: fowler_microservices.pdf, chunk_42
    - Connected to: [Event Schema, Kafka, Compatibility Types]
```

**Entity Structure** (what you get back):
- `entity_name`: The concept identifier
- `entity_type`: Category (Person, Concept, Framework, Pattern, etc.)
- `description`: LLM-generated summary from source material
- `source_id`: Which document chunks mention this entity
- `file_path`: Original source document
- `rank`: Number of connections (highly connected = important concept)

**Relationship Structure** (how concepts connect):
- `src_id`: Source entity
- `tgt_id`: Target entity
- `description`: How they're related (from original text)
- `keywords`: Key terms from the relationship
- `weight`: Strength of connection (frequency, co-occurrence)
- `source_id`: Which chunks establish this relationship

---

## 2. Query Modes: Choosing the Right Strategy

LightRAG has **6 query modes**. For your use case (cross-source synthesis), you'll primarily use **3 modes**:

### Mode 1: `"mix"` (DEFAULT) - For Comprehensive Cross-Source Queries

**When to use:**
- Questions requiring synthesis across multiple sources
- "How do different authors approach X?"
- "Connect concepts from source A to source B"
- General exploratory questions

**What it does:**
1. Vector search on **entities** (finds relevant concepts)
2. Vector search on **relationships** (finds relevant connections)
3. Vector search on **chunks** (finds relevant raw text)
4. Merges all three sources with frequency tracking
5. Builds comprehensive context for LLM

**Example query from your use case:**
```python
query = "How do different authors approach event schema evolution in distributed systems?"
mode = "mix"  # Gets entities (Schema Registry, Event Schema),
              # relationships (versioning strategies),
              # and chunks (original explanations from each author)
```

**Result structure:**
- Entities: [Schema Registry, Event Schema, Compatibility Types, Semantic Versioning]
- Relationships: [(Schema Registry, Kafka, "integrates with"), (Fowler, Schema Registry, "recommends")]
- Chunks: Original paragraphs from Fowler's book, Kafka docs, Stripe podcast, academic paper
- LLM synthesizes across all sources showing different approaches

### Mode 2: `"global"` - For Relationship-Focused Queries

**When to use:**
- Questions about connections between concepts
- "What's the relationship between X and Y?"
- "How does concept A relate to concept B?"
- Cross-domain mapping queries

**What it does:**
1. Extracts high-level keywords from query
2. Vector search on **relationship embeddings** (not entities!)
3. Retrieves entities involved in those relationships
4. Builds context focused on connections

**Example query from your use case:**
```python
query = "Connect bounded context from DDD to TOGAF enterprise architecture framework"
mode = "global"  # Prioritizes finding relationships between DDD and TOGAF concepts
```

**Why this works:**
- Searches relationships first (how concepts connect)
- Pulls in relevant entities second (what those concepts are)
- Ideal for cross-domain synthesis where you need mapping between different frameworks

### Mode 3: `"hybrid"` - For Balanced Entity + Relationship Queries

**When to use:**
- Questions requiring both specific entities AND their relationships
- "Tell me about X and how it connects to Y"
- Uncertain whether entity-first or relationship-first is better

**What it does:**
1. Runs LOCAL mode (entity search) in parallel with GLOBAL mode (relationship search)
2. Merges results with round-robin strategy (maintains diversity)
3. Deduplicates entities and relationships
4. Builds context from both perspectives

**Example query:**
```python
query = "Explain the Schema Registry pattern and how it connects to Kafka's compatibility model"
mode = "hybrid"  # Gets both the entity (Schema Registry) and relationships (to Kafka)
```

### Quick Reference: Mode Selection Decision Tree

```
┌─────────────────────────────────────────────────────────┐
│ What kind of query is this?                             │
└─────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
  ┌─────▼─────┐  ┌──────▼──────┐  ┌────▼─────┐
  │ Synthesis │  │ Connection  │  │ Specific │
  │ across    │  │ mapping     │  │ entity   │
  │ sources   │  │ between     │  │ lookup   │
  └─────┬─────┘  └──────┬──────┘  └────┬─────┘
        │               │               │
   Use "mix"       Use "global"    Use "local"
        │               │               │
  ┌─────▼──────────────────────────────▼──────┐
  │ Not sure? Use "mix" - it's the default    │
  │ and works for most comprehensive queries  │
  └────────────────────────────────────────────┘
```

### Modes You'll Rarely Need

- **`"local"`**: For simple entity lookups ("Who is Alice?") - too narrow for your use case
- **`"naive"`**: Simple vector search, no knowledge graph - loses multi-hop reasoning
- **`"bypass"`**: No retrieval, just LLM - when you don't need the knowledge base

---

## 3. Query Parameters: Fine-Tuning Retrieval

### Essential Parameters for Your Use Case

```python
from lightrag.base import QueryParam

# For cross-source synthesis (broad exploration)
param = QueryParam(
    mode="mix",
    top_k=15,                      # More entities/relationships
    chunk_top_k=10,                # More source chunks
    max_entity_tokens=1000,        # More entity context
    max_relation_tokens=800,       # More relationship context
    max_total_tokens=4000,         # Larger context window
    only_need_context=False        # Get LLM synthesis, not just data
)

# For cross-domain mapping (relationship focus)
param = QueryParam(
    mode="global",
    top_k=10,                      # Focused on key relationships
    chunk_top_k=5,                 # Supporting chunks
    max_relation_tokens=1000,      # Prioritize relationship descriptions
    max_entity_tokens=600,         # Less entity detail needed
    max_total_tokens=3000
)

# For quick exploration (faster, lighter)
param = QueryParam(
    mode="mix",
    top_k=8,
    chunk_top_k=5,
    max_total_tokens=2000
)
```

### Parameter Explanations

| Parameter | Purpose | Your Use Case Guidance |
|-----------|---------|------------------------|
| `mode` | Query strategy | Use `"mix"` for synthesis, `"global"` for cross-domain mapping |
| `top_k` | # of entities/relationships to retrieve | 10-20 for comprehensive answers, 5-8 for focused queries |
| `chunk_top_k` | # of raw document chunks (mix mode only) | 5-10 to preserve original author voices |
| `max_entity_tokens` | Token budget for entity descriptions | 800-1200 for detailed concept explanations |
| `max_relation_tokens` | Token budget for relationships | 600-1000 for cross-source connections |
| `max_total_tokens` | Total context size | 3000-5000 for multi-source synthesis |
| `only_need_context` | Skip LLM synthesis, return raw data | `False` (you want synthesis), `True` (just get data) |
| `stream` | Stream LLM response | `True` for real-time, `False` for complete response |

### Token Budget Strategy

LightRAG allocates tokens like this:
```
Total Budget (max_total_tokens = 4000)
├─ System Prompt: ~600 tokens (fixed)
├─ User Query: ~100 tokens (your question)
├─ Entity Descriptions: max_entity_tokens (1000) ← Concept definitions
├─ Relationship Descriptions: max_relation_tokens (800) ← How concepts connect
└─ Document Chunks: Remaining (~1500) ← Original source text

= Total: 4000 tokens of context for LLM to synthesize
```

**For your use case:**
- **High entity tokens**: Need detailed explanations of concepts (Schema Registry, Bounded Context, etc.)
- **High relationship tokens**: Need to understand how concepts connect across sources
- **Moderate chunk tokens**: Need original text for citation and voice preservation

---

## 4. Understanding Query Results

### Result Structure

```python
result = await rag.aquery_llm(query, param)

# Result structure:
{
    "status": "success",
    "data": {
        "entities": [...]        # Relevant concepts found
        "relationships": [...]   # How concepts connect
        "chunks": [...]          # Original text excerpts
        "references": [...]      # Source citations
    },
    "metadata": {
        "query_mode": "mix",
        "keywords": {
            "high_level": [...],  # Relationship keywords extracted
            "low_level": [...]    # Entity keywords extracted
        },
        "processing_info": {
            "total_entities_found": 42,
            "entities_after_truncation": 15,
            "merged_chunks_count": 87,
            "final_chunks_count": 10
        }
    },
    "llm_response": {
        "content": "..."  # Synthesized answer
    }
}
```

### Interpreting Entities

```python
entity = {
    "entity_name": "Schema Registry",
    "entity_type": "Pattern",
    "description": "Centralized service for managing event schema versions, ensuring compatibility across producers and consumers",
    "source_id": "chunk_123,chunk_456,chunk_789",  # Multiple chunks = mentioned across sources
    "file_path": "fowler_microservices.pdf,kafka_docs.pdf,stripe_podcast.txt",
    "rank": 8,  # High rank = well-connected concept (important!)
    "reference_id": "ref_001"
}
```

**Key insights:**
- `rank`: High rank (>5) means central concept in your knowledge graph
- Multiple `source_id`: Concept appears across multiple sources (validates cross-source query)
- `file_path`: Shows which sources discuss this concept

### Interpreting Relationships

```python
relationship = {
    "src_id": "Schema Registry",
    "tgt_id": "Kafka",
    "description": "Schema Registry integrates with Kafka to provide centralized schema management for topics",
    "keywords": ["integration", "schema management", "topics"],
    "weight": 0.85,  # High weight = strong connection
    "source_id": "chunk_456,chunk_789",
    "file_path": "kafka_docs.pdf,stripe_podcast.txt",
    "reference_id": "ref_002"
}
```

**Key insights:**
- `weight`: Higher weight (>0.7) = stronger/more frequently mentioned connection
- Multiple sources: Same relationship described by multiple authors (validates synthesis)
- `keywords`: Help understand the nature of the connection

### Interpreting Chunks

```python
chunk = {
    "content": "Martin Fowler advocates for using a Schema Registry as a central authority for event schemas. This ensures that all services agree on the structure of events, preventing compatibility issues during schema evolution.",
    "file_path": "fowler_microservices.pdf",
    "chunk_id": "chunk_456",
    "reference_id": "ref_001"
}
```

**Key insights:**
- Preserves original author voice ("Martin Fowler advocates...")
- Can cite specific sources in your response
- Combines with other chunks from different sources for synthesis

---

## 5. Practical Prompting Strategies for Your Use Case

### Use Case 1: Cross-Source Conceptual Connections

**Your example:** "How do different authors approach event schema evolution in distributed systems?"

**Optimal query strategy:**
```python
query = "Compare approaches to event schema evolution from Martin Fowler, Kafka documentation, Stripe architecture, and academic research"

param = QueryParam(
    mode="mix",          # Need entities (patterns) + relationships + original text
    top_k=15,            # Get multiple approaches
    chunk_top_k=10,      # Preserve each author's voice
    max_entity_tokens=1000,
    max_relation_tokens=800,
    max_total_tokens=4500
)

result = await rag.aquery_llm(query, param)
```

**Why this works:**
- `mode="mix"`: Finds relevant patterns (entities), their connections (relationships), and original explanations (chunks)
- High `top_k`: Retrieves multiple approaches (Schema Registry, Compatibility Types, Versioning Strategies)
- High `chunk_top_k`: Preserves original text from each source
- LLM synthesizes across all sources showing similarities and differences

**Expected result structure:**
```
Entities found:
- Schema Registry (Fowler) - rank: 12
- Compatibility Types (Kafka) - rank: 10
- Semantic Versioning (Academic) - rank: 8
- Schema Evolution Strategy (Stripe) - rank: 7

Relationships found:
- (Schema Registry, Compatibility Types, "implements")
- (Fowler, Schema Registry, "advocates for")
- (Kafka, Backward Compatibility, "defaults to")

Chunks preserve:
- Fowler: "I recommend using a Schema Registry..."
- Kafka Docs: "Kafka supports three compatibility types..."
- Stripe Podcast: "We version our events using..."
- Academic Paper: "Semantic versioning ensures..."

LLM synthesis:
"Different authors approach event schema evolution with compatible but distinct strategies:
1. Martin Fowler advocates for Schema Registry pattern...
2. Kafka documentation focuses on compatibility types...
3. Stripe's architecture emphasizes...
4. Academic research proposes semantic versioning...
These approaches align on [commonalities] but differ in [differences]..."
```

### Use Case 2: Cross-Domain Synthesis

**Your example:** "Connect bounded context from DDD to TOGAF enterprise architecture framework"

**Optimal query strategy:**
```python
query = "Map Domain-Driven Design bounded contexts to TOGAF application components and identify alignment and conflicts"

param = QueryParam(
    mode="global",       # Prioritize relationships between frameworks
    top_k=12,            # Get key concepts from both domains
    chunk_top_k=6,       # Supporting context
    max_relation_tokens=1200,  # Emphasize cross-domain connections
    max_entity_tokens=800,
    max_total_tokens=4000
)

result = await rag.aquery_llm(query, param)
```

**Why this works:**
- `mode="global"`: Searches for relationships FIRST (cross-domain connections)
- High `max_relation_tokens`: Prioritizes explanations of how concepts map between DDD and TOGAF
- Moderate `top_k`: Focuses on key concepts from each framework
- LLM identifies alignments, conflicts, and mapping strategies

**Expected result structure:**
```
Relationships found (cross-domain):
- (Bounded Context, Application Component, "aligns with TOGAF's concept of")
- (DDD Context Map, TOGAF Integration View, "maps to")
- (Ubiquitous Language, TOGAF Business Vocabulary, "corresponds to")
- (Anti-Corruption Layer, TOGAF Integration Boundary, "similar defensive pattern")

Entities from both domains:
- Bounded Context (DDD) - rank: 15
- Application Component (TOGAF) - rank: 12
- Context Map (DDD) - rank: 10
- Integration View (TOGAF) - rank: 9

LLM synthesis:
"DDD's bounded contexts align with TOGAF's application components in that both...
However, they conflict in governance models: DDD emphasizes team autonomy while TOGAF...
To map between them: treat each bounded context as a TOGAF application component,
map context maps to TOGAF integration views, and translate ubiquitous language to
business vocabulary in the TOGAF architecture repository..."
```

### Use Case 3: Deep Exploration of a Single Concept Across Sources

**Query:** "What do different sources say about the Schema Registry pattern?"

**Optimal query strategy:**
```python
query = "Schema Registry pattern across all sources - definitions, implementations, tradeoffs, and recommendations"

param = QueryParam(
    mode="local",        # Entity-focused: Schema Registry
    top_k=10,            # Get the entity + related concepts
    chunk_top_k=8,       # Multiple source excerpts
    max_entity_tokens=1200,
    max_total_tokens=3500
)
```

**Why this works:**
- `mode="local"`: Focuses on "Schema Registry" entity
- Retrieves all chunks mentioning Schema Registry
- Gets related entities (Kafka, Compatibility, Event Schema)
- Builds comprehensive view from all sources

---

## 6. Advanced Techniques

### Technique 1: Iterative Refinement

For complex synthesis, query in multiple passes:

```python
# Pass 1: Broad exploration
query1 = "Event schema evolution approaches in distributed systems"
param1 = QueryParam(mode="mix", top_k=20, chunk_top_k=15)
result1 = await rag.aquery_llm(query1, param1)

# Extract key concepts from result1
key_concepts = extract_entities(result1["data"]["entities"])

# Pass 2: Deep dive on specific connection
query2 = f"How do {key_concepts[0]} and {key_concepts[1]} relate? Show conflicts and alignments"
param2 = QueryParam(mode="global", top_k=10)
result2 = await rag.aquery_llm(query2, param2)
```

### Technique 2: Source Attribution

Use `references` to cite sources in your response:

```python
result = await rag.aquery_llm(query, param)

# Build citation map
citations = {}
for ref in result["data"]["references"]:
    citations[ref["reference_id"]] = ref["file_path"]

# In your response:
for entity in result["data"]["entities"]:
    source = citations[entity["reference_id"]]
    print(f"{entity['entity_name']} (from {source}): {entity['description']}")
```

### Technique 3: Relationship Traversal

For multi-hop reasoning, extract relationship chains:

```python
result = await rag.aquery_llm(query, param)

# Build relationship graph
relationships = result["data"]["relationships"]
graph = {}
for rel in relationships:
    if rel["src_id"] not in graph:
        graph[rel["src_id"]] = []
    graph[rel["src_id"]].append((rel["tgt_id"], rel["description"]))

# Traverse: Bounded Context → ? → TOGAF Application Component
def find_path(graph, start, end, path=[]):
    path = path + [start]
    if start == end:
        return [path]
    if start not in graph:
        return []
    paths = []
    for node, desc in graph[start]:
        if node not in path:
            newpaths = find_path(graph, node, end, path)
            paths.extend(newpaths)
    return paths

paths = find_path(graph, "Bounded Context", "TOGAF Application Component")
# Shows multi-hop connections between concepts
```

### Technique 4: Data-Only Retrieval

For when you want to process data yourself (not LLM synthesis):

```python
param = QueryParam(
    mode="mix",
    top_k=15,
    only_need_context=True  # Skip LLM synthesis
)

result = await rag.aquery_data(query, param)
# Returns only data (entities, relationships, chunks) - no LLM response
# Process data with custom logic
```

---

## 7. Integration with MCP

### MCP Wrapper Pattern

LightRAG doesn't have native MCP support, but you can create an MCP wrapper:

```python
# MCP Server Implementation
from lightrag import LightRAG
from lightrag.base import QueryParam

class LightRAGMCPServer:
    def __init__(self, working_dir: str):
        self.rag = LightRAG(working_dir=working_dir)

    # MCP Tool: Query LightRAG
    async def query_tool(
        self,
        query: str,
        mode: str = "mix",
        top_k: int = 10,
        chunk_top_k: int = 5
    ) -> dict:
        """
        Query LightRAG knowledge graph

        Args:
            query: Natural language question
            mode: Query mode (mix/global/local/hybrid)
            top_k: Number of entities/relationships to retrieve
            chunk_top_k: Number of document chunks to retrieve

        Returns:
            Synthesized answer with entities, relationships, and sources
        """
        param = QueryParam(
            mode=mode,
            top_k=top_k,
            chunk_top_k=chunk_top_k
        )

        result = await self.rag.aquery_llm(query, param)

        return {
            "answer": result["llm_response"]["content"],
            "entities": [
                {
                    "name": e["entity_name"],
                    "type": e["entity_type"],
                    "description": e["description"],
                    "source": e["file_path"]
                }
                for e in result["data"]["entities"]
            ],
            "relationships": [
                {
                    "from": r["src_id"],
                    "to": r["tgt_id"],
                    "description": r["description"],
                    "source": r["file_path"]
                }
                for r in result["data"]["relationships"]
            ],
            "sources": [
                ref["file_path"]
                for ref in result["data"]["references"]
            ]
        }

    # MCP Tool: Insert document
    async def insert_tool(
        self,
        content: str,
        file_path: str
    ) -> dict:
        """
        Add document to LightRAG knowledge graph

        Args:
            content: Document text content
            file_path: Source file path (for reference)

        Returns:
            Status and statistics
        """
        await self.rag.ainsert(content)
        return {
            "status": "success",
            "message": f"Inserted document: {file_path}"
        }
```

### MCP Tool Definitions

```json
{
  "tools": [
    {
      "name": "lightrag_query",
      "description": "Query LightRAG knowledge graph for cross-source synthesis and multi-hop reasoning",
      "parameters": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "Natural language question to query the knowledge graph"
          },
          "mode": {
            "type": "string",
            "enum": ["mix", "global", "local", "hybrid"],
            "default": "mix",
            "description": "Query mode: 'mix' for synthesis, 'global' for relationships, 'local' for entities"
          },
          "top_k": {
            "type": "integer",
            "default": 10,
            "description": "Number of entities/relationships to retrieve (5-20 recommended)"
          }
        },
        "required": ["query"]
      }
    },
    {
      "name": "lightrag_insert",
      "description": "Add document to LightRAG knowledge graph",
      "parameters": {
        "type": "object",
        "properties": {
          "content": {
            "type": "string",
            "description": "Document text content"
          },
          "file_path": {
            "type": "string",
            "description": "Source file path for reference"
          }
        },
        "required": ["content", "file_path"]
      }
    }
  ]
}
```

---

## 8. System Prompt Template

Here's a complete system prompt template for your agent:

```markdown
# LightRAG Expert Agent

You are an expert at querying and retrieving information from a LightRAG knowledge graph containing books, technical documentation, research papers, podcast transcripts, and articles.

## Your Capabilities

You have access to a LightRAG knowledge graph through MCP that enables:
1. **Cross-source synthesis**: Connect ideas from different authors and sources
2. **Multi-hop reasoning**: Traverse relationships between concepts across documents
3. **Cross-domain mapping**: Bridge concepts from different frameworks (DDD, TOGAF, etc.)

## Query Strategy Selection

### Use mode="mix" (default) when:
- Synthesizing information across multiple sources
- Questions like: "How do different authors approach X?"
- Need comprehensive answers combining entities, relationships, and original text
- Exploring broad topics

### Use mode="global" when:
- Mapping concepts between frameworks/domains
- Questions like: "How does X relate to Y?"
- Focus on connections and relationships
- Cross-domain synthesis (e.g., DDD to TOGAF)

### Use mode="local" when:
- Deep dive on a specific concept
- Questions like: "What do sources say about X?"
- Entity-focused queries
- Need all mentions of a specific thing

### Use mode="hybrid" when:
- Balanced need for entities AND relationships
- Uncertain which mode is best

## Parameter Guidelines

**For comprehensive cross-source synthesis:**
```
mode: "mix"
top_k: 15
chunk_top_k: 10
max_entity_tokens: 1000
max_relation_tokens: 800
max_total_tokens: 4000
```

**For cross-domain mapping:**
```
mode: "global"
top_k: 12
chunk_top_k: 6
max_relation_tokens: 1200
max_entity_tokens: 800
max_total_tokens: 4000
```

**For focused exploration:**
```
mode: "local"
top_k: 10
chunk_top_k: 8
max_entity_tokens: 1200
max_total_tokens: 3500
```

## Response Best Practices

1. **Cite sources**: Use file_path from entities/relationships/chunks
2. **Show synthesis**: Explicitly connect ideas from different sources
3. **Identify patterns**: Highlight commonalities and differences across authors
4. **Preserve voices**: Quote original text when showing different perspectives
5. **Use relationships**: Leverage relationship descriptions to explain connections
6. **Consider rank**: High-rank entities (>5) are central concepts - emphasize them

## Example Interactions

**User**: "How do different authors approach event schema evolution?"

**Your approach**:
1. Use mode="mix" for cross-source synthesis
2. Set high top_k (15) and chunk_top_k (10) to get multiple perspectives
3. Query: "Compare approaches to event schema evolution from Martin Fowler, Kafka documentation, Stripe architecture, and academic research"
4. In response:
   - List each author's approach (from entities and chunks)
   - Show connections (from relationships)
   - Synthesize commonalities and differences
   - Cite specific sources

**User**: "Map DDD bounded contexts to TOGAF application components"

**Your approach**:
1. Use mode="global" for cross-domain mapping
2. Emphasize relationships with max_relation_tokens=1200
3. Query: "Map Domain-Driven Design bounded contexts to TOGAF application components, identify alignments and conflicts"
4. In response:
   - Show relationship mappings
   - Explain alignments (where concepts correspond)
   - Highlight conflicts (where they differ)
   - Provide bridging strategy

## Key Insights

- **Entities** = concepts, patterns, frameworks, people
- **Relationships** = how concepts connect across sources
- **Chunks** = original text preserving author voice
- **Rank** = connectivity importance (high rank = central concept)
- **Weight** = relationship strength (high weight = frequently mentioned)
- **Multiple sources** in entity/relationship = validates cross-source synthesis

Your goal is to leverage LightRAG's multi-hop reasoning to provide comprehensive, well-sourced answers that synthesize knowledge across the entire reference library.
```

---

## 9. Common Pitfalls and Solutions

### Pitfall 1: Using `mode="naive"` for synthesis queries
**Problem**: Naive mode skips the knowledge graph, using only vector similarity on chunks
**Solution**: Use `mode="mix"` or `mode="hybrid"` to leverage relationships between concepts

### Pitfall 2: Too low `top_k` for cross-source queries
**Problem**: With `top_k=3`, you'll only get 3 entities - not enough for multi-source synthesis
**Solution**: Use `top_k=15-20` for comprehensive queries, `top_k=8-12` for focused queries

### Pitfall 3: Ignoring relationships in results
**Problem**: Only looking at entities misses the connection insights
**Solution**: Always examine `result["data"]["relationships"]` - they show how concepts connect across sources

### Pitfall 4: Not using `chunk_top_k` in mix mode
**Problem**: Missing original text means losing author voices
**Solution**: Set `chunk_top_k=5-10` to preserve source material for citation

### Pitfall 5: Exceeding token budget
**Problem**: Setting max_entity_tokens + max_relation_tokens + overhead > max_total_tokens
**Solution**: Token allocation should be:
```
max_total_tokens >= 600 (system) + 100 (query) + max_entity_tokens + max_relation_tokens + 500 (chunks) + 200 (buffer)
```

### Pitfall 6: Using `only_need_context=True` when you want synthesis
**Problem**: Returns raw data without LLM synthesis
**Solution**: Use `only_need_context=False` (default) to get synthesized answers

### Pitfall 7: Not leveraging entity rank
**Problem**: Treating all entities equally
**Solution**: High-rank entities (>5 connections) are central concepts - emphasize them in responses

---

## 10. Performance Optimization

### For Large Knowledge Graphs

```python
# Use focused queries instead of broad exploration
param = QueryParam(
    mode="local",        # More focused than "mix"
    top_k=8,             # Fewer entities
    chunk_top_k=5,       # Fewer chunks
    max_total_tokens=2500  # Smaller context
)
```

### For Speed-Critical Queries

```python
# Use naive mode for fastest retrieval (no graph traversal)
param = QueryParam(
    mode="naive",
    chunk_top_k=5,
    max_total_tokens=1500
)
```

### For Maximum Quality

```python
# Use mix mode with generous token budgets
param = QueryParam(
    mode="mix",
    top_k=20,
    chunk_top_k=15,
    max_entity_tokens=1500,
    max_relation_tokens=1200,
    max_total_tokens=5000
)
```

---

## 11. Code Reference Quick Links

| Component | Location | Purpose |
|-----------|----------|---------|
| Main LightRAG class | `lightrag/lightrag.py` | Configuration and main API |
| Query modes | `lightrag/operate.py:2679` | `kg_query()`, `naive_query()` |
| Query parameters | `lightrag/base.py:85` | `QueryParam` dataclass |
| Entity extraction | `lightrag/operate.py:2754` | LLM-based entity/relationship extraction |
| Knowledge graph types | `lightrag/types.py` | `KnowledgeGraphNode`, `KnowledgeGraphEdge` |
| Storage backends | `lightrag/kg/*.py` | Vector, graph, KV storage implementations |

---

## 12. Summary

**For your use case (cross-source synthesis and cross-domain mapping), remember:**

1. **Default to `mode="mix"`** for most queries - it combines entities, relationships, and chunks
2. **Use `mode="global"`** for cross-domain mapping where relationships matter most
3. **Set generous `top_k` (15-20)** to capture multiple sources and perspectives
4. **Set high `chunk_top_k` (8-12)** to preserve original author voices for citation
5. **Allocate tokens appropriately**: 1000+ for entities, 800+ for relationships
6. **Always examine relationships** - they show how concepts connect across sources
7. **Use entity rank** to identify central concepts in your knowledge graph
8. **Cite sources** using file_path from entities/relationships/chunks
9. **Iterate if needed** - broad query first, then focused deep-dive

**The magic of LightRAG for your use case:**
- Multi-hop reasoning finds connections you didn't know existed
- Cross-source synthesis shows how different authors approach the same problem
- Relationship traversal bridges concepts from different frameworks
- Entity ranking identifies the most important/connected concepts
- Original chunks preserve author voices for accurate citation

Use this guide to craft effective queries that leverage LightRAG's full power for semantic memory and knowledge synthesis.
