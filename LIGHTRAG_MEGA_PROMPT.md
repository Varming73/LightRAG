# MEGA PROMPT: Complete LightRAG Knowledge Graph RAG Application

## Project Overview
Create a production-ready full-stack web application that enables users to upload documents and interact with them through an AI-powered chat interface using LightRAG's Knowledge Graph-based RAG system. The application implements a complete Retrieval Augmented Generation (RAG) system with:
- Modern, clean UI with tabbed interface
- Knowledge Graph visualization capabilities
- API documentation with ready-to-use code examples
- Markdown-formatted responses with citations
- Real-time streaming responses
- Complete file and knowledge graph management
- Multiple query modes (local, global, hybrid, mix, naive, bypass)
- System prompt customization
- Citation tracking and reference display

## Core Technologies
- **Backend Framework**: FastAPI (Python 3.10+)
- **Frontend**: Vanilla JavaScript, HTML5, CSS3, marked.js for markdown
- **RAG System**: LightRAG with Knowledge Graph
- **Graph Storage**: Neo4j (recommended) or NetworkX
- **Vector Storage**: PostgreSQL with pgvector (recommended) or NanoVectorDB
- **File Handling**: FastAPI file uploads with textract
- **Environment Management**: python-dotenv
- **CORS**: FastAPI CORS middleware
- **Streaming**: NDJSON format for real-time responses

## LightRAG Architecture

### Knowledge Graph RAG
Unlike traditional vector-only RAG systems, LightRAG builds a knowledge graph from your documents:
1. **Chunking**: Documents split into overlapping chunks
2. **Entity Extraction**: LLM extracts entities (people, places, concepts, etc.)
3. **Relationship Extraction**: LLM identifies relationships between entities
4. **Graph Building**: Entities and relationships form a queryable knowledge graph
5. **Hybrid Retrieval**: Combines knowledge graph + vector search for superior results

### Query Modes
- **local**: Entity-focused retrieval with direct relationships (best for specific questions)
- **global**: Pattern analysis across entire knowledge graph (best for broad questions)
- **hybrid**: Combines local and global strategies
- **naive**: Simple vector similarity search only (traditional RAG)
- **mix**: Integrates knowledge graph + vector retrieval (RECOMMENDED - best of both worlds)
- **bypass**: Direct LLM query without retrieval

## Dependencies (requirements.txt)
```
# LightRAG is already installed in this repository
fastapi==0.115.0
uvicorn[standard]==0.32.0
python-dotenv==1.0.0
marked==0.3.0
python-multipart==0.0.12
aiofiles==24.1.0

# LightRAG dependencies (already in repository)
# Ensure these are installed:
# - lightrag (from this repo)
# - neo4j (if using Neo4j storage)
# - psycopg (if using Postgres storage)
# - textract (for file processing)
```

## Architecture & File Structure

```
lightrag-web-app/
├── app.py                    # FastAPI backend application
├── templates/
│   └── index.html           # Single-page frontend application
├── uploads/                 # Temporary file storage (auto-created)
├── rag_storage/             # LightRAG working directory (auto-created)
├── .env                     # Environment variables (LLM, embeddings, storage)
├── .gitignore              # Git exclusions
├── requirements.txt        # Python dependencies
├── README.md               # Documentation
└── LIGHTRAG_MEGA_PROMPT.md # This file
```

## Critical Implementation Requirements

### 1. LightRAG Integration
**MUST IMPLEMENT**: Properly initialize LightRAG with storage backends and model functions.

### 2. Multiple Query Modes
**MUST IMPLEMENT**: Support all LightRAG query modes with clear UI selection.

### 3. API Documentation Tab
**MUST IMPLEMENT**: A dedicated tab showing:
- Current configuration and storage information
- Ready-to-copy cURL commands for all endpoints
- Python SDK examples
- Query mode explanations

### 4. Markdown Support with Citations
**MUST IMPLEMENT**: AI responses must be formatted using marked.js library with:
- Proper markdown rendering
- Reference citations at the end
- Source file tracking

### 5. Streaming Indicators
**MUST IMPLEMENT**: Animated typing indicator while waiting for streaming AI responses.

### 6. Knowledge Graph Features
**MUST IMPLEMENT**: Display knowledge graph statistics (entities, relationships, documents).

## Backend Implementation (app.py)

### Complete FastAPI Application

