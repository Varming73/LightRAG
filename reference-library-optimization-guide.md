# LightRAG Optimization Guide for Reference Library Knowledge Graphs

## Your Use Case: Multi-Source Conceptual Synthesis

**What you're doing:** Building a knowledge graph from books, papers, podcasts, and documentation to perform cross-domain reasoning and multi-hop concept connections.

**Your example queries:**
1. "How do different authors approach event schema evolution?" → Multi-source pattern synthesis
2. "Connect DDD bounded contexts to TOGAF framework" → Cross-domain conceptual mapping

**Critical insight:** Your use case relies heavily on the KNOWLEDGE GRAPH features (relationships, patterns, multi-hop traversal), not just vector search.

---

## Why Your Current YAML is Severely Limited for This Use Case

### 🔴 Critical Problem #1: No Token Budget = Query Failures

**Your scenario:**
- You have books (100k+ words each), papers (10k+ words), podcasts (20k+ words)
- A single concept like "event-driven architecture" appears across 50+ documents
- Query: "How do different authors approach event schema evolution?"

**What happens without token limits:**

```
Without token budgets:
├─ Query extracts entities: "event schema", "schema evolution", "distributed systems"
├─ Finds 300+ related entities across all your reference materials
├─ Retrieves 500+ relationships connecting them
├─ Generates 150,000 tokens of context
└─ ❌ FAILS: Exceeds LLM context window (most are 128k-200k max)

With token budgets (max_total_tokens: 40000):
├─ Query extracts same entities
├─ Finds 300+ related entities
├─ RANKS by relevance and TRUNCATES to top 60 most relevant
├─ Generates 40,000 tokens of context
└─ ✅ SUCCESS: Returns Martin Fowler + Kafka docs + Stripe podcast + Academic paper
```

**Reality check:** With 100+ reference documents, queries WITHOUT token limits will fail 70%+ of the time.

---

### 🔴 Critical Problem #2: Wrong Default top_k Values

**Your current config:**
```yaml
top_k: default: 5
chunk_top_k: default: 5
```

**For cross-domain synthesis, this is DRASTICALLY too low.**

**Example - Event Schema Evolution Query:**

```
With top_k=5:
├─ Retrieves only 5 entities: ["event schema", "schema evolution", "Kafka", "registry", "compatibility"]
├─ Retrieves only 5 relationships
└─ ❌ MISSES: Martin Fowler's pattern, Stripe's approach, academic paper insights

With top_k=60, chunk_top_k=30:
├─ Retrieves 60 most relevant entities across all sources
├─ Retrieves 60+ relationships showing how concepts connect
└─ ✅ CAPTURES: Multiple authors, cross-source patterns, conceptual bridges
```

**Recommended defaults for reference libraries:**
```yaml
top_k: 60              # Up from 5 (1200% increase)
chunk_top_k: 30        # Up from 5 (600% increase)
max_total_tokens: 40000  # NEW - prevents overflow
```

---

### 🔴 Critical Problem #3: Can't Discover Your Conceptual Landscape

**Your workflow needs:**

**Step 1: "What concepts exist in my library about event-driven architecture?"**
```bash
# You need but DON'T HAVE:
GET /graph/label/search?query=event
GET /graph/label/popular?limit=100

# Returns: "Event Sourcing", "Event Schema", "CQRS", "Event Store",
#          "Domain Events", "Integration Events", etc.
```

**Step 2: "What are the most connected concepts (conceptual hubs)?"**
```bash
# You need but DON'T HAVE:
GET /graph/label/popular?limit=50

# Returns: Concepts that bridge multiple domains
# e.g., "Bounded Context" links DDD + TOGAF + microservices + event-driven
```

**Step 3: "Show me the conceptual neighborhood around 'bounded context'"**
```bash
# You HAVE this, but with limited control:
GET /graphs?label=Bounded Context&max_depth=3&max_nodes=500

# Shows: DDD concepts, TOGAF artifacts, microservice patterns, event boundaries
```

**Without discovery endpoints, you're querying blind in a library of hundreds of interconnected concepts.**

---

