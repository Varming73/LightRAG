# LightRAG API Configuration Comparison

## Executive Summary

**Your current YAML exposes ~20% of LightRAG's knowledge graph capabilities.**

The most critical missing features for diverse subjects are:
1. **Token budget controls** - Prevents context overflow
2. **Entity discovery endpoints** - Explore what's in your KB
3. **Entity/relationship management** - Maintain quality

---

## Side-by-Side Comparison

### Current YAML Configuration

```yaml
paths:
  /query/data:
    post:
      parameters:
        - query (required)
        - mode (5 options)
        - top_k (default: 5)
        - chunk_top_k (default: 5)

  /graphs:
    get:
      parameters:
        - label (required)
        - max_depth (default: 2)
```

**Exposed Features:**
- ✅ Basic query with mode selection
- ✅ Basic graph traversal from a known entity
- ❌ No token budget control
- ❌ No entity discovery
- ❌ No quality management
- ❌ No citations/references
- ❌ No streaming
- ❌ No manual keyword control

---

### Improved YAML Configuration

```yaml
paths:
  # QUERY ENDPOINTS (3 endpoints)
  /query/data:         # Data retrieval without LLM
  /query:              # Full RAG with LLM response
  /query/stream:       # Streaming responses

  # DISCOVERY ENDPOINTS (3 endpoints) - NEW!
  /graph/label/list:      # List all entities
  /graph/label/popular:   # Most connected entities
  /graph/label/search:    # Fuzzy search entities

  # GRAPH EXPLORATION (1 endpoint - enhanced)
  /graphs:             # Subgraph with better controls

  # QUALITY MANAGEMENT (6 endpoints) - NEW!
  /graph/entity/exists:   # Check entity
  /graph/entity/create:   # Add entity
  /graph/entity/edit:     # Update entity
  /graph/entities/merge:  # Consolidate duplicates
  /graph/relation/create: # Add relationship
  /graph/relation/edit:   # Update relationship
```

**Exposed Features:**
- ✅ Query with 15+ parameters
- ✅ Token budget controls
- ✅ Entity discovery (3 methods)
- ✅ Graph traversal with limits
- ✅ Full entity/relationship CRUD
- ✅ Citations and references
- ✅ Streaming support
- ✅ Manual keyword override

---

## Query Parameter Comparison

### Your Current /query/data

```json
{
  "query": "string",
  "mode": "mix|hybrid|naive|local|global",
  "top_k": 5,
  "chunk_top_k": 5
}
```

**4 parameters total**

---

### Improved /query/data

```json
{
  // Basic Parameters (you have these)
  "query": "string",
  "mode": "mix|hybrid|naive|local|global",
  "top_k": 40,
  "chunk_top_k": 20,

  // Token Budget Controls (NEW - CRITICAL!)
  "max_entity_tokens": 6000,
  "max_relation_tokens": 8000,
  "max_total_tokens": 30000,

  // Advanced Options (NEW)
  "enable_rerank": true,
  "include_references": true,

  // Manual Control (NEW)
  "hl_keywords": ["high", "level"],
  "ll_keywords": ["low", "level"]
}
```

**11 parameters total** (175% more control)

---

## Feature Impact Analysis for Diverse Subjects

### 🔴 CRITICAL Missing Features (High Impact)

#### 1. Token Budget Controls

**Missing Parameters:**
- `max_entity_tokens`
- `max_relation_tokens`
- `max_total_tokens`

**Impact:**
- ❌ Queries can overflow context window unpredictably
- ❌ Can't control retrieval size for different query types
- ❌ May hit LLM token limits with diverse subjects
- ❌ Unpredictable costs

**Example Problem:**
```
Query: "What are machine learning applications?"

Without token limits:
- Retrieves 500 entities (medical ML + vision ML + NLP ML + ...)
- Generates 80,000 tokens of context
- Exceeds most LLM context windows
- Query fails or gets truncated randomly

With token limits (max_total_tokens: 30000):
- Retrieves top 40 most relevant entities
- Generates 30,000 tokens max
- Fits in context window
- Query succeeds with best results
```