```python
from fastapi import FastAPI, File, UploadFile, HTTPException, Body
from fastapi.responses import HTMLResponse, StreamingResponse
from fastapi.staticfiles import StaticFiles
from fastapi.middleware.cors import CORSMiddleware
from fastapi.templating import Jinja2Templates
from fastapi import Request
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any, Literal
import os
import json
import asyncio
from pathlib import Path
from dotenv import load_dotenv
import logging
from datetime import datetime

# LightRAG imports
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import openai_complete_if_cache, openai_embedding
from lightrag.llm.ollama import ollama_model_complete, ollama_embedding
from lightrag.kg.shared_storage import initialize_pipeline_status

# Load environment variables
load_dotenv()

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Configuration
UPLOAD_FOLDER = 'uploads'
WORKING_DIR = os.getenv('WORKING_DIR', './rag_storage')
INPUT_DIR = os.getenv('INPUT_DIR', './inputs')
MAX_FILE_SIZE = 100 * 1024 * 1024  # 100MB

# Create directories
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(WORKING_DIR, exist_ok=True)
os.makedirs(INPUT_DIR, exist_ok=True)
os.makedirs('templates', exist_ok=True)

# Initialize FastAPI app
app = FastAPI(
    title="LightRAG Knowledge Graph Assistant",
    description="AI-powered document chat with Knowledge Graph RAG",
    version="1.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Templates
templates = Jinja2Templates(directory="templates")

# ============================================
# LightRAG INITIALIZATION
# ============================================

# Determine LLM configuration
LLM_BINDING = os.getenv('LLM_BINDING', 'openai').lower()
LLM_MODEL = os.getenv('LLM_MODEL', 'gpt-4o-mini')
EMBEDDING_BINDING = os.getenv('EMBEDDING_BINDING', 'openai').lower()
EMBEDDING_MODEL = os.getenv('EMBEDDING_MODEL', 'text-embedding-3-small')

# Select LLM function based on binding
if LLM_BINDING == 'openai':
    from lightrag.llm.openai import openai_complete_if_cache
    llm_model_func = openai_complete_if_cache
elif LLM_BINDING == 'ollama':
    from lightrag.llm.ollama import ollama_model_complete
    llm_model_func = ollama_model_complete
else:
    raise ValueError(f"Unsupported LLM binding: {LLM_BINDING}")

# Select embedding function based on binding
if EMBEDDING_BINDING == 'openai':
    from lightrag.llm.openai import openai_embedding
    embedding_func = openai_embedding
elif EMBEDDING_BINDING == 'ollama':
    from lightrag.llm.ollama import ollama_embedding
    embedding_func = ollama_embedding
else:
    raise ValueError(f"Unsupported embedding binding: {EMBEDDING_BINDING}")

# Global LightRAG instance
rag: Optional[LightRAG] = None

async def init_rag():
    """Initialize LightRAG instance"""
    global rag
    try:
        logger.info("Initializing LightRAG...")
        rag = LightRAG(
            working_dir=WORKING_DIR,
            llm_model_func=llm_model_func,
            embedding_func=embedding_func,
        )

        # Initialize storage backends
        await rag.initialize_storages()
        await initialize_pipeline_status()

        logger.info("LightRAG initialized successfully")
        logger.info(f"Working directory: {WORKING_DIR}")
        logger.info(f"LLM: {LLM_BINDING} - {LLM_MODEL}")
        logger.info(f"Embeddings: {EMBEDDING_BINDING} - {EMBEDDING_MODEL}")

    except Exception as e:
        logger.error(f"Failed to initialize LightRAG: {str(e)}")
        raise

# Startup event
@app.on_event("startup")
async def startup_event():
    await init_rag()

# Shutdown event
@app.on_event("shutdown")
async def shutdown_event():
    if rag:
        logger.info("Shutting down LightRAG...")
        # Finalize storage connections
        try:
            await rag.finalize_storages()
        except Exception as e:
            logger.error(f"Error during shutdown: {str(e)}")

# ============================================
# PYDANTIC MODELS
# ============================================

class QueryRequest(BaseModel):
    query: str = Field(..., min_length=3, description="The query text")
    mode: Literal["local", "global", "hybrid", "naive", "mix", "bypass"] = Field(
        default="mix",
        description="Query mode"
    )
    stream: bool = Field(default=True, description="Enable streaming response")
    include_references: bool = Field(default=True, description="Include source citations")
    response_type: str = Field(default="Multiple Paragraphs", description="Response format")
    top_k: Optional[int] = Field(default=None, ge=1, description="Number of entities/relations to retrieve")
    chunk_top_k: Optional[int] = Field(default=None, ge=1, description="Number of chunks to retrieve")
    max_total_tokens: Optional[int] = Field(default=None, ge=1, description="Maximum total tokens")
    conversation_history: Optional[List[Dict[str, str]]] = Field(
        default=None,
        description="Conversation history"
    )
    user_prompt: Optional[str] = Field(default=None, description="Custom instructions for LLM")

class UploadResponse(BaseModel):
    success: bool
    message: str
    filename: str
    file_size: int
    processing_status: str

class QueryResponse(BaseModel):
    response: str
    references: Optional[List[Dict[str, str]]] = None

class StatsResponse(BaseModel):
    success: bool
    total_documents: int
    total_entities: int
    total_relationships: int
    total_chunks: int
    storage_info: Dict[str, Any]

# ============================================
# MAIN ROUTES
# ============================================

@app.get("/", response_class=HTMLResponse)
async def index(request: Request):
    """Serve the main HTML page"""
    return templates.TemplateResponse("index.html", {"request": request})

# ============================================
# DOCUMENT UPLOAD
# ============================================

@app.post("/upload", response_model=UploadResponse)
async def upload_file(file: UploadFile = File(...)):
    """Upload and process a document"""
    if not rag:
        raise HTTPException(status_code=500, detail="RAG system not initialized")

    try:
        # Save file temporarily
        file_path = os.path.join(UPLOAD_FOLDER, file.filename)

        # Read file content
        content = await file.read()
        file_size = len(content)

        # Save to disk
        with open(file_path, 'wb') as f:
            f.write(content)

        logger.info(f"File saved: {file.filename}, size: {file_size} bytes")

        # Read file content as text
        with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
            text_content = f.read()

        # Insert into LightRAG (async)
        logger.info(f"Processing {file.filename} with LightRAG...")
        await rag.ainsert(text_content)

        logger.info(f"File {file.filename} processed successfully")

        # Clean up temporary file
        os.remove(file_path)

        return UploadResponse(
            success=True,
            message=f"File '{file.filename}' uploaded and processed successfully",
            filename=file.filename,
            file_size=file_size,
            processing_status="completed"
        )

    except Exception as e:
        logger.error(f"Error uploading file: {str(e)}")
        # Clean up file if it exists
        if os.path.exists(file_path):
            os.remove(file_path)
        raise HTTPException(status_code=500, detail=f"Error uploading file: {str(e)}")

# ============================================
# CHAT WITH STREAMING
# ============================================

@app.post("/query")
async def query_non_streaming(request: QueryRequest):
    """Non-streaming query endpoint"""
    if not rag:
        raise HTTPException(status_code=500, detail="RAG system not initialized")

    try:
        # Build QueryParam
        param = QueryParam(
            mode=request.mode,
            stream=False,
            response_type=request.response_type,
            include_references=request.include_references,
        )

        if request.top_k:
            param.top_k = request.top_k
        if request.chunk_top_k:
            param.chunk_top_k = request.chunk_top_k
        if request.max_total_tokens:
            param.max_total_tokens = request.max_total_tokens
        if request.conversation_history:
            param.conversation_history = request.conversation_history
        if request.user_prompt:
            param.user_prompt = request.user_prompt

        # Query LightRAG
        result = await rag.aquery_llm(request.query, param=param)

        # Extract response and references
        llm_response = result.get("llm_response", {})
        response_content = llm_response.get("content", "No response generated")

        # Extract references
        references = None
        if request.include_references:
            data = result.get("data", {})
            references = data.get("references", [])

        return QueryResponse(
            response=response_content,
            references=references
        )

    except Exception as e:
        logger.error(f"Error in query: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Error processing query: {str(e)}")

@app.post("/query/stream")
async def query_streaming(request: QueryRequest):
    """Streaming query endpoint with NDJSON format"""
    if not rag:
        raise HTTPException(status_code=500, detail="RAG system not initialized")

    try:
        # Build QueryParam
        param = QueryParam(
            mode=request.mode,
            stream=request.stream,
            response_type=request.response_type,
            include_references=request.include_references,
        )

        if request.top_k:
            param.top_k = request.top_k
        if request.chunk_top_k:
            param.chunk_top_k = request.chunk_top_k
        if request.max_total_tokens:
            param.max_total_tokens = request.max_total_tokens
        if request.conversation_history:
            param.conversation_history = request.conversation_history
        if request.user_prompt:
            param.user_prompt = request.user_prompt

        # Query LightRAG
        result = await rag.aquery_llm(request.query, param=param)

        async def stream_generator():
            # Extract references and LLM response
            references = result.get("data", {}).get("references", [])
            llm_response = result.get("llm_response", {})

            if llm_response.get("is_streaming"):
                # Streaming mode: send references first, then stream response
                if request.include_references:
                    yield f"{json.dumps({'references': references})}\n"

                response_stream = llm_response.get("response_iterator")
                if response_stream:
                    try:
                        async for chunk in response_stream:
                            if chunk:
                                yield f"{json.dumps({'response': chunk})}\n"
                    except Exception as e:
                        logger.error(f"Streaming error: {str(e)}")
                        yield f"{json.dumps({'error': str(e)})}\n"
            else:
                # Non-streaming: send complete response
                response_content = llm_response.get("content", "")
                complete_response = {"response": response_content}
                if request.include_references:
                    complete_response["references"] = references
                yield f"{json.dumps(complete_response)}\n"

        return StreamingResponse(
            stream_generator(),
            media_type="application/x-ndjson",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",
            }
        )

    except Exception as e:
        logger.error(f"Error in streaming query: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Error processing query: {str(e)}")

# ============================================
# KNOWLEDGE GRAPH STATS
# ============================================

@app.get("/stats", response_model=StatsResponse)
async def get_stats():
    """Get knowledge graph statistics"""
    if not rag:
        raise HTTPException(status_code=500, detail="RAG system not initialized")

    try:
        # Get storage statistics
        stats = {
            "success": True,
            "total_documents": 0,
            "total_entities": 0,
            "total_relationships": 0,
            "total_chunks": 0,
            "storage_info": {
                "working_dir": WORKING_DIR,
                "llm_binding": LLM_BINDING,
                "llm_model": LLM_MODEL,
                "embedding_binding": EMBEDDING_BINDING,
                "embedding_model": EMBEDDING_MODEL,
            }
        }

        # Try to get entity count
        try:
            if hasattr(rag, 'chunk_entity_relation_graph'):
                graph = rag.chunk_entity_relation_graph
                if hasattr(graph, 'nodes'):
                    stats["total_entities"] = len([n for n in graph.nodes() if graph.nodes[n].get('entity_type')])
                if hasattr(graph, 'edges'):
                    stats["total_relationships"] = len(graph.edges())
        except Exception as e:
            logger.warning(f"Could not get graph stats: {e}")

        # Try to get chunk count
        try:
            if hasattr(rag, 'text_chunks'):
                stats["total_chunks"] = len(rag.text_chunks)
        except Exception as e:
            logger.warning(f"Could not get chunk count: {e}")

        return StatsResponse(**stats)

    except Exception as e:
        logger.error(f"Error getting stats: {str(e)}")
        raise HTTPException(status_code=500, detail=f"Error getting stats: {str(e)}")

# ============================================
# API DOCUMENTATION INFO
# ============================================

@app.get("/api-info")
async def get_api_info():
    """Get API configuration for documentation tab"""
    return {
        "success": True,
        "llm_binding": LLM_BINDING,
        "llm_model": LLM_MODEL,
        "embedding_binding": EMBEDDING_BINDING,
        "embedding_model": EMBEDDING_MODEL,
        "working_dir": WORKING_DIR,
        "query_modes": [
            {
                "name": "mix",
                "description": "Integrates knowledge graph + vector retrieval (RECOMMENDED)",
                "best_for": "Most use cases - best balance of accuracy and context"
            },
            {
                "name": "local",
                "description": "Entity-focused retrieval with direct relationships",
                "best_for": "Specific questions about particular entities or topics"
            },
            {
                "name": "global",
                "description": "Pattern analysis across entire knowledge graph",
                "best_for": "Broad questions requiring understanding of overall themes"
            },
            {
                "name": "hybrid",
                "description": "Combines local and global strategies",
                "best_for": "Complex questions requiring both specific and broad context"
            },
            {
                "name": "naive",
                "description": "Simple vector similarity search only",
                "best_for": "Testing or when knowledge graph is not needed"
            },
            {
                "name": "bypass",
                "description": "Direct LLM query without retrieval",
                "best_for": "General questions not requiring document context"
            }
        ],
        "endpoints": [
            {
                "path": "/upload",
                "method": "POST",
                "description": "Upload and process documents"
            },
            {
                "path": "/query",
                "method": "POST",
                "description": "Non-streaming query endpoint"
            },
            {
                "path": "/query/stream",
                "method": "POST",
                "description": "Streaming query endpoint (NDJSON format)"
            },
            {
                "path": "/stats",
                "method": "GET",
                "description": "Get knowledge graph statistics"
            }
        ]
    }

# ============================================
# HEALTH CHECK
# ============================================

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "rag_initialized": rag is not None,
        "working_dir": WORKING_DIR
    }

if __name__ == "__main__":
    import uvicorn

    host = os.getenv("HOST", "0.0.0.0")
    port = int(os.getenv("PORT", "8000"))

    uvicorn.run(
        "app:app",
        host=host,
        port=port,
        reload=True,
        log_level="info"
    )
```