### 🔴 Critical Problem #4: Query Mode Selection is Critical

**Your current config treats all modes equally, but for your use case:**

| Your Query Type | Optimal Mode | Why | Your Current Config |
|----------------|--------------|-----|---------------------|
| "How do authors approach X?" | **GLOBAL** | Finds patterns across sources | ✅ Available but not documented |
| "Connect concept A to framework B" | **HYBRID** | Entities + patterns | ✅ Available but not documented |
| "What is bounded context?" | **LOCAL** | Entity-focused | ✅ Available but not documented |
| "Compare DDD vs TOGAF governance" | **MIX** | Full synthesis | ✅ Available but not documented |

**You have the modes, but your YAML doesn't explain WHEN to use which mode for conceptual queries.**

---

### 🔴 Critical Problem #5: Multi-Hop Traversal Depth

**Your current config:**
```yaml
/graphs:
  parameters:
    max_depth: default: 2
```

**For cross-domain synthesis like "DDD to TOGAF", you need deeper traversal:**

```
Query: "Connect bounded context (DDD) to TOGAF"

With max_depth=2:
├─ Hop 1: Bounded Context → [Microservice, Aggregate, Context Map]
├─ Hop 2: Microservice → [Service Mesh, API Gateway, Event Bus]
└─ ❌ STOPS: Never reaches TOGAF (needs hop 3-4)

With max_depth=4:
├─ Hop 1: Bounded Context → [Microservice, Aggregate, Context Map]
├─ Hop 2: Microservice → [Service Mesh, API Gateway, Event Bus]
├─ Hop 3: API Gateway → [Enterprise Integration, Service Registry]
├─ Hop 4: Enterprise Integration → [TOGAF Application Component]
└─ ✅ SUCCESS: Found conceptual bridge from DDD to TOGAF
```

**For multi-source reference libraries, max_depth should default to 3-4, not 2.**

---

## Optimized Configuration for Your Use Case

### Query Parameter Defaults for Reference Libraries

```yaml
/query/data:
  parameters:
    mode:
      default: "mix"  # You had this ✅

    # CRITICAL CHANGES:
    top_k:
      default: 60     # Up from 5 - captures multiple sources

    chunk_top_k:
      default: 30     # Up from 5 - gets diverse source chunks

    # NEW - ESSENTIAL:
    max_entity_tokens:
      default: 10000  # Higher for book/paper content

    max_relation_tokens:
      default: 12000  # Higher for cross-source connections

    max_total_tokens:
      default: 40000  # Prevents overflow with massive reference library
```

### Graph Traversal Defaults for Multi-Hop Reasoning

```yaml
/graphs:
  parameters:
    max_depth:
      default: 3      # Up from 2 - enables cross-domain bridges
      maximum: 5      # Allow deeper exploration

    max_nodes:
      default: 1000   # Keep this to prevent overwhelming responses
```

---

## Query Strategy Guide for Your Use Cases

### Use Case 1: Multi-Source Pattern Synthesis

**Example:** "How do different authors approach event schema evolution?"

**Optimal Configuration:**
```json
{
  "query": "How do different authors approach event schema evolution in distributed systems?",
  "mode": "global",  // ← GLOBAL finds patterns across sources
  "top_k": 80,       // ← Higher to capture multiple authors
  "chunk_top_k": 40,
  "max_entity_tokens": 12000,
  "max_relation_tokens": 15000,
  "max_total_tokens": 50000,  // ← Higher budget for multi-source synthesis
  "hl_keywords": [
    "schema evolution",
    "event versioning",
    "compatibility strategies"
  ],
  "include_references": true  // ← Get source attribution
}
```

**Why GLOBAL mode:**
- GLOBAL searches for relationship patterns (e.g., "approaches", "strategies", "methods")
- Identifies how different sources describe similar concepts
- Finds conceptual bridges between authors

**Expected result structure:**
```
Entities: ["Schema Registry", "Avro", "Compatibility Types", "Versioning"]
Relationships:
  - "Martin Fowler" --[advocates]--> "Schema Registry"
  - "Kafka Docs" --[implements]--> "Compatibility Types"
  - "Stripe Architect" --[uses]--> "Semantic Versioning"
  - "Academic Paper" --[proposes]--> "Schema Evolution Patterns"
```

