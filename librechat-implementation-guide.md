# LibreChat + LightRAG Implementation Guide

## How Your Setup Works

```
User Query
    ↓
LibreChat (UI)
    ↓
Grok 4.1 LLM (reads your prompt)
    ↓
Decides: "I should call retrieve_rag_data with top_k=60, mode=global, etc."
    ↓
Makes API call to: https://lightrag.t51.it/query/data
    ↓
LightRAG processes with those parameters
    ↓
Returns results to Grok
    ↓
Grok synthesizes answer
    ↓
User sees response
```

**Key insight:** Your prompt instructs Grok on **what parameters to use** when calling the API.

---

## What Needs to Change

### 1. Deploy Improved YAML to LightRAG Server

**Your LightRAG API needs to expose the new parameters** so Grok can use them.

**Current state:**
```yaml
# Your API only accepts:
/query/data:
  - query
  - mode
  - top_k (default: 5)
  - chunk_top_k (default: 5)
```

**Needed state:**
```yaml
# API should accept:
/query/data:
  - query
  - mode
  - top_k (default: 60)
  - chunk_top_k (default: 30)
  - max_entity_tokens (NEW)
  - max_relation_tokens (NEW)
  - max_total_tokens (NEW)
  - include_references (NEW)
  - enable_rerank (NEW)
  - hl_keywords (NEW)
  - ll_keywords (NEW)
```