## Frontend Implementation (templates/index.html)

### CRITICAL REQUIREMENTS

1. **Include marked.js CDN**: For markdown rendering
2. **Three-tab interface**: "Chat", "Knowledge Graph Stats", "API Documentation"
3. **Query mode selector**: Dropdown for all 6 query modes with descriptions
4. **Streaming support**: Handle NDJSON streaming responses
5. **Citations display**: Show reference sources after each response
6. **Knowledge graph visualization**: Display stats about entities and relationships
7. **API examples**: cURL and Python code samples

### Complete HTML Structure

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LightRAG Knowledge Graph Assistant</title>

    <!-- Marked.js for markdown rendering -->
    <script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>

    <style>
        /* ==========================================
           MODERN LIGHTRAG ASSISTANT UI
           ========================================== */

        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        :root {
            --primary-color: #0F766E;
            --primary-light: #0D9488;
            --secondary-color: #F0FDFA;
            --text-dark: #111827;
            --text-medium: #6B7280;
            --text-light: #9CA3AF;
            --bg-light: #F9FAFB;
            --bg-lighter: #F3F4F6;
            --border-color: #E5E7EB;
            --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
            --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
            --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Inter', 'Roboto', 'Segoe UI', sans-serif;
            background: linear-gradient(to bottom, var(--bg-light) 0%, var(--bg-lighter) 100%);
            min-height: 100vh;
            padding: 32px 20px;
            color: var(--text-dark);
            line-height: 1.6;
        }

        .container {
            max-width: 1600px;
            margin: 0 auto;
        }

        /* === HEADER === */
        header {
            text-align: center;
            margin-bottom: 24px;
            padding: 48px 32px;
            background: #FFFFFF;
            border-radius: 20px;
            box-shadow: var(--shadow-md);
            border: 1px solid var(--border-color);
        }

        header h1 {
            font-size: 2.75em;
            margin-bottom: 12px;
            background: linear-gradient(135deg, var(--primary-color) 0%, var(--primary-light) 100%);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            background-clip: text;
            font-weight: 800;
            letter-spacing: -1px;
        }

        header p {
            font-size: 1.15em;
            color: var(--text-medium);
            font-weight: 400;
        }

        .badge {
            display: inline-block;
            padding: 6px 12px;
            background: var(--secondary-color);
            color: var(--primary-color);
            border-radius: 6px;
            font-size: 0.85em;
            font-weight: 600;
            margin-top: 12px;
        }

        /* === TAB NAVIGATION === */
        .tab-navigation {
            background: #FFFFFF;
            border-radius: 16px;
            padding: 8px;
            margin-bottom: 24px;
            box-shadow: var(--shadow-md);
            border: 1px solid var(--border-color);
            display: flex;
            gap: 8px;
        }

        .tab-btn {
            flex: 1;
            padding: 16px 24px;
            border: none;
            background: transparent;
            color: var(--text-medium);
            font-size: 1em;
            font-weight: 600;
            cursor: pointer;
            border-radius: 12px;
            transition: all 0.2s ease;
        }

        .tab-btn:hover {
            background: var(--bg-lighter);
            color: var(--text-dark);
        }

        .tab-btn.active {
            background: linear-gradient(135deg, var(--primary-color) 0%, var(--primary-light) 100%);
            color: white;
            box-shadow: 0 4px 8px rgba(15, 118, 110, 0.2);
        }

        /* === TAB CONTENT === */
        .tab-content {
            display: none;
        }

        .tab-content.active {
            display: block;
        }

        /* === MAIN GRID === */
        .main-content {
            display: grid;
            grid-template-columns: 380px 1fr;
            gap: 24px;
            margin-bottom: 20px;
            align-items: start;
        }

        @media (max-width: 1200px) {
            .main-content {
                grid-template-columns: 1fr;
            }
        }

        /* === CARD === */
        .card {
            background: rgba(255, 255, 255, 0.95);
            backdrop-filter: blur(10px);
            border-radius: 16px;
            padding: 28px;
            box-shadow: var(--shadow-md);
            border: 1px solid var(--border-color);
        }

        .card h2 {
            color: var(--primary-color);
            margin-bottom: 20px;
            font-size: 1.15em;
            font-weight: 700;
        }

        /* === FILE UPLOAD === */
        .drop-zone {
            border: 2px dashed #CBD5E1;
            border-radius: 12px;
            padding: 40px 20px;
            text-align: center;
            cursor: pointer;
            transition: all 0.3s ease;
            background: #F8FAFC;
        }

        .drop-zone:hover,
        .drop-zone.dragover {
            border-color: var(--primary-color);
            background: var(--secondary-color);
        }

        .file-input {
            display: none;
        }

        .upload-icon {
            font-size: 3em;
            margin-bottom: 10px;
        }

        /* === QUERY MODE SELECTOR === */
        .query-mode-selector {
            margin-bottom: 16px;
        }

        .query-mode-selector label {
            display: block;
            font-weight: 600;
            margin-bottom: 8px;
            color: var(--text-dark);
        }

        .query-mode-selector select {
            width: 100%;
            padding: 12px;
            border: 1px solid var(--border-color);
            border-radius: 8px;
            font-size: 0.95em;
            background: white;
            cursor: pointer;
        }

        .mode-description {
            margin-top: 8px;
            padding: 12px;
            background: var(--secondary-color);
            border-radius: 8px;
            font-size: 0.9em;
            color: var(--text-dark);
        }

        /* === CHAT === */
        .chat-messages {
            min-height: 400px;
            max-height: 600px;
            overflow-y: auto;
            padding: 20px;
            background: #F8FAFC;
            border-radius: 12px;
            margin-bottom: 16px;
        }

        .message {
            margin-bottom: 20px;
        }

        .message.user {
            text-align: right;
        }

        .message-content {
            display: inline-block;
            max-width: 80%;
            padding: 14px 18px;
            border-radius: 12px;
            word-wrap: break-word;
            font-size: 0.95em;
            line-height: 1.5;
        }

        .message.user .message-content {
            background: linear-gradient(135deg, var(--primary-color) 0%, var(--primary-light) 100%);
            color: white;
            border-bottom-right-radius: 6px;
        }

        .message.assistant .message-content {
            background: #FFFFFF;
            color: var(--text-dark);
            border: 1px solid var(--border-color);
            border-bottom-left-radius: 6px;
        }

        /* === MARKDOWN IN MESSAGES === */
        .message-content p {
            margin: 0 0 12px 0;
        }

        .message-content p:last-child {
            margin-bottom: 0;
        }

        .message-content ul,
        .message-content ol {
            margin: 8px 0 12px 20px;
        }

        .message-content li {
            margin: 4px 0;
        }

        .message-content strong {
            font-weight: 700;
            color: var(--primary-color);
        }

        .message-content code {
            background: var(--bg-lighter);
            padding: 2px 6px;
            border-radius: 4px;
            font-family: 'Monaco', 'Courier New', monospace;
            font-size: 0.9em;
        }

        .message-content pre {
            background: #1F2937;
            color: #F3F4F6;
            padding: 12px;
            border-radius: 8px;
            overflow-x: auto;
            margin: 12px 0;
        }

        /* === REFERENCES === */
        .references {
            margin-top: 12px;
            padding-top: 12px;
            border-top: 1px solid var(--border-color);
        }

        .references-title {
            font-size: 0.85em;
            font-weight: 600;
            color: var(--text-medium);
            margin-bottom: 8px;
        }

        .reference-item {
            font-size: 0.8em;
            color: var(--text-medium);
            padding: 4px 0;
        }

        /* === LOADING === */
        .typing-indicator {
            display: flex;
            align-items: center;
            gap: 6px;
            padding: 8px 0;
        }

        .typing-indicator span {
            height: 8px;
            width: 8px;
            background: var(--primary-color);
            border-radius: 50%;
            display: inline-block;
            animation: typing 1.4s infinite;
        }

        .typing-indicator span:nth-child(2) {
            animation-delay: 0.2s;
        }

        .typing-indicator span:nth-child(3) {
            animation-delay: 0.4s;
        }

        @keyframes typing {
            0%, 60%, 100% {
                transform: translateY(0);
                opacity: 0.7;
            }
            30% {
                transform: translateY(-10px);
                opacity: 1;
            }
        }

        /* === CHAT INPUT === */
        .chat-input-container {
            display: flex;
            gap: 12px;
        }

        .chat-input-container input {
            flex: 1;
            padding: 14px 18px;
            border: 1px solid var(--border-color);
            border-radius: 12px;
            font-size: 1em;
        }

        /* === BUTTONS === */
        .btn {
            padding: 12px 24px;
            border: none;
            border-radius: 8px;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.2s ease;
        }

        .btn-primary {
            background: linear-gradient(135deg, var(--primary-color) 0%, var(--primary-light) 100%);
            color: white;
        }

        .btn-primary:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 16px rgba(15, 118, 110, 0.3);
        }

        .btn-primary:disabled {
            opacity: 0.5;
            cursor: not-allowed;
            transform: none;
        }

        /* === STATS GRID === */
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 16px;
            margin-top: 20px;
        }

        .stat-card {
            background: var(--secondary-color);
            padding: 20px;
            border-radius: 12px;
            border-left: 4px solid var(--primary-color);
            text-align: center;
        }

        .stat-value {
            font-size: 2.5em;
            font-weight: 700;
            color: var(--primary-color);
        }

        .stat-label {
            font-size: 0.9em;
            color: var(--text-medium);
            margin-top: 8px;
        }

        /* === CODE BLOCKS === */
        .code-block-container {
            margin: 16px 0;
        }

        .code-block-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            background: #1F2937;
            padding: 12px 16px;
            border-radius: 12px 12px 0 0;
        }

        .code-language {
            color: #9CA3AF;
            font-size: 0.85em;
            font-weight: 600;
            text-transform: uppercase;
        }

        .copy-btn {
            background: #374151;
            color: white;
            border: none;
            padding: 8px 16px;
            border-radius: 8px;
            cursor: pointer;
            font-size: 0.85em;
            font-weight: 600;
            transition: all 0.2s ease;
        }

        .copy-btn:hover {
            background: var(--primary-color);
        }

        .code-block {
            background: #1F2937;
            color: #F3F4F6;
            padding: 20px;
            border-radius: 0 0 12px 12px;
            overflow-x: auto;
            font-family: 'Monaco', 'Courier New', monospace;
            font-size: 0.9em;
            line-height: 1.6;
        }

        .code-block pre {
            margin: 0;
            white-space: pre-wrap;
            word-wrap: break-word;
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>🧠 LightRAG Knowledge Graph Assistant</h1>
            <p>Upload documents and chat with AI-powered Knowledge Graph RAG</p>
            <span class="badge">Knowledge Graph Powered</span>
        </header>

        <!-- Tab Navigation -->
        <div class="tab-navigation">
            <button class="tab-btn active" onclick="switchTab('chat')">
                💬 Chat
            </button>
            <button class="tab-btn" onclick="switchTab('stats')">
                📊 Knowledge Graph
            </button>
            <button class="tab-btn" onclick="switchTab('api-docs')">
                🔌 API Docs
            </button>
        </div>

        <!-- Chat Tab -->
        <div id="chat-tab" class="tab-content active">
            <div class="main-content">
                <!-- Sidebar -->
                <div class="sidebar">
                    <!-- Upload -->
                    <div class="card">
                        <h2>📤 Upload Document</h2>
                        <div class="drop-zone" id="dropZone">
                            <div class="upload-icon">📁</div>
                            <p><strong>Drag & drop</strong> your file here</p>
                            <p style="font-size: 0.8em;">or click to browse</p>
                        </div>
                        <input type="file" id="fileInput" class="file-input">
                        <div id="uploadStatus"></div>
                    </div>

                    <!-- Query Mode Selector -->
                    <div class="card">
                        <h2>⚙️ Query Mode</h2>
                        <div class="query-mode-selector">
                            <label for="queryMode">Select Mode:</label>
                            <select id="queryMode" onchange="updateModeDescription()">
                                <option value="mix">Mix (Recommended)</option>
                                <option value="local">Local</option>
                                <option value="global">Global</option>
                                <option value="hybrid">Hybrid</option>
                                <option value="naive">Naive</option>
                                <option value="bypass">Bypass</option>
                            </select>
                            <div class="mode-description" id="modeDescription"></div>
                        </div>
                    </div>
                </div>

                <!-- Chat Section -->
                <div class="card chat-section">
                    <h2>💬 Chat with Your Knowledge Graph</h2>
                    <div class="chat-messages" id="chatMessages">
                        <div class="empty-state">Upload a document to start chatting</div>
                    </div>
                    <div class="chat-input-container">
                        <input type="text" id="chatInput" placeholder="Ask a question...">
                        <button class="btn btn-primary" id="sendBtn" onclick="sendMessage()">Send</button>
                    </div>
                </div>
            </div>
        </div>

        <!-- Stats Tab -->
        <div id="stats-tab" class="tab-content">
            <div class="card">
                <h2>📊 Knowledge Graph Statistics</h2>
                <div class="stats-grid" id="statsGrid">
                    <div class="stat-card">
                        <div class="stat-value" id="statDocs">-</div>
                        <div class="stat-label">Documents</div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-value" id="statEntities">-</div>
                        <div class="stat-label">Entities</div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-value" id="statRelations">-</div>
                        <div class="stat-label">Relationships</div>
                    </div>
                    <div class="stat-card">
                        <div class="stat-value" id="statChunks">-</div>
                        <div class="stat-label">Text Chunks</div>
                    </div>
                </div>
                <div id="storageInfo" style="margin-top: 24px;"></div>
            </div>
        </div>

        <!-- API Docs Tab -->
        <div id="api-docs-tab" class="tab-content">
            <div class="card">
                <h2>🔌 API Documentation</h2>
                <div id="apiDocsContainer">
                    <p>Loading API documentation...</p>
                </div>
            </div>
        </div>
    </div>

    <script>
        // Query mode descriptions
        const modeDescriptions = {
            'mix': 'Integrates knowledge graph + vector retrieval. Best for most use cases.',
            'local': 'Entity-focused retrieval. Best for specific questions about entities.',
            'global': 'Pattern analysis across knowledge graph. Best for broad questions.',
            'hybrid': 'Combines local and global. Best for complex questions.',
            'naive': 'Simple vector search. Best for testing or when KG not needed.',
            'bypass': 'Direct LLM query without retrieval. Best for general questions.'
        };

        // Update mode description
        function updateModeDescription() {
            const mode = document.getElementById('queryMode').value;
            document.getElementById('modeDescription').textContent = modeDescriptions[mode];
        }
        updateModeDescription();

        // File upload
        const dropZone = document.getElementById('dropZone');
        const fileInput = document.getElementById('fileInput');

        dropZone.addEventListener('click', () => fileInput.click());
        dropZone.addEventListener('dragover', (e) => {
            e.preventDefault();
            dropZone.classList.add('dragover');
        });
        dropZone.addEventListener('dragleave', () => {
            dropZone.classList.remove('dragover');
        });
        dropZone.addEventListener('drop', (e) => {
            e.preventDefault();
            dropZone.classList.remove('dragover');
            if (e.dataTransfer.files.length > 0) {
                handleFile(e.dataTransfer.files[0]);
            }
        });
        fileInput.addEventListener('change', (e) => {
            if (e.target.files.length > 0) {
                handleFile(e.target.files[0]);
            }
        });

        async function handleFile(file) {
            const formData = new FormData();
            formData.append('file', file);

            try {
                const response = await fetch('/upload', {
                    method: 'POST',
                    body: formData
                });

                const data = await response.json();

                if (response.ok) {
                    alert('✅ File uploaded successfully!');
                    loadStats();
                } else {
                    alert('❌ Error: ' + data.detail);
                }
            } catch (error) {
                alert('❌ Error: ' + error.message);
            }
        }

        // Send message with streaming
        async function sendMessage() {
            const input = document.getElementById('chatInput');
            const message = input.value.trim();
            if (!message) return;

            const mode = document.getElementById('queryMode').value;

            addMessage('user', message);
            input.value = '';
            document.getElementById('sendBtn').disabled = true;

            // Show loading
            const loadingDiv = document.createElement('div');
            loadingDiv.className = 'message assistant loading';
            loadingDiv.id = 'loading-message';
            loadingDiv.innerHTML = `
                <div class="message-content">
                    <div class="typing-indicator">
                        <span></span>
                        <span></span>
                        <span></span>
                    </div>
                </div>
            `;
            document.getElementById('chatMessages').appendChild(loadingDiv);
            document.getElementById('chatMessages').scrollTop = document.getElementById('chatMessages').scrollHeight;

            try {
                const response = await fetch('/query/stream', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        query: message,
                        mode: mode,
                        stream: true,
                        include_references: true
                    })
                });

                const reader = response.body.getReader();
                const decoder = new TextDecoder();
                let buffer = '';
                let references = null;
                let fullResponse = '';

                // Remove loading
                const loading = document.getElementById('loading-message');
                if (loading) loading.remove();

                while (true) {
                    const { value, done } = await reader.read();
                    if (done) break;

                    buffer += decoder.decode(value, { stream: true });
                    const lines = buffer.split('\n');
                    buffer = lines.pop();

                    for (const line of lines) {
                        if (line.trim()) {
                            const data = JSON.parse(line);

                            if (data.references) {
                                references = data.references;
                            }
                            if (data.response) {
                                fullResponse += data.response;
                                updateStreamingMessage(fullResponse, references);
                            }
                            if (data.error) {
                                alert('Error: ' + data.error);
                            }
                        }
                    }
                }

            } catch (error) {
                const loading = document.getElementById('loading-message');
                if (loading) loading.remove();
                alert('Error: ' + error.message);
            } finally {
                document.getElementById('sendBtn').disabled = false;
            }
        }

        function updateStreamingMessage(content, references) {
            let msgDiv = document.getElementById('streaming-message');
            if (!msgDiv) {
                msgDiv = document.createElement('div');
                msgDiv.className = 'message assistant';
                msgDiv.id = 'streaming-message';
                document.getElementById('chatMessages').appendChild(msgDiv);
            }

            const formattedContent = typeof marked !== 'undefined'
                ? marked.parse(content)
                : content;

            let html = `<div class="message-content">${formattedContent}`;

            if (references && references.length > 0) {
                html += `<div class="references">
                    <div class="references-title">📚 Sources:</div>`;
                references.forEach((ref, idx) => {
                    html += `<div class="reference-item">[${idx + 1}] ${ref.file_path}</div>`;
                });
                html += `</div>`;
            }

            html += `</div>`;
            msgDiv.innerHTML = html;
            document.getElementById('chatMessages').scrollTop = document.getElementById('chatMessages').scrollHeight;
        }

        function addMessage(role, content, references = null) {
            const msgDiv = document.createElement('div');
            msgDiv.className = `message ${role}`;

            const formattedContent = role === 'assistant' && typeof marked !== 'undefined'
                ? marked.parse(content)
                : content;

            let html = `<div class="message-content">${formattedContent}`;

            if (references && references.length > 0) {
                html += `<div class="references">
                    <div class="references-title">📚 Sources:</div>`;
                references.forEach((ref, idx) => {
                    html += `<div class="reference-item">[${idx + 1}] ${ref.file_path}</div>`;
                });
                html += `</div>`;
            }

            html += `</div>`;
            msgDiv.innerHTML = html;

            document.getElementById('chatMessages').appendChild(msgDiv);
            document.getElementById('chatMessages').scrollTop = document.getElementById('chatMessages').scrollHeight;
        }

        // Tab switching
        function switchTab(tabName) {
            document.querySelectorAll('.tab-content').forEach(tab => {
                tab.classList.remove('active');
            });
            document.querySelectorAll('.tab-btn').forEach(btn => {
                btn.classList.remove('active');
            });

            document.getElementById(`${tabName}-tab`).classList.add('active');
            event.target.classList.add('active');

            if (tabName === 'stats') {
                loadStats();
            } else if (tabName === 'api-docs') {
                loadApiDocs();
            }
        }

        // Load stats
        async function loadStats() {
            try {
                const response = await fetch('/stats');
                const data = await response.json();

                document.getElementById('statDocs').textContent = data.total_documents;
                document.getElementById('statEntities').textContent = data.total_entities;
                document.getElementById('statRelations').textContent = data.total_relationships;
                document.getElementById('statChunks').textContent = data.total_chunks;

                const storageInfo = document.getElementById('storageInfo');
                storageInfo.innerHTML = `
                    <h3 style="margin-bottom: 12px;">Storage Configuration</h3>
                    <p><strong>Working Directory:</strong> ${data.storage_info.working_dir}</p>
                    <p><strong>LLM:</strong> ${data.storage_info.llm_binding} - ${data.storage_info.llm_model}</p>
                    <p><strong>Embeddings:</strong> ${data.storage_info.embedding_binding} - ${data.storage_info.embedding_model}</p>
                `;
            } catch (error) {
                console.error('Error loading stats:', error);
            }
        }

        // Load API docs
        async function loadApiDocs() {
            try {
                const response = await fetch('/api-info');
                const data = await response.json();

                const container = document.getElementById('apiDocsContainer');
                container.innerHTML = generateApiDocs(data);
                attachCopyListeners();
            } catch (error) {
                console.error('Error loading API docs:', error);
            }
        }

        function generateApiDocs(data) {
            return `
                <h3>Query Modes</h3>
                ${data.query_modes.map(mode => `
                    <div style="margin: 16px 0; padding: 16px; background: var(--secondary-color); border-radius: 8px;">
                        <strong>${mode.name}</strong>: ${mode.description}<br>
                        <em>Best for: ${mode.best_for}</em>
                    </div>
                `).join('')}

                <h3 style="margin-top: 32px;">cURL Example: Streaming Query</h3>
                <div class="code-block-container">
                    <div class="code-block-header">
                        <span class="code-language">cURL</span>
                        <button class="copy-btn" data-copy-target="curl-example">📋 Copy</button>
                    </div>
                    <div class="code-block" id="curl-example">
<pre>curl -X POST "http://localhost:8000/query/stream" \\
  -H "Content-Type: application/json" \\
  -d '{
    "query": "What are the main topics in my documents?",
    "mode": "mix",
    "stream": true,
    "include_references": true
  }'</pre>
                    </div>
                </div>

                <h3 style="margin-top: 32px;">Python Example</h3>
                <div class="code-block-container">
                    <div class="code-block-header">
                        <span class="code-language">Python</span>
                        <button class="copy-btn" data-copy-target="python-example">📋 Copy</button>
                    </div>
                    <div class="code-block" id="python-example">
<pre>import requests
import json

# Query with streaming
response = requests.post(
    "http://localhost:8000/query/stream",
    json={
        "query": "What are the main topics?",
        "mode": "mix",
        "stream": True,
        "include_references": True
    },
    stream=True
)

for line in response.iter_lines():
    if line:
        data = json.loads(line)
        if "references" in data:
            print("References:", data["references"])
        if "response" in data:
            print(data["response"], end="", flush=True)</pre>
                    </div>
                </div>
            `;
        }

        function attachCopyListeners() {
            document.querySelectorAll('.copy-btn').forEach(btn => {
                btn.addEventListener('click', function() {
                    const targetId = this.getAttribute('data-copy-target');
                    const codeBlock = document.getElementById(targetId);
                    const text = codeBlock.textContent.trim();

                    navigator.clipboard.writeText(text).then(() => {
                        const original = this.innerHTML;
                        this.innerHTML = '✅ Copied!';
                        setTimeout(() => {
                            this.innerHTML = original;
                        }, 2000);
                    });
                });
            });
        }

        // Enter key support
        document.getElementById('chatInput').addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });

        // Load initial stats
        loadStats();
    </script>