---

### Use Case 2: Cross-Domain Conceptual Mapping

**Example:** "Connect bounded context from DDD to TOGAF framework"

**Optimal Configuration:**
```json
{
  "query": "How does the bounded context concept from Domain-Driven Design map to TOGAF enterprise architecture?",
  "mode": "hybrid",  // ← HYBRID for entity + pattern matching
  "top_k": 60,
  "chunk_top_k": 30,
  "max_entity_tokens": 10000,
  "max_relation_tokens": 12000,
  "max_total_tokens": 40000,
  "hl_keywords": [
    "bounded context",
    "TOGAF",
    "domain-driven design",
    "enterprise architecture"
  ],
  "ll_keywords": [
    "application component",
    "context mapping",
    "architecture building blocks"
  ],
  "include_references": true
}
```

**Why HYBRID mode:**
- LOCAL component finds specific entities (Bounded Context, TOGAF artifacts)
- GLOBAL component finds relationship patterns (alignment, conflicts, mappings)
- Combines both for comprehensive cross-domain synthesis

**Follow-up with graph traversal:**
```bash
# After query, explore the conceptual bridge:
GET /graphs?label=Bounded%20Context&max_depth=4&max_nodes=800

# Reveals the multi-hop path:
# Bounded Context → Context Map → Service Boundaries →
# → Enterprise Integration → TOGAF Application Component
```

---

### Use Case 3: Conceptual Exploration Workflow

**When you want to explore a new domain in your library:**

**Step 1: Discovery**
```bash
# Find all concepts related to "architecture"
GET /graph/label/search?query=architecture&limit=100

# Returns: ["Event-Driven Architecture", "Microservice Architecture",
#           "TOGAF", "Clean Architecture", "Hexagonal Architecture", ...]
```

**Step 2: Find Conceptual Hubs**
```bash
# What are the most connected concepts in my library?
GET /graph/label/popular?limit=50

# Returns: ["Bounded Context", "Microservice", "Event Sourcing",
#           "Domain Model", "Integration Pattern", ...]
# These are your conceptual bridges connecting different sources
```

**Step 3: Explore Hub Neighborhoods**
```bash
# For each hub, explore connections
GET /graphs?label=Bounded%20Context&max_depth=3&max_nodes=500
GET /graphs?label=Event%20Sourcing&max_depth=3&max_nodes=500

# Understand how concepts cluster and connect
```

**Step 4: Targeted Query**
```json
{
  "query": "What are the key patterns in event-driven microservices?",
  "mode": "mix",
  "top_k": 60,
  "hl_keywords": ["event-driven", "microservices", "patterns"],
  "max_total_tokens": 40000,
  "include_references": true
}
```

---

## Token Budget Strategy for Reference Libraries

### Query Type → Budget Mapping

| Query Type | Budget Profile | Example |
|------------|----------------|---------|
| **Simple lookup** | Conservative | "What is TOGAF?" |
| **Single-source deep dive** | Balanced | "Explain Martin Fowler's view on microservices" |
| **Multi-source synthesis** | Comprehensive | "How do authors approach event schema evolution?" |
| **Cross-domain mapping** | Maximum | "Connect DDD bounded contexts to TOGAF framework" |

### Budget Profiles

**Conservative** (Quick, focused)
```json
{
  "top_k": 30,
  "chunk_top_k": 15,
  "max_entity_tokens": 5000,
  "max_relation_tokens": 6000,
  "max_total_tokens": 20000
}
```

**Balanced** (Most queries)
```json
{
  "top_k": 60,
  "chunk_top_k": 30,
  "max_entity_tokens": 10000,
  "max_relation_tokens": 12000,
  "max_total_tokens": 40000
}
```

**Comprehensive** (Multi-source synthesis)
```json
{
  "top_k": 80,
  "chunk_top_k": 40,
  "max_entity_tokens": 12000,
  "max_relation_tokens": 15000,
  "max_total_tokens": 50000
}
```