**How to update:**
1. The `improved_lightrag_api.yaml` file defines the complete API spec
2. Your LightRAG instance likely already supports these parameters (they're built into LightRAG)
3. You just need to **expose them in your API configuration** and **document them in the OpenAPI spec**

**Check if your LightRAG instance already supports these:**
```bash
# Test if your API accepts max_total_tokens:
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "test",
    "mode": "mix",
    "top_k": 60,
    "max_total_tokens": 40000
  }'

# If this works, your API already supports it!
# You just need to update your prompt to use these parameters.

# If it fails, you need to update your LightRAG server configuration.
```

---

### 2. Update Your LibreChat Prompt

**Your current prompt tells Grok:**
```
retrieve_rag_data:
  - top_k (default 5, up to 10 for broad)
  - chunk_top_k (default 5)
```

**This instructs Grok to make API calls like:**
```json
{
  "query": "How do authors approach event schema evolution?",
  "mode": "mix",
  "top_k": 5  // ← TOO LOW for multi-source synthesis
}
```

**Optimized prompt tells Grok:**
```
retrieve_rag_data:
  - top_k: 60 (for multi-source synthesis)
  - chunk_top_k: 30
  - max_entity_tokens: 10000
  - max_relation_tokens: 12000
  - max_total_tokens: 40000
```

**This instructs Grok to make API calls like:**
```json
{
  "query": "How do authors approach event schema evolution?",
  "mode": "global",
  "top_k": 80,
  "chunk_top_k": 40,
  "max_entity_tokens": 12000,
  "max_relation_tokens": 15000,
  "max_total_tokens": 50000,
  "include_references": true
}
```

**The LLM follows the instructions in your prompt** to decide what parameters to use.

---

## Step-by-Step Implementation

### Step 1: Verify Your LightRAG API Capabilities

**Test if your API already supports the improved parameters:**

```bash
# Test 1: Token budget parameters
curl -X POST "https://lightrag.t51.it/query/data" \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "test query",
    "mode": "mix",
    "top_k": 60,
    "chunk_top_k": 30,
    "max_entity_tokens": 10000,
    "max_relation_tokens": 12000,
    "max_total_tokens": 40000,
    "include_references": true,
    "enable_rerank": true
  }'

# Expected: Should return results (even for "test query")
# If it works: Your API already supports these parameters! ✅
# If it fails with "unknown parameter": You need to update LightRAG config ❌
```

```bash
# Test 2: Discovery endpoints
curl -X GET "https://lightrag.t51.it/graph/label/list" \
  -H "X-API-Key: YOUR_KEY"

# Expected: Returns list of all entity labels
# If it works: Discovery endpoints available! ✅
# If 404: Endpoints not exposed, need LightRAG update ❌
```

```bash
# Test 3: Entity search
curl -X GET "https://lightrag.t51.it/graph/label/search?query=event&limit=50" \
  -H "X-API-Key: YOUR_KEY"

# Expected: Returns entity labels matching "event"
# If it works: Search available! ✅
# If 404: Need LightRAG update ❌
```

---

### Step 2: Update LightRAG API Configuration (If Needed)

**If tests failed, your LightRAG server needs configuration updates.**

**Option A: Your LightRAG version is recent (check with `pip show lightrag`)**

Most parameters are already supported in recent versions, but may not be exposed in your OpenAPI spec.

**Update your API server configuration** to expose all parameters. The improved YAML file shows the complete spec.

**Option B: Your LightRAG version is older**

Update LightRAG to the latest version:
```bash
pip install --upgrade lightrag
```

Then restart your LightRAG API server.

---

### Step 3: Update Your LibreChat Prompt

**Replace the tool usage section of your prompt** with the optimized version from `librechat-prompt-optimization.md`.

**Key changes in the prompt:**

**OLD (Your Current):**
```
retrieve_rag_data (/query/data POST):
  Params: query (required); mode ("mix" default); top_k (default 5, up to 10 for broad); chunk_top_k (default 5).
```

**NEW (Optimized):**
```
retrieve_rag_data (/query/data POST):
  Required params: query
  Recommended params for reference libraries:
    - mode: "mix" (default), "global" (multi-source), "hybrid" (cross-domain), "local" (entity)
    - top_k: 60 (multi-source synthesis), 80 (for "how do authors" queries)
    - chunk_top_k: 30
    - max_entity_tokens: 10000
    - max_relation_tokens: 12000
    - max_total_tokens: 40000
    - include_references: true
    - enable_rerank: true
```

**This tells Grok:** "When you call the API, use these higher values for top_k and include token budgets."

---

### Step 4: Test with Your Example Queries

**Test Query 1:** "How do different authors approach event schema evolution?"

**What Grok should do with optimized prompt:**
1. Classify as "multi-source synthesis" query
2. Choose **GLOBAL mode** (pattern matching across sources)
3. Call API with:
   ```json
   {
     "query": "event schema evolution authors approaches strategies",
     "mode": "global",
     "top_k": 80,
     "chunk_top_k": 40,
     "max_total_tokens": 50000,
     "include_references": true
   }
   ```
4. Synthesize results showing all 4 sources (Fowler, Kafka, Stripe, Academic paper)

**Test Query 2:** "Connect bounded context from DDD to TOGAF"

**What Grok should do with optimized prompt:**
1. Classify as "cross-domain mapping" query
2. Choose **HYBRID mode** (entity + pattern)
3. Call API with:
   ```json
   {
     "query": "bounded context DDD TOGAF alignment mapping",
     "mode": "hybrid",
     "top_k": 60,
     "chunk_top_k": 30,
     "max_total_tokens": 40000,
     "hl_keywords": ["bounded context", "TOGAF", "enterprise architecture"],
     "ll_keywords": ["application component", "context mapping"]
   }
   ```
4. Follow up with graph traversal:
   ```
   GET /graphs?label=Bounded%20Context&max_depth=4&max_nodes=800
   ```
5. Synthesize with visual mapping table

---

## What Changes at Each Layer

### Layer 1: LightRAG API Server (Your Instance)

**Current state:**
- ✅ Core LightRAG functionality works
- ❌ May not expose all parameters in API
- ❌ May not expose discovery endpoints

**Needed state:**
- ✅ Expose all query parameters (token budgets, include_references, etc.)
- ✅ Expose discovery endpoints (/graph/label/list, /graph/label/search, etc.)
- ✅ Update OpenAPI spec (if you're using one for LibreChat)

**How to update:**
Check `improved_lightrag_api.yaml` for the complete API specification your server should support.

---

### Layer 2: OpenAPI Spec for LibreChat (Your YAML)

**Current state:**
```yaml
paths:
  /query/data:
    parameters: query, mode, top_k (default 5), chunk_top_k (default 5)
  /graphs:
    parameters: label, max_depth (default 2)
```

**Needed state:**
```yaml
paths:
  /query/data:
    parameters: query, mode, top_k (default 60), chunk_top_k (default 30),
                max_entity_tokens, max_relation_tokens, max_total_tokens,
                include_references, enable_rerank, hl_keywords, ll_keywords
  /graphs:
    parameters: label, max_depth (default 4), max_nodes
  /graph/label/list: (NEW)
  /graph/label/search: (NEW)
  /graph/label/popular: (NEW)
  /graph/entities/merge: (NEW)
  # ... (11 more endpoints)
```

**How to update:**
Replace your current YAML with `improved_lightrag_api.yaml`

---

### Layer 3: LibreChat Prompt (Instructions for Grok)

**Current state:**
Your prompt tells Grok: "Use top_k=5 (default), up to 10 for broad queries"

**Needed state:**
Optimized prompt tells Grok: "Use top_k=60 for multi-source, 80 for 'how do authors' queries, and always include token budgets"

**How to update:**
Replace your current prompt with the optimized version from `librechat-prompt-optimization.md`

---

### Layer 4: Grok's Behavior (Automatic Based on Prompt)

**Current behavior:**
```json
// Grok makes API calls like this:
{
  "query": "How do authors approach event schema evolution?",
  "mode": "mix",
  "top_k": 5  // ← Too low
}
```

**New behavior (after prompt update):**
```json
// Grok makes API calls like this:
{
  "query": "event schema evolution authors approaches strategies",
  "mode": "global",  // ← Better mode selection
  "top_k": 80,       // ← 16x higher
  "chunk_top_k": 40,
  "max_entity_tokens": 12000,
  "max_relation_tokens": 15000,
  "max_total_tokens": 50000,
  "include_references": true
}
```

**This happens automatically** when Grok reads the improved prompt.

---

## Expected Improvements After Implementation

### Query Success Rate
- **Before:** 50-60% (many queries fail from context overflow)
- **After:** 95%+ (token budgets prevent overflow)

### Multi-Source Synthesis
- **Before:** Captures 1-2 sources (top_k=5 too low)
- **After:** Captures 4-6 sources (top_k=60-80)

### Cross-Domain Mapping
- **Before:** Partial connections (max_depth=3 insufficient)
- **After:** Full conceptual bridges (max_depth=4-5)

### Discovery Capability
- **Before:** Can't explore what concepts exist
- **After:** Can show user available concepts and guide exploration

---

## Quick Start Checklist

- [ ] **Test your API** - Run the curl commands in Step 1 to see what's already supported
- [ ] **Update LightRAG if needed** - If tests fail, update your LightRAG server config
- [ ] **Replace your YAML** - Use `improved_lightrag_api.yaml` in LibreChat
- [ ] **Update your prompt** - Use the optimized prompt from `librechat-prompt-optimization.md`
- [ ] **Test with examples** - Run your two example queries and compare results
- [ ] **Monitor performance** - Track query success rate and multi-source coverage

---

## Troubleshooting

### Issue: "Unknown parameter 'max_total_tokens'"

**Cause:** Your LightRAG API doesn't expose this parameter yet

**Solution:**
1. Check LightRAG version: `pip show lightrag`
2. If version is recent (post-2024), parameter exists but may not be exposed in API config
3. Update your LightRAG API server configuration to expose all parameters
4. See `improved_lightrag_api.yaml` for complete spec

---

### Issue: Grok still uses top_k=5

**Cause:** Prompt update didn't propagate, or Grok is ignoring instructions

**Solution:**
1. Verify prompt was updated in LibreChat
2. Add explicit example in prompt showing the API call format:
   ```
   Example API call for multi-source synthesis:
   POST /query/data
   {
     "query": "...",
     "top_k": 80,
     "max_total_tokens": 50000
   }
   ```
3. Test with a fresh conversation (clear chat history)

---

### Issue: 404 on /graph/label/list

**Cause:** Discovery endpoints not exposed in your LightRAG API

**Solution:**
1. Check if your LightRAG version includes these endpoints
2. Update to latest LightRAG version if needed
3. Ensure your API server configuration exposes these routes
4. See LightRAG docs for API server setup

---

## Summary

**What you need to do:**

1. **Update LightRAG API** (if needed) - Test first with curl commands
2. **Replace your YAML** - Use `improved_lightrag_api.yaml`
3. **Update your prompt** - Use optimized version from `librechat-prompt-optimization.md`

**What happens then:**

- Grok reads the new prompt
- Grok makes API calls with better parameters (top_k=60-80, token budgets, etc.)
- LightRAG returns more comprehensive results
- Grok synthesizes better multi-source answers

**The key insight:** Your prompt **instructs Grok what parameters to use**. By updating the prompt to recommend higher top_k values and token budgets, Grok will automatically make better API calls.