</body>
</html>
```

## Environment Configuration (.env)

```bash
###########################################################################################
### LightRAG Web Application Configuration
###########################################################################################

###########################
### Server Configuration
###########################
HOST=0.0.0.0
PORT=8000

### Directory Configuration
WORKING_DIR=./rag_storage
INPUT_DIR=./inputs

### Logging
LOG_LEVEL=INFO

###########################
### LLM Configuration
### Choose one: openai, ollama, azure_openai, gemini, etc.
###########################
LLM_BINDING=openai
LLM_MODEL=gpt-4o-mini
LLM_BINDING_HOST=https://api.openai.com/v1
LLM_BINDING_API_KEY=your_openai_api_key_here
LLM_TIMEOUT=180

###########################
### Embedding Configuration
### IMPORTANT: Do NOT change after processing first file!
###########################
EMBEDDING_BINDING=openai
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIM=1536
EMBEDDING_BINDING_HOST=https://api.openai.com/v1
EMBEDDING_BINDING_API_KEY=your_openai_api_key_here
EMBEDDING_TIMEOUT=60

###########################
### Query Configuration
###########################
ENABLE_LLM_CACHE=true
TOP_K=40
CHUNK_TOP_K=20
MAX_ENTITY_TOKENS=6000
MAX_RELATION_TOKENS=8000
MAX_TOTAL_TOKENS=30000

