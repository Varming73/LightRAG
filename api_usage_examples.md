# LightRAG API Usage Examples for Diverse Subjects

## Overview
These examples demonstrate how to leverage the full power of LightRAG's knowledge graph features for a knowledge base with very diverse subjects.

---

## 1. Discovery Workflow (Start Here!)

### A. Find What's in Your Knowledge Base

```bash
# Get all entities
curl -X GET "https://lightrag.t51.it/graph/label/list" \
  -H "X-API-Key: YOUR_API_KEY"

# Response example:
{
  "labels": [
    "Abena Wingman",
    "Neural Networks",
    "Python",
    "Deep Learning",
    "Transformer Architecture",
    "Medical Devices",
    ...
  ]
}
```

### B. Find Most Important Entities (Knowledge Hubs)

```bash
# Get top 50 most connected entities
curl -X GET "https://lightrag.t51.it/graph/label/popular?limit=50" \
  -H "X-API-Key: YOUR_API_KEY"

# These are your "hub" concepts that bridge different subjects
```

### C. Search for Specific Entities

```bash
# Fuzzy search (useful when you're not sure of exact names)
curl -X GET "https://lightrag.t51.it/graph/label/search?query=neural&limit=20" \
  -H "X-API-Key: YOUR_API_KEY"

# Response example:
{
  "labels": [
    "Neural Networks",
    "Convolutional Neural Networks",
    "Recurrent Neural Networks",
    "Neural Architecture Search"
  ]
}
```

---

## 2. Optimized Queries for Diverse Subjects

### A. Basic Query with Token Budget (Recommended)

```bash
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the specifications of Abena Wingman?",
    "mode": "mix",
    "top_k": 40,
    "chunk_top_k": 20,
    "max_entity_tokens": 6000,
    "max_relation_tokens": 8000,
    "max_total_tokens": 30000,
    "include_references": true,
    "enable_rerank": true
  }'
```

**Response Structure:**
```json
{
  "status": "success",
  "data": {
    "entities": [
      {
        "entity_name": "Abena Wingman",
        "entity_type": "Product",
        "description": "Premium incontinence product...",
        "file_path": "/path/to/source.pdf",
        "reference_id": "ref_001"
      }
    ],
    "relationships": [
      {
        "src_id": "Abena Wingman",
        "tgt_id": "Abena",
        "description": "is manufactured by",
        "weight": 0.95
      }
    ],
    "chunks": [
      {
        "content": "The Wingman features...",
        "file_path": "/path/to/source.pdf",
        "reference_id": "ref_001"
      }
    ],
    "references": [
      {
        "reference_id": "ref_001",
        "file_path": "/path/to/source.pdf"
      }
    ]
  },
  "metadata": {
    "query_mode": "mix",
    "keywords": {
      "high_level": ["medical products", "incontinence"],
      "low_level": ["Wingman", "specifications", "Abena"]
    },
    "processing_info": {
      "total_entities_found": 127,
      "entities_after_truncation": 40,
      "total_relations_found": 234,
      "relations_after_truncation": 50,
      "final_chunks_count": 20
    }
  }
}
```

### B. Query with Manual Keyword Control

```bash
# When automatic keyword extraction isn't capturing your intent
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "Compare absorption technologies",
    "mode": "mix",
    "hl_keywords": ["medical technology", "absorbent materials"],
    "ll_keywords": ["superabsorbent polymer", "SAP", "absorption capacity"],
    "max_entity_tokens": 6000,
    "max_relation_tokens": 8000,
    "max_total_tokens": 30000
  }'
```

### C. Different Query Modes for Different Use Cases

#### Local Mode: When you know the specific entity

```bash
# Best for: "Tell me about entity X"
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Abena Wingman?",
    "mode": "local",
    "top_k": 40,
    "max_total_tokens": 20000
  }'
```

#### Global Mode: When you want patterns/trends

```bash
# Best for: "What are common patterns in X domain?"
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are trends in medical device innovation?",
    "mode": "global",
    "top_k": 60,
    "max_total_tokens": 30000
  }'
```

#### Hybrid Mode: When you want both entity details and patterns

```bash
# Best for: "Tell me about X and how it relates to broader trends"
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How does Abena Wingman compare to other products?",
    "mode": "hybrid",
    "top_k": 50,
    "max_total_tokens": 30000
  }'
```

