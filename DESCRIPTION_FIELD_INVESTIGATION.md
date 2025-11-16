# LightRAG Description Field Investigation

**Investigation Date:** 2025-11-16
**Branch:** claude/investigate-description-search-01LwsnXn23D66N5t83bmDHmY

## Executive Summary

**Critical Finding:** LightRAG does **NOT support document-level descriptions**. The `description` field only exists for entities and relationships within the knowledge graph, not for documents themselves.

---

## Key Findings

### 1. ❌ Document-Level Description: NOT SUPPORTED

**The description field does NOT exist at the document level.**

When inserting documents via `insert()` or `ainsert()`, the available parameters are:
- `input`: Document content (string or list)
- `ids`: Document IDs
- `file_paths`: File paths for citations
- `track_id`: Processing tracking ID

**Location:** `lightrag/lightrag.py:1122-1155`

Documents are stored using `DocProcessingStatus` structure:
```python
@dataclass
class DocProcessingStatus:
    content_summary: str       # First 100 chars preview
    content_length: int        # Total length
    file_path: str            # File path
    status: DocStatus         # Processing status
    created_at: str          # Timestamp
    updated_at: str          # Timestamp
    track_id: str | None     # Tracking ID
    chunks_count: int | None # Chunk count
    chunks_list: list[str]   # Chunk IDs
    error_msg: str | None    # Errors
    metadata: dict[str, Any] # Generic metadata (unused)
```

**Location:** `lightrag/base.py:677-722`

**Note:** There is a generic `metadata` field, but:
- Not exposed through the insert API
- Not used in entity extraction
- Not used in queries
- Only for internal status tracking

---

### 2. ❌ Description is NOT Searchable or Filterable

Document descriptions cannot be searched or filtered because:

1. **No storage mechanism** - Documents don't have a description field
2. **Query pipeline ignores document metadata** - See `lightrag/operate.py:2999-3148`
   - Retrieves entities from knowledge graph
   - Retrieves relationships from knowledge graph
   - Retrieves text chunks from vector storage
   - Retrieves file paths for citations
   - **Does NOT access document-level metadata**

3. **Reference generation** uses only:
   - `file_path` for citations
   - `reference_id` for numbering
   - **Location:** `lightrag/utils.py:3096-3159`

**However:** Entity and relationship descriptions ARE fully searchable:
- Indexed in vector databases (entities_vdb, relationships_vdb)
- Used in semantic search
- Included in query context

---

### 3. ❌ Description NOT Used in Entity Extraction

The entity extraction process receives **ONLY chunk content**, no document metadata.

**Location:** `lightrag/operate.py:2754-2804`

Input to entity extraction:
- `TextChunkSchema` with:
  - `content` - Raw chunk text
  - `tokens` - Token count
  - `full_doc_id` - Document ID
  - `chunk_order_index` - Chunk position
- **No document-level context, tags, or metadata**

**Extraction Prompt** (`lightrag/prompt.py:11-69`):
```
Entity_types: [{entity_types}]
Text:
```
{input_text}
```
```

**The LLM only sees the raw chunk text** - no access to any document-level description or tags.

---

### 4. ✅ Description is CRITICAL for Entities (NOT Documents)

**Entity descriptions are fully integrated into the knowledge graph:**

**Location:** `lightrag/lightrag.py:2238-2268`

```python
node_data = {
    "entity_id": entity_name,
    "entity_type": entity_type,
    "description": description,  # From LLM extraction
    "source_id": source_id,
    "file_path": file_path,
    "created_at": int(time.time()),
}
```

**Entity descriptions are:**
1. ✅ Extracted by LLM from document text (`lightrag/operate.py:408-419`)
2. ✅ Stored in knowledge graph
3. ✅ Indexed in vector databases as `entity_name + "\n" + description`
4. ✅ Used for semantic search during queries
5. ✅ Included in query context sent to LLM

**Relationship descriptions** follow the same pattern with `src_id`, `tgt_id`, `description`, `keywords`, and `weight`.

**Document-level descriptions would NOT be integrated** because they're not part of the insertion flow.

---

### 5. ❌ Tags in Document Descriptions Would Be IGNORED

If you tried to add tags like `[DOMAIN:photography]` to a document description:

1. ❌ Document insertion doesn't accept description parameter
2. ❌ Entity extraction doesn't receive document metadata
3. ❌ LLM only sees chunk text, not document-level tags