---

#### 2. Entity Discovery Endpoints

**Missing Endpoints:**
- `/graph/label/list`
- `/graph/label/popular`
- `/graph/label/search`

**Impact:**
- ❌ No way to see what's in your knowledge base
- ❌ Can't find entity names for graph queries
- ❌ Must guess exact entity names
- ❌ Can't identify knowledge gaps

**Example Problem:**
```
You want to query about "neural networks" but don't know if:
- It's stored as "Neural Networks" or "neural networks"
- Related entities exist (CNN, RNN, Transformers, etc.)
- It's a hub connecting to other domains

Without discovery: Trial and error, guessing names
With discovery: Search returns exact names and connections
```

---

#### 3. Entity/Relationship Management

**Missing Endpoints:**
- `/graph/entity/create`
- `/graph/entity/edit`
- `/graph/entities/merge`
- `/graph/relation/create`
- `/graph/relation/edit`

**Impact:**
- ❌ Can't fix extraction errors
- ❌ Can't merge duplicate entities
- ❌ Can't add missing connections
- ❌ Knowledge graph quality degrades over time

**Example Problem:**
```
Your diverse documents mention:
- "Covid-19" (100 times)
- "COVID-19" (75 times)
- "coronavirus" (50 times)
- "SARS-CoV-2" (25 times)

Without merging:
- 4 separate entities with split information
- Queries miss relevant results
- Graph is fragmented

With merging:
- Single "COVID-19" entity with all information
- Comprehensive query results
- Clean, usable graph
```

---

### 🟡 MEDIUM Missing Features (Moderate Impact)

#### 4. Include References

**Missing Parameter:**
- `include_references`

**Impact:**
- ❌ Can't trace facts to source documents
- ❌ Difficult to verify information
- ❌ No way to provide citations

**Use Case:**
With diverse subjects, you want to verify which document stated a fact.

---

#### 5. Manual Keyword Control

**Missing Parameters:**
- `hl_keywords` (high-level keywords)
- `ll_keywords` (low-level keywords)

**Impact:**
- ❌ Relying entirely on automatic keyword extraction
- ❌ Can't guide retrieval when automatic extraction fails
- ❌ Less control over what gets retrieved

**Use Case:**
When query "transformer" should retrieve ML transformers, not electrical transformers.

---

#### 6. Streaming Support

**Missing Endpoint:**
- `/query/stream`

**Impact:**
- ❌ Must wait for entire response
- ❌ Worse user experience for long queries
- ❌ Can't show progressive results

---

### 🟢 LOW Missing Features (Nice to Have)

#### 7. Reranking Control

**Missing Parameter:**
- `enable_rerank`

**Impact:** Minor - reranking is on by default, rarely need to disable

---

#### 8. Response Type Hints

**Missing Parameter:**
- `response_type`

**Impact:** Minor - LLM adapts automatically

---

## Real-World Example: Diverse Subject Query

### Scenario
Knowledge base contains:
- Medical device documentation
- Machine learning papers
- Python programming tutorials
- Business process documents

### Query
"How can machine learning improve product quality?"

---

### With Your Current Config

```json
{
  "query": "How can machine learning improve product quality?",
  "mode": "mix",
  "top_k": 5,
  "chunk_top_k": 5
}
```

**What Happens:**
1. ❌ No token budget → May retrieve 10,000+ entities across all domains
2. ❌ Only 5 entities/chunks → Misses relevant information
3. ❌ No way to check what entities exist beforehand
4. ❌ No way to focus on specific domains
5. ❌ No citations → Can't verify which documents provided info
6. ❌ If "Machine Learning" entity is duplicated as "ML", "machine learning", etc. → Fragmented results

**Result:** Incomplete, potentially overwhelming, unverifiable answer

---

### With Improved Config