#### Mix Mode: Best general-purpose mode

```bash
# Best for: Most queries, especially with diverse subjects
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the key features of incontinence products?",
    "mode": "mix",
    "top_k": 40,
    "chunk_top_k": 20,
    "max_total_tokens": 30000
  }'
```

---

## 3. Graph Exploration (Understanding Connections)

### A. Explore Entity Neighborhood

```bash
# Get 2-hop neighborhood around "Neural Networks"
curl -X GET "https://lightrag.t51.it/graphs?label=Neural%20Networks&max_depth=2&max_nodes=500" \
  -H "X-API-Key: YOUR_API_KEY"

# Response: Graph structure with nodes and edges
{
  "nodes": [
    {
      "id": "Neural Networks",
      "label": "Neural Networks",
      "properties": {
        "type": "Concept",
        "description": "..."
      }
    },
    {
      "id": "Deep Learning",
      "label": "Deep Learning",
      "properties": {...}
    }
  ],
  "edges": [
    {
      "source": "Neural Networks",
      "target": "Deep Learning",
      "relationship": "is a subset of",
      "properties": {
        "weight": 0.95
      }
    }
  ]
}
```

### B. Cross-Domain Discovery

```bash
# Find connections between two diverse subjects
# Step 1: Get subgraph for first entity
curl -X GET "https://lightrag.t51.it/graphs?label=Medical%20Devices&max_depth=2" \
  -H "X-API-Key: YOUR_API_KEY"

# Step 2: Get subgraph for second entity
curl -X GET "https://lightrag.t51.it/graphs?label=Machine%20Learning&max_depth=2" \
  -H "X-API-Key: YOUR_API_KEY"

# Analyze overlap in your application layer
```

---

## 4. Knowledge Graph Quality Management

### A. Check for Duplicate Entities

```bash
# Search for variations of an entity name
curl -X GET "https://lightrag.t51.it/graph/label/search?query=neural%20network" \
  -H "X-API-Key: YOUR_API_KEY"

# Result might show:
# - "Neural Network"
# - "neural network"
# - "Neural Networks"
# - "neural-network"
```

### B. Merge Duplicate Entities

```bash
# Consolidate duplicates into canonical form
curl -X POST "https://lightrag.t51.it/graph/entities/merge" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "source_entities": [
      "neural network",
      "Neural Networks",
      "neural-network"
    ],
    "target_entity": "Neural Network"
  }'
```

### C. Add Missing Entity

```bash
# If extraction missed an important entity
curl -X POST "https://lightrag.t51.it/graph/entity/create" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_name": "Transformer Architecture",
    "entity_type": "Concept",
    "description": "Neural network architecture based on self-attention mechanisms",
    "source_id": "manual_addition"
  }'
```

### D. Create Missing Relationship

```bash
# Add a relationship that should exist
curl -X POST "https://lightrag.t51.it/graph/relation/create" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "src_id": "Transformer Architecture",
    "tgt_id": "Attention Mechanism",
    "description": "is based on",
    "keywords": "self-attention, neural architecture",
    "weight": 0.95
  }'
```

---

## 5. Advanced Patterns for Diverse Subjects

### A. Multi-Subject Query Pattern

```bash
# Query touching multiple diverse subjects
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How can machine learning improve medical device quality control?",
    "mode": "mix",
    "hl_keywords": [
      "machine learning",
      "medical devices",
      "quality control"
    ],
    "top_k": 60,
    "chunk_top_k": 30,
    "max_entity_tokens": 8000,
    "max_relation_tokens": 10000,
    "max_total_tokens": 35000,
    "include_references": true
  }'
```

### B. Progressive Token Budget Strategy

```bash
# Start conservative, then expand if needed

# STEP 1: Quick retrieval with tight budget
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Abena Wingman?",
    "mode": "mix",
    "top_k": 20,
    "chunk_top_k": 10,
    "max_entity_tokens": 3000,
    "max_relation_tokens": 4000,
    "max_total_tokens": 15000
  }'

# If insufficient results, STEP 2: Expand budget
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Abena Wingman?",
    "mode": "mix",
    "top_k": 50,
    "chunk_top_k": 25,
    "max_entity_tokens": 8000,
    "max_relation_tokens": 10000,
    "max_total_tokens": 40000
  }'
```