**Workaround:** Tags in the actual document **content** would be processed:
- If you include `[DOMAIN:photography]` in the document text itself
- The LLM sees it during entity extraction
- Might extract it as an entity (unlikely due to brackets)
- More likely used as context for nearby entities

**Custom KG Insertion** allows tagged descriptions:
```python
# Location: lightrag/lightrag.py:2194
custom_kg = {
    "entities": [{
        "entity_name": "PhotoEntity",
        "entity_type": "DOMAIN",
        "description": "Photography-related entity [DOMAIN:photography]",
        # Can contain any tags, stored as text
    }]
}
```

Tags in entity descriptions are stored as text, not parsed as structured metadata.

---

## What We DO Know (Confirmed)

✅ **v1.4.9 release added references field** - Confirmed in query results
✅ **Documents have IDs** - Returned in status and tracking
✅ **Document status system** - `DocProcessingStatus` tracks document lifecycle
✅ **Entity/relationship descriptions** - Fully integrated and searchable
✅ **Generic metadata field exists** - But not exposed or utilized

---

## Architectural Implications

### Current Architecture

```
Document Insert Flow:
┌─────────────┐
│  Document   │ (content, id, file_path)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Chunking  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ LLM Extract │ (only sees chunk text)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Entities   │ (name, type, description)
│ Relationships│ (src, tgt, description)
└─────────────┘
```

**No path exists** for document-level description to reach:
- Entity extraction
- Knowledge graph
- Query system

---

## Recommendations

### Current Workarounds

1. **Embed metadata in content**
   ```python
   content = "[DOMAIN:photography] [TAGS:landscape,nature] " + actual_content
   lightrag.insert(content)
   ```
   - Pros: Gets processed by entity extraction
   - Cons: Pollutes document content, inconsistent extraction

2. **Use file_path for metadata**
   ```python
   lightrag.insert(content, file_paths=["photography/landscape/photo1.txt"])
   ```
   - Pros: Preserved in citations
   - Cons: Limited structure, path constraints

3. **Use custom KG insertion**
   ```python
   lightrag.insert_custom_kg({
       "entities": [{
           "entity_name": "Photo1",
           "entity_type": "Document",
           "description": "[DOMAIN:photography] Landscape photo description"
       }]
   })
   ```
   - Pros: Full control over entities
   - Cons: Bypasses automatic extraction, manual maintenance

4. **Fork and modify**
   - Add description parameter to insert methods
   - Expose metadata field in DocProcessingStatus
   - Pass description to entity extraction prompts
   - Files to modify:
     - `lightrag/lightrag.py` - insert methods
     - `lightrag/base.py` - DocProcessingStatus
     - `lightrag/operate.py` - entity extraction
     - `lightrag/api/routers/document_routes.py` - API models

### Feature Request Recommendation

Consider proposing to the LightRAG maintainers:

**Document-level metadata support:**
```python
lightrag.insert(
    input=content,
    metadata={
        "description": "Photography tutorial",
        "tags": ["photography", "landscape"],
        "domain": "tutorial",
        "author": "John Doe"
    }
)
```

**Integration points:**
1. Include in entity extraction context
2. Store in vector database for filtering
3. Expose in query results
4. Allow filtering: `query(..., filter={"domain": "photography"})`

---

## Code References

- Document insertion: `lightrag/lightrag.py:1122-1155`
- DocProcessingStatus: `lightrag/base.py:677-722`
- Entity extraction: `lightrag/operate.py:2754-2804`
- Entity prompts: `lightrag/prompt.py:11-69`
- Knowledge graph insertion: `lightrag/lightrag.py:2238-2268`
- Query pipeline: `lightrag/operate.py:2999-3148`
- Reference generation: `lightrag/utils.py:3096-3159`
- Custom KG: `lightrag/lightrag.py:2194`

---

## Conclusion

The `description` field **does not exist for documents** in LightRAG. It only exists for entities and relationships within the knowledge graph. To use descriptions or tags effectively, you must either:

1. Embed them in document content (processed during extraction)
2. Use custom KG insertion to manually create tagged entities
3. Fork the codebase to add document-level metadata support

**The knowledge graph is entity-centric, not document-centric.** Documents are merely sources from which entities and relationships are extracted. All searchable, queryable metadata lives at the entity/relationship level, not the document level.