###########################
### Document Processing
###########################
CHUNK_SIZE=1200
CHUNK_OVERLAP_SIZE=100
ENABLE_LLM_CACHE_FOR_EXTRACT=true
MAX_PARALLEL_INSERT=2

###########################
### Reranking (Optional)
###########################
# RERANK_BINDING=jina
# RERANK_MODEL=jina-reranker-v2-base-multilingual
# RERANK_BINDING_API_KEY=your_jina_api_key_here

###########################
### Storage Configuration
### Default: Local files (NetworkX + NanoVectorDB)
### For production: Use Neo4j + PostgreSQL (see .env.unraid.example)
###########################
# LIGHTRAG_GRAPH_STORAGE=NetworkXStorage
# LIGHTRAG_VECTOR_STORAGE=NanoVectorDBStorage
# LIGHTRAG_KV_STORAGE=JsonKVStorage
# LIGHTRAG_DOC_STATUS_STORAGE=JsonDocStatusStorage
```

## Features Comparison

### ✅ Features LightRAG CAN Implement (vs Gemini RAG)

1. **Knowledge Graph RAG** ⭐ - MAJOR ADVANTAGE
   - Automatic entity extraction
   - Relationship mapping
   - Graph-based retrieval

2. **Multiple Query Modes** ⭐ - MAJOR ADVANTAGE
   - local, global, hybrid, mix, naive, bypass
   - Optimized for different question types

3. **Streaming Responses** ✓
   - NDJSON format
   - Real-time token delivery

4. **Citation/Reference Tracking** ✓
   - Source file attribution
   - Reference IDs

5. **Conversation History** ✓
   - Multi-turn conversations
   - Context maintenance

6. **Multiple Storage Backends** ⭐
   - Neo4j, PostgreSQL, MongoDB, etc.
   - Production-ready scaling

7. **Reranking Support** ⭐
   - Cohere, Jina, Aliyun rerankers
   - Improved retrieval accuracy

8. **Token Budget Control** ⭐
   - Fine-grained control over context size
   - Separate budgets for entities, relations, chunks

9. **Direct Text Insertion** ⭐
   - No file upload required
   - Programmatic document insertion

10. **Raw Data Retrieval** ⭐
    - `/query/data` endpoint
    - Access to entities, relationships, chunks

11. **Knowledge Graph Visualization** ⭐
    - Export to Neo4j Browser
    - HTML graph visualization

### ❌ Features LightRAG CANNOT Implement (vs Gemini)

1. **Custom Metadata Per File** ❌
   - Gemini allows custom metadata (tags, categories, etc.) per file
   - LightRAG extracts metadata automatically but doesn't support user-defined metadata per file
   - **Workaround**: Encode metadata in filenames or document content

2. **Custom Chunking Config Per File** ❌
   - Gemini allows different chunking strategies per file
   - LightRAG uses global chunking configuration
   - **Workaround**: Process different file types in separate LightRAG instances

3. **Metadata Filtering** ❌
   - Gemini supports filtering queries by metadata
   - LightRAG doesn't have built-in metadata filtering
   - **Workaround**: Use different workspaces for different document categories

### ⭐ Additional LightRAG Features (Not in Gemini)

1. **Knowledge Graph** - Entities and relationships extracted from documents
2. **Multiple Query Modes** - Optimized strategies for different question types
3. **Graph Visualization** - Interactive knowledge graph exploration
4. **Entity Merging** - Automatic deduplication of similar entities
5. **Custom Entity Types** - Define domain-specific entity categories
6. **Token Budget Management** - Fine-grained control over context allocation
7. **Reranking** - Optional semantic reranking of results
8. **Multiple Storage Backends** - Neo4j, Postgres, MongoDB, etc.
9. **Ollama Support** - Use local LLMs (Llama, Mistral, etc.)
10. **Raw Data Access** - Direct access to knowledge graph data

## Setup Instructions

### 1. Install Dependencies

```bash
# LightRAG is already installed in this repository
pip install fastapi uvicorn python-dotenv python-multipart aiofiles
```

### 2. Configure Environment

```bash
# Copy example env file
cp .env.unraid.example .env

