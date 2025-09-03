# RAG Chatbot Query Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 FRONTEND (script.js)                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  1. User types query in chatInput                                                  │
│  2. sendMessage() triggered                                                        │
│  3. UI disabled, loading animation shown                                           │
│  4. POST /api/query with {query, session_id}                                       │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              FASTAPI BACKEND (app.py)                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  5. @app.post("/api/query") endpoint receives request                              │
│  6. Create session_id if not provided                                              │
│  7. Call rag_system.query(request.query, session_id)                               │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            RAG SYSTEM (rag_system.py)                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  8. Build prompt: "Answer this question about course materials: {query}"           │
│  9. Get conversation history from session_manager                                  │
│ 10. Call ai_generator.generate_response() with tools                               │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          AI GENERATOR (ai_generator.py)                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 11. Build system prompt with conversation context                                  │
│ 12. Call Anthropic Claude API with tools available                                 │
│ 13. Claude decides: Use general knowledge OR search course content                 │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
                    ┌─────────────┐
                    │   CLAUDE    │ ──── General Knowledge? ──── Return Direct Answer
                    │  DECISION   │
                    └─────────────┘
                          │
                    Need to search?
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         TOOL EXECUTION (search_tools.py)                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 14. tool_manager.execute_tool("search_course_content", query, course_name, ...)    │
│ 15. CourseSearchTool.execute() called                                              │
│ 16. Call vector_store.search() with parameters                                     │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         VECTOR STORE (vector_store.py)                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 17. Embed query using all-MiniLM-L6-v2                                             │
│ 18. Search ChromaDB with filters (course_name, lesson_number)                      │
│ 19. Return top 5 relevant chunks with metadata                                     │
│ 20. SearchResults(documents, metadata, distances)                                  │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      RESULT FORMATTING (search_tools.py)                           │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 21. Format results: "[Course Title - Lesson X]\n{content}"                         │
│ 22. Track sources: ["Course Title - Lesson X", ...]                               │
│ 23. Return formatted string to AI Generator                                        │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                    FINAL AI RESPONSE (ai_generator.py)                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 24. Send tool results back to Claude                                               │
│ 25. Claude generates final answer based on search results                          │
│ 26. Return response text                                                           │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                       RESPONSE ASSEMBLY (rag_system.py)                            │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 27. Get sources from tool_manager.get_last_sources()                               │
│ 28. Update conversation history with session_manager                               │
│ 29. Return (response, sources) tuple                                               │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         API RESPONSE (app.py)                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 30. Create QueryResponse(answer, sources, session_id)                              │
│ 31. Return JSON response to frontend                                               │
└─────────────────────────┬───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        FRONTEND DISPLAY (script.js)                                │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ 32. Remove loading animation                                                       │
│ 33. Convert markdown to HTML with marked.parse()                                   │
│ 34. Display answer with collapsible sources section                                │
│ 35. Re-enable UI, scroll to bottom                                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════════════

                                    DATA FLOW

Frontend Query ──→ FastAPI ──→ RAG System ──→ AI Generator ──→ Claude API
                                    │              │              │
                                    │              │              ▼
                                    │              │         Tool Decision
                                    │              │              │
                                    │              │              ▼
                                    │              │         Search Tool ──→ Vector Store
                                    │              │              │              │
                                    │              │              │              ▼
                                    │              │              │         ChromaDB Search
                                    │              │              │              │
                                    │              │              │              ▼
                                    │              │              ◄──────── Search Results
                                    │              │              │
                                    │              │              ▼
                                    │              ◄────── Formatted Results
                                    │              │
                                    │              ▼
                                    ◄───── Final AI Response + Sources
                                    │
                                    ▼
Frontend Display ◄─── JSON Response

═══════════════════════════════════════════════════════════════════════════════════════

                                 KEY COMPONENTS

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Session       │  │   Document      │  │   Vector        │  │   Search        │
│   Manager       │  │   Processor     │  │   Store         │  │   Tools         │
│                 │  │                 │  │                 │  │                 │
│ • Conversation  │  │ • Text Chunking │  │ • ChromaDB      │  │ • Course Search │
│   History       │  │ • Metadata      │  │ • Embeddings    │  │ • Tool Manager  │
│ • Session IDs   │  │   Extraction    │  │ • Semantic      │  │ • Source        │
│                 │  │ • Lesson        │  │   Search        │  │   Tracking      │
│                 │  │   Parsing       │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘

═══════════════════════════════════════════════════════════════════════════════════════

                               TIMING & PERFORMANCE

Step 1-4:   Frontend Processing     ~50ms    (UI updates, network request)
Step 5-7:   FastAPI Routing        ~10ms    (request validation, routing)
Step 8-10:  RAG System Setup       ~20ms    (session lookup, prompt building)
Step 11-13: Initial Claude Call    ~800ms   (API call to Anthropic)
Step 14-20: Tool Execution         ~200ms   (vector search, formatting)
Step 21-26: Final Claude Call      ~600ms   (API call with search results)
Step 27-31: Response Assembly      ~30ms    (source collection, JSON response)
Step 32-35: Frontend Display       ~50ms    (DOM updates, markdown parsing)

Total Time: ~1.8-2.5 seconds (typical query requiring search)
           ~1.0-1.5 seconds (general knowledge query, no search)
```