### C. Citation-Focused Query

```bash
# When you need to verify facts across diverse subjects
curl -X POST "https://lightrag.t51.it/query" \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the absorption specifications for Abena products?",
    "mode": "mix",
    "include_references": true,
    "response_type": "Detailed bullet points with citations",
    "max_total_tokens": 30000
  }'
```

---

## 6. Token Budget Guidelines for Diverse Subjects

### Conservative (Fast, Lower Cost)
```json
{
  "top_k": 20,
  "chunk_top_k": 10,
  "max_entity_tokens": 3000,
  "max_relation_tokens": 4000,
  "max_total_tokens": 15000
}
```
**Use when:** Simple queries, known entities, tight latency requirements

### Balanced (Recommended)
```json
{
  "top_k": 40,
  "chunk_top_k": 20,
  "max_entity_tokens": 6000,
  "max_relation_tokens": 8000,
  "max_total_tokens": 30000
}
```
**Use when:** General purpose queries, diverse subjects, good balance

### Comprehensive (Thorough, Higher Cost)
```json
{
  "top_k": 60,
  "chunk_top_k": 30,
  "max_entity_tokens": 10000,
  "max_relation_tokens": 12000,
  "max_total_tokens": 45000
}
```
**Use when:** Complex cross-domain queries, exhaustive research needed

### Maximum (Use Sparingly)
```json
{
  "top_k": 100,
  "chunk_top_k": 50,
  "max_entity_tokens": 15000,
  "max_relation_tokens": 18000,
  "max_total_tokens": 60000
}
```
**Use when:** Comprehensive analysis required, willing to pay for tokens

---

## 7. Common Workflows

### Workflow 1: Exploring a New Topic Area

```bash
# 1. Find entities in the topic area
curl -X GET "https://lightrag.t51.it/graph/label/search?query=medical&limit=50"

# 2. Get popular entities to find hubs
curl -X GET "https://lightrag.t51.it/graph/label/popular?limit=20"

# 3. Explore neighborhood of a hub entity
curl -X GET "https://lightrag.t51.it/graphs?label=Medical%20Devices&max_depth=2"

# 4. Query with discovered entities
curl -X POST "https://lightrag.t51.it/query/data" -d '{
  "query": "What medical devices are used for incontinence?",
  "mode": "mix",
  "max_total_tokens": 30000
}'
```

### Workflow 2: Cleaning Up Duplicates

```bash
# 1. Search for potential duplicates
curl -X GET "https://lightrag.t51.it/graph/label/search?query=neural"

# 2. Check each variant exists
curl -X GET "https://lightrag.t51.it/graph/entity/exists?entity_name=neural%20network"
curl -X GET "https://lightrag.t51.it/graph/entity/exists?entity_name=Neural%20Network"

# 3. Merge duplicates
curl -X POST "https://lightrag.t51.it/graph/entities/merge" -d '{
  "source_entities": ["neural network", "Neural Networks"],
  "target_entity": "Neural Network"
}'
```

### Workflow 3: Cross-Domain Analysis

```bash
# 1. Query first domain
curl -X POST "https://lightrag.t51.it/query/data" -d '{
  "query": "What are key concepts in machine learning?",
  "mode": "global",
  "max_total_tokens": 20000
}'

# 2. Query second domain
curl -X POST "https://lightrag.t51.it/query/data" -d '{
  "query": "What are key concepts in medical devices?",
  "mode": "global",
  "max_total_tokens": 20000
}'

# 3. Cross-domain query
curl -X POST "https://lightrag.t51.it/query/data" -d '{
  "query": "How can machine learning improve medical devices?",
  "mode": "mix",
  "hl_keywords": ["machine learning", "medical devices", "innovation"],
  "max_total_tokens": 40000
}'
```

---

## Summary: Key Improvements for Your Use Case

✅ **Token budgets** prevent context overflow with diverse subjects
✅ **Discovery endpoints** help you explore what's in your knowledge base
✅ **Entity management** maintains quality as your KB grows
✅ **Manual keywords** give control when automatic extraction fails
✅ **include_references** enables fact verification across domains
✅ **Different modes** optimize for different query types

The improved YAML configuration exposes all these features!