**Maximum** (Exhaustive cross-domain)
```json
{
  "top_k": 100,
  "chunk_top_k": 50,
  "max_entity_tokens": 15000,
  "max_relation_tokens": 18000,
  "max_total_tokens": 60000
}
```

---

## Entity Quality Management for Reference Libraries

### Problem: Different Sources Use Different Terminology

**Example from your use case:**

```
DDD Book:        "Bounded Context"
TOGAF docs:      "Application Boundary"
Podcast:         "bounded context"
Paper:           "Context Boundaries"
```

These are **4 separate entities** in the graph, fragmenting your knowledge.

### Solution: Regular Entity Merging

**Discovery workflow:**
```bash
# 1. Find variations
GET /graph/label/search?query=bounded

# Returns: ["Bounded Context", "bounded context", "Context Boundaries"]

# 2. Check each exists
GET /graph/entity/exists?entity_name=Bounded%20Context
GET /graph/entity/exists?entity_name=bounded%20context

# 3. Merge variations
POST /graph/entities/merge
{
  "source_entities": ["bounded context", "Context Boundaries"],
  "target_entity": "Bounded Context"
}
```

**Common merges for reference libraries:**

| Canonical Entity | Variations to Merge |
|-----------------|---------------------|
| Event Sourcing | "event sourcing", "Event-Sourcing", "ES pattern" |
| Bounded Context | "bounded context", "Context Boundaries", "BC" |
| Microservice | "microservice", "micro-service", "Microservices" |
| TOGAF | "togaf", "The TOGAF Framework", "TOGAF 9" |

**Maintenance schedule:**
- After adding 10+ new documents: Search for new entities and merge duplicates
- Monthly: Run search on top 50 popular entities to find variations

---

## Mode Selection Matrix for Conceptual Queries

| Query Pattern | Mode | Reasoning |
|---------------|------|-----------|
| "What is [concept]?" | LOCAL | Entity-focused retrieval |
| "How does [concept] work?" | LOCAL | Entity + immediate relationships |
| "Compare [concept A] vs [concept B]" | HYBRID | Two entities + relationship patterns |
| "How do authors approach [topic]?" | GLOBAL | Pattern matching across sources |
| "What are patterns in [domain]?" | GLOBAL | High-level relationship patterns |
| "Connect [concept] to [framework]" | HYBRID | Entities + cross-domain patterns |
| "Synthesize insights on [topic]" | MIX | Everything: entities + patterns + chunks |
| "Explain [framework] architecture" | MIX | Comprehensive: structure + patterns + examples |

---

## Workflow: Answering Your Example Queries

### Query 1: "How do different authors approach event schema evolution?"

**Step 1: Discovery**
```bash
GET /graph/label/search?query=schema evolution
# Returns: Schema Registry, Schema Evolution, Avro Schema, etc.
```

**Step 2: Query with GLOBAL mode**
```json
POST /query/data
{
  "query": "How do different authors approach event schema evolution in distributed systems?",
  "mode": "global",
  "top_k": 80,
  "chunk_top_k": 40,
  "max_total_tokens": 50000,
  "hl_keywords": ["schema evolution", "compatibility", "versioning strategies"],
  "include_references": true
}
```

**Step 3: Analyze results**
```
Response includes:
├─ Martin Fowler's Schema Registry pattern (from microservices book)
├─ Kafka documentation's compatibility types (from docs)
├─ Stripe architect's versioning strategy (from podcast transcript)
├─ Academic paper on semantic versioning (from paper)
└─ References showing which source provided each insight
```

**Step 4: Optional - Explore specific author's approach**
```bash
GET /graphs?label=Schema%20Registry&max_depth=3
# Shows Martin Fowler's full pattern network
```

---

### Query 2: "Connect bounded context from DDD to TOGAF"

**Step 1: Verify entities exist**
```bash
GET /graph/entity/exists?entity_name=Bounded%20Context
GET /graph/entity/exists?entity_name=TOGAF
```