# Edit .env and fill in:
# - Your OpenAI API key (or Ollama configuration)
# - Embedding API key
# - (Optional) Neo4j/Postgres credentials for production
```

### 3. Create Templates Directory

```bash
mkdir -p templates
# Place the index.html file in templates/
```

### 4. Run the Application

```bash
python app.py
```

### 5. Access the Application

Open `http://localhost:8000` in your browser.

## Production Deployment

For production, use the existing LightRAG Docker setup with Neo4j + PostgreSQL:

```bash
# Use the existing docker-compose configuration
# Update .env with production settings
docker-compose up -d
```

## Key Differences from Gemini RAG

| Feature | Gemini RAG | LightRAG |
|---------|-----------|----------|
| RAG Approach | Vector-only | Knowledge Graph + Vector |
| Query Modes | Single mode | 6 modes (local, global, hybrid, mix, naive, bypass) |
| Entity Extraction | No | Yes - automatic |
| Relationship Mapping | No | Yes - automatic |
| Custom Metadata | Yes - per file | No - global only |
| Chunking Config | Per file | Global |
| Metadata Filtering | Yes | No |
| Storage Backends | Google-managed | Multiple (Neo4j, Postgres, etc.) |
| Reranking | No | Yes - optional |
| Knowledge Graph Viz | No | Yes |
| Local LLM Support | No | Yes - Ollama |
| Raw Data Access | Limited | Full access to entities/relations |

## When to Use LightRAG vs Gemini

**Use LightRAG when:**
- You need knowledge graph capabilities
- You want entity/relationship extraction
- You need multiple query strategies
- You want self-hosted/local deployment
- You need production-grade storage (Neo4j, Postgres)
- You want to use local LLMs (Ollama)

**Use Gemini RAG when:**
- You need per-file custom metadata
- You need per-file chunking strategies
- You want fully managed infrastructure
- You need metadata-based filtering
- You prefer Google ecosystem integration

## This mega-prompt contains EVERYTHING needed to build a complete LightRAG web application with all features!