**Step 1: Discovery**
```bash
# Find relevant entities
GET /graph/label/search?query=machine learning
GET /graph/label/search?query=quality control
GET /graph/label/popular?limit=50
```

**Step 2: Optimized Query**
```json
{
  "query": "How can machine learning improve product quality?",
  "mode": "mix",
  "top_k": 40,
  "chunk_top_k": 20,
  "max_entity_tokens": 8000,
  "max_relation_tokens": 10000,
  "max_total_tokens": 35000,
  "hl_keywords": [
    "machine learning",
    "quality control",
    "product quality"
  ],
  "include_references": true,
  "enable_rerank": true
}
```

**What Happens:**
1. ✅ Token budget prevents overflow, retrieves top 40 most relevant
2. ✅ Manual keywords focus on ML + quality domains
3. ✅ Reranking ensures best chunks rise to top
4. ✅ References show which documents provided each fact
5. ✅ Sufficient top_k (40) captures diverse relevant entities

**Result:** Comprehensive, focused, verifiable answer within token budget

**Step 3: Quality Maintenance** (if needed)
```bash
# If you notice duplicate entities
POST /graph/entities/merge
{
  "source_entities": ["ML", "machine learning"],
  "target_entity": "Machine Learning"
}
```

---

## Token Budget Strategy Table

| Query Type | Your Config | Improved Config | Why? |
|------------|-------------|-----------------|------|
| **Simple entity lookup** | No limit (risky) | 15k tokens | Fast, focused |
| **Cross-domain query** | No limit (risky) | 35k tokens | Needs more context |
| **Comprehensive research** | No limit (fails) | 50k tokens | Maximum context |
| **Quick check** | No limit (risky) | 10k tokens | Just the essentials |

**Your Current Approach:** One size fits all, no control
**Improved Approach:** Tune budget to query complexity

---

## Recommended Default Parameters for Diverse Subjects

```json
{
  "mode": "mix",
  "top_k": 40,
  "chunk_top_k": 20,
  "max_entity_tokens": 6000,
  "max_relation_tokens": 8000,
  "max_total_tokens": 30000,
  "include_references": true,
  "enable_rerank": true
}
```

**This provides:**
- ✅ Comprehensive retrieval (top_k: 40)
- ✅ Safe token limits (won't overflow)
- ✅ Citations for verification
- ✅ Quality reranking
- ✅ Works across diverse subjects

---

## Migration Path

### Phase 1: Add Token Controls (Immediate)
Update your `/query/data` endpoint to include:
- `max_entity_tokens`
- `max_relation_tokens`
- `max_total_tokens`

**Impact:** Prevents context overflow, makes queries reliable

---

### Phase 2: Add Discovery (High Priority)
Add endpoints:
- `/graph/label/list`
- `/graph/label/popular`
- `/graph/label/search`

**Impact:** Enables exploration of your diverse knowledge base

---

### Phase 3: Add Quality Management (Medium Priority)
Add endpoints:
- `/graph/entities/merge`
- `/graph/entity/create`
- `/graph/entity/edit`

**Impact:** Maintain and improve knowledge graph quality

---

### Phase 4: Add Advanced Features (Nice to Have)
Add:
- `/query/stream` endpoint
- `include_references` parameter
- Manual keyword parameters

**Impact:** Better UX and control

---

## Bottom Line

| Metric | Current YAML | Improved YAML | Improvement |
|--------|--------------|---------------|-------------|
| **Endpoints** | 2 | 13 | +550% |
| **Query parameters** | 4 | 11 | +175% |
| **Token control** | ❌ None | ✅ Full | Infinite |
| **Discovery capability** | ❌ None | ✅ 3 methods | N/A |
| **Quality management** | ❌ None | ✅ 6 operations | N/A |
| **Suitability for diverse subjects** | ⚠️ Limited | ✅ Excellent | Major |

**Recommendation:** Migrate to the improved configuration, prioritizing token controls and discovery endpoints first.