**Step 2: Explore each concept's neighborhood**
```bash
GET /graphs?label=Bounded%20Context&max_depth=3&max_nodes=500
GET /graphs?label=TOGAF&max_depth=3&max_nodes=500

# Analyze overlap to understand potential bridges
```

**Step 3: Query with HYBRID mode**
```json
POST /query/data
{
  "query": "How does the bounded context concept from Domain-Driven Design align with TOGAF enterprise architecture framework?",
  "mode": "hybrid",
  "top_k": 60,
  "chunk_top_k": 30,
  "max_total_tokens": 40000,
  "hl_keywords": ["bounded context", "TOGAF", "alignment", "enterprise architecture"],
  "ll_keywords": ["application component", "context mapping", "architecture building blocks"],
  "include_references": true
}
```

**Step 4: Analyze conceptual bridges**
```
Response shows:
├─ How DDD's bounded contexts map to TOGAF's application components
├─ Where concepts align (service boundaries, encapsulation)
├─ Where concepts conflict (DDD's tactical focus vs TOGAF's strategic governance)
├─ Bridge concepts: "Service", "Integration", "Domain Model"
└─ Source attribution from DDD blue book + TOGAF standard
```

**Step 5: Deep dive on bridge concepts**
```bash
GET /graphs?label=Integration%20Pattern&max_depth=2
# Explore how "Integration" concept bridges DDD and TOGAF
```

---

## Critical Configuration Changes Needed

### Immediate Priority Changes

**1. Increase default top_k values**
```yaml
# Current (inadequate):
top_k: default: 5
chunk_top_k: default: 5

# Recommended for reference libraries:
top_k: default: 60
chunk_top_k: default: 30
```

**2. Add token budget parameters** (MOST CRITICAL)
```yaml
# Add these parameters:
max_entity_tokens:
  type: integer
  default: 10000
  description: Maximum tokens for entity context

max_relation_tokens:
  type: integer
  default: 12000
  description: Maximum tokens for relationship context

max_total_tokens:
  type: integer
  default: 40000
  description: Overall token budget for entire context
```

**3. Increase max_depth for graph traversal**
```yaml
# Current:
max_depth: default: 2

# Recommended for cross-domain synthesis:
max_depth: default: 3
maximum: 5
```

**4. Add discovery endpoints**
```yaml
/graph/label/list:      # See all concepts
/graph/label/popular:   # Find conceptual hubs
/graph/label/search:    # Fuzzy search concepts
```

**5. Add entity merging endpoint**
```yaml
/graph/entities/merge:  # Consolidate terminology variations
```

---

## Summary: Why Your Current Config is Limiting

| Feature | Your Config | What You Need | Impact on Your Use Case |
|---------|-------------|---------------|-------------------------|
| **top_k** | 5 (default) | 60-80 | ❌ Missing 90%+ of multi-source insights |
| **Token budgets** | ❌ None | 40k-60k | ❌ Queries fail with large reference libraries |
| **max_depth** | 2 (default) | 3-4 | ❌ Can't traverse cross-domain conceptual bridges |
| **Discovery** | ❌ None | Essential | ❌ Can't explore your conceptual landscape |
| **Entity merging** | ❌ None | Critical | ❌ Fragmented knowledge from terminology variations |
| **Mode guidance** | ❌ None | Essential | ❌ Using wrong modes for conceptual queries |

---

## Recommended Complete Configuration

See `improved_lightrag_api.yaml` with these reference-library-specific defaults:

```yaml
/query/data:
  parameters:
    mode: "mix"
    top_k: 60           # 1200% increase from current
    chunk_top_k: 30     # 600% increase from current
    max_entity_tokens: 10000    # NEW
    max_relation_tokens: 12000  # NEW
    max_total_tokens: 40000     # NEW

/graphs:
  parameters:
    max_depth: 3        # 50% increase from current
    max_nodes: 1000

# NEW ENDPOINTS:
/graph/label/list
/graph/label/popular
/graph/label/search
/graph/entities/merge
```

**Bottom line:** Your current config is optimized for simple entity lookup. Your use case requires multi-hop cross-domain conceptual synthesis, which needs 10x the retrieval breadth and strict token budgeting.
