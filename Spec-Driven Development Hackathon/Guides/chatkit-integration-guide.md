# ChatKit Integration Guide

> **A comprehensive guide for integrating OpenAI ChatKit into any application with Python FastAPI backend, React frontend, and Neon PostgreSQL database.**

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Summary](#architecture-summary)
3. [Dependencies & Installation](#dependencies--installation)
4. [Backend Implementation](#backend-implementation)
   - [Database Schema](#database-schema)
   - [Database Connection Service](#database-connection-service)
   - [ChatKit Store (In-Memory)](#chatkit-store-in-memory)
   - [ChatKit Store (PostgreSQL)](#chatkit-store-postgresql)
   - [ChatKit Server & Routes](#chatkit-server--routes)
   - [Authentication Dependencies](#authentication-dependencies)
5. [Frontend Implementation](#frontend-implementation)
   - [Auth Client Setup](#auth-client-setup)
   - [ChatBot Component](#chatbot-component)
   - [Styling](#styling)
   - [Configuration](#configuration)
6. [Environment Variables](#environment-variables)
7. [Features & Capabilities](#features--capabilities)
8. [API Endpoints](#api-endpoints)
9. [Complete Flow Diagram](#complete-flow-diagram)

---

## Overview

This guide documents how to integrate **OpenAI ChatKit** into a full-stack application. ChatKit provides a standardized, streaming chat interface with built-in:

- **Thread management** (conversations)
- **Real-time streaming** via Server-Sent Events (SSE)
- **Persistent storage** (memory or database)
- **Authentication integration**
- **RAG (Retrieval-Augmented Generation)** support

### Tech Stack Requirements

| Layer | Technology | Version |
|-------|------------|---------|
| Backend | Python FastAPI | >= 0.124.4 |
| Frontend | React | >= 19.0.0 |
| Database | PostgreSQL (Neon) | Any |
| LLM | Gemini/OpenAI via LiteLLM | - |
| Python | - | >= 3.13 |

---

## Architecture Summary

```mermaid
graph TB
    subgraph Frontend [React Frontend]
        CB[ChatBot.tsx]
        AC[AuthClient]
        LS[localStorage]
    end

    subgraph Backend [FastAPI Backend]
        CR[chatkit_routes.py]
        CKS[ChatKitServer]
        QS[QueryService - RAG]
        PS[PostgresStore]
        DB[(Neon PostgreSQL)]
    end

    CB -->|POST /chatkit| CR
    CB -->|Bearer Token| CR
    CR --> CKS
    CKS --> PS
    PS --> DB
    CKS --> QS
    QS -->|Qdrant + Cohere| VDB[(Vector DB)]
    CB <-->|thread_id| LS
    CB <--> AC
```

---

## Dependencies & Installation

### Backend Dependencies

Add to your `pyproject.toml` or `requirements.txt`:

```toml
# pyproject.toml
[project]
dependencies = [
    "asyncpg>=0.31.0",                    # PostgreSQL async driver
    "fastapi[standard]>=0.124.4",         # FastAPI framework
    "openai>=2.12.0",                     # OpenAI SDK
    "openai-agents[litellm]>=0.6.3",      # Agent framework with LiteLLM
    "openai-chatkit>=1.4.0",              # ChatKit server library
    "python-dotenv>=1.2.1",               # Environment variables
    "pyjwt[crypto]>=2.10.1",              # JWT handling
    # Optional for RAG:
    "qdrant-client>=1.16.2",              # Vector database
    "cohere>=5.20.0",                     # Embeddings
]
```

**Install with uv:**
```bash
cd backend
uv add "openai-chatkit>=1.4.0" "openai-agents[litellm]>=0.6.3"
```

### Frontend Dependencies

Add to your `package.json`:

```json
{
  "dependencies": {
    "@openai/chatkit-react": "^1.4.0",
    "better-auth": "^1.4.7",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-icons": "^5.5.0"
  }
}
```

**Install with npm:**
```bash
cd frontend
npm install @openai/chatkit-react better-auth react-icons
```

---

## Backend Implementation

### Database Schema

Create the ChatKit database tables in PostgreSQL. Save this as `scripts/init_chat_db.py`:

```python
import asyncio
import os
import sys
sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from services.db import Database

async def init_chat_db():
    print("Initializing ChatKit Database Schema...")

    db = Database()
    await db.connect()

    try:
        # Create Threads Table
        print("Creating chatkit_threads table...")
        await db.execute("""
            CREATE TABLE IF NOT EXISTS chatkit_threads (
                id TEXT PRIMARY KEY,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                metadata JSONB DEFAULT '{}'::jsonb,
                user_id TEXT
            );
        """)

        # Create Items Table
        print("Creating chatkit_items table...")
        await db.execute("""
            CREATE TABLE IF NOT EXISTS chatkit_items (
                id TEXT PRIMARY KEY,
                thread_id TEXT REFERENCES chatkit_threads(id) ON DELETE CASCADE,
                type TEXT NOT NULL,
                content JSONB DEFAULT '{}'::jsonb,
                created_at TIMESTAMPTZ DEFAULT NOW()
            );
        """)

        # Create Indexes for performance
        print("Creating indexes...")
        await db.execute("CREATE INDEX IF NOT EXISTS idx_threads_user_id ON chatkit_threads(user_id);")
        await db.execute("CREATE INDEX IF NOT EXISTS idx_items_thread_id ON chatkit_items(thread_id);")
        await db.execute("CREATE INDEX IF NOT EXISTS idx_items_created_at ON chatkit_items(created_at);")

        print("ChatKit Database Schema initialized successfully.")

    except Exception as e:
        print(f"Error initializing database: {e}")
    finally:
        await db.disconnect()

if __name__ == "__main__":
    if sys.platform == "win32":
        asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())
    asyncio.run(init_chat_db())
```

**Run the migration:**
```bash
python scripts/init_chat_db.py
```

---

### Database Connection Service

Create `services/db.py` for async PostgreSQL connection pooling:

```python
import os
from typing import Optional
from dotenv import load_dotenv
load_dotenv()
import asyncpg

DATABASE_URL = os.getenv("DATABASE_URL")

class Database:
    _pool: Optional[asyncpg.Pool] = None

    @classmethod
    async def connect(cls):
        if cls._pool is None:
            if not DATABASE_URL:
                raise ValueError("DATABASE_URL environment variable is not set")
            cls._pool = await asyncpg.create_pool(DATABASE_URL)
            print("Connected to Neon Postgres")

    @classmethod
    async def disconnect(cls):
        if cls._pool:
            await cls._pool.close()
            cls._pool = None
            print("Disconnected from Neon Postgres")

    @classmethod
    def get_pool(cls) -> asyncpg.Pool:
        if cls._pool is None:
            raise RuntimeError("Database not connected. Call connect() first.")
        return cls._pool

    @classmethod
    async def execute(cls, query: str, *args):
        pool = cls.get_pool()
        async with pool.acquire() as conn:
            return await conn.execute(query, *args)

    @classmethod
    async def fetch(cls, query: str, *args):
        pool = cls.get_pool()
        async with pool.acquire() as conn:
            return await conn.fetch(query, *args)

    @classmethod
    async def fetchrow(cls, query: str, *args):
        pool = cls.get_pool()
        async with pool.acquire() as conn:
            return await conn.fetchrow(query, *args)
```

---

### ChatKit Store (In-Memory)

For development/MVP, create `services/chatkit_store.py`:

```python
import uuid
from datetime import datetime, timezone
from typing import Any, List
from dataclasses import dataclass, field

from chatkit.store import Store
from chatkit.types import ThreadMetadata, ThreadItem, Page

@dataclass
class ThreadState:
    thread: ThreadMetadata
    items: list[ThreadItem] = field(default_factory=list)

class MemoryStore(Store[dict]):
    """Thread-safe in-memory store matching official ChatKit implementation"""

    def __init__(self) -> None:
        self._threads: dict[str, ThreadState] = {}
        self._attachments: dict[str, Any] = {}

    def generate_thread_id(self, context: dict) -> str:
        return f"thread_{uuid.uuid4().hex[:12]}"

    def generate_item_id(self, item_type: str, thread: ThreadMetadata, context: dict) -> str:
        return f"{item_type}_{uuid.uuid4().hex[:12]}"

    def _get_items(self, thread_id: str) -> list[ThreadItem]:
        state = self._threads.get(thread_id)
        return state.items if state else []

    async def load_thread(self, thread_id: str, context: dict) -> ThreadMetadata:
        state = self._threads.get(thread_id)
        if state:
            return state.thread.model_copy(deep=True)

        # Create new thread with user_id from context
        user_id = context.get("user_id")
        thread = ThreadMetadata(
            id=thread_id,
            created_at=datetime.now(timezone.utc),
            metadata={"user_id": user_id}
        )
        self._threads[thread_id] = ThreadState(thread=thread.model_copy(deep=True), items=[])
        return thread

    async def save_thread(self, thread: ThreadMetadata, context: dict) -> None:
        state = self._threads.get(thread.id)
        if state:
            # Preserve user_id if not in update
            if "user_id" in state.thread.metadata and "user_id" not in thread.metadata:
                thread.metadata["user_id"] = state.thread.metadata["user_id"]
            state.thread = thread.model_copy(deep=True)
        else:
            user_id = context.get("user_id")
            if "user_id" not in thread.metadata:
                thread.metadata["user_id"] = user_id
            self._threads[thread.id] = ThreadState(thread=thread.model_copy(deep=True), items=[])

    async def load_thread_items(
        self, thread_id: str, after: str | None, limit: int, order: str, context: dict
    ) -> Page[ThreadItem]:
        items = [item.model_copy(deep=True) for item in self._get_items(thread_id)]
        items.sort(
            key=lambda i: getattr(i, "created_at", datetime.now(timezone.utc)),
            reverse=(order == "desc"),
        )

        start = 0
        if after:
            index_map = {item.id: idx for idx, item in enumerate(items)}
            start = index_map.get(after, -1) + 1

        slice_items = items[start: start + limit + 1]
        has_more = len(slice_items) > limit

        return Page(
            data=slice_items[:limit],
            has_more=has_more,
            after=slice_items[limit].id if has_more else None
        )

    async def add_thread_item(self, thread_id: str, item: ThreadItem, context: dict) -> None:
        state = self._threads.get(thread_id)
        if not state:
            await self.load_thread(thread_id, context)
            state = self._threads[thread_id]

        # Update if exists, else append
        for i, existing in enumerate(state.items):
            if existing.id == item.id:
                state.items[i] = item.model_copy(deep=True)
                return
        state.items.append(item.model_copy(deep=True))

    async def save_item(self, thread_id: str, item: ThreadItem, context: dict) -> None:
        await self.add_thread_item(thread_id, item, context)

    async def load_item(self, thread_id: str, item_id: str, context: dict) -> ThreadItem:
        for item in self._get_items(thread_id):
            if item.id == item_id:
                return item.model_copy(deep=True)
        raise ValueError(f"Item {item_id} not found")

    async def delete_thread_item(self, thread_id: str, item_id: str, context: dict) -> None:
        state = self._threads.get(thread_id)
        if state:
            state.items = [i for i in state.items if i.id != item_id]

    async def load_threads(self, limit: int, after: str | None, order: str, context: dict) -> Page[ThreadMetadata]:
        user_id = context.get("user_id")
        matching = [
            s.thread.model_copy(deep=True)
            for s in self._threads.values()
            if s.thread.metadata.get("user_id") == user_id
        ]
        return Page(data=matching[-limit:], has_more=False)

    async def delete_thread(self, thread_id: str, context: dict) -> None:
        self._threads.pop(thread_id, None)

    async def save_attachment(self, attachment: Any, context: dict) -> None:
        self._attachments[attachment.id] = attachment

    async def load_attachment(self, attachment_id: str, context: dict) -> Any:
        if attachment_id not in self._attachments:
            raise ValueError(f"Attachment {attachment_id} not found")
        return self._attachments[attachment_id]

    async def delete_attachment(self, attachment_id: str, context: dict) -> None:
        self._attachments.pop(attachment_id, None)
```

---

### ChatKit Store (PostgreSQL)

For production persistence, create `services/chatkit_postgres_store.py`:

```python
import json
from datetime import datetime, timezone
from typing import Any
from pydantic import TypeAdapter
from chatkit.store import Store
from chatkit.types import ThreadMetadata, ThreadItem, Page
from services.db import Database

class PostgresStore(Store[dict]):
    """
    Postgres-backed store for ChatKit.
    Persists threads and items to 'chatkit_threads' and 'chatkit_items' tables.
    """

    def __init__(self):
        self._attachments = {}  # Stub memory store for attachments

    async def load_thread(self, thread_id: str, context: dict) -> ThreadMetadata:
        row = await Database.fetchrow("SELECT * FROM chatkit_threads WHERE id = $1", thread_id)

        if row:
            metadata = json.loads(row["metadata"])
            return ThreadMetadata(
                id=row["id"],
                created_at=row["created_at"],
                metadata=metadata,
                title=metadata.get("title")
            )

        # Create new thread if not found
        user_id = context.get("user_id")
        created_at = datetime.now(timezone.utc)
        metadata = {"user_id": user_id}

        await Database.execute(
            """
            INSERT INTO chatkit_threads (id, created_at, metadata, user_id)
            VALUES ($1, $2, $3, $4)
            """,
            thread_id, created_at, json.dumps(metadata), user_id
        )

        return ThreadMetadata(
            id=thread_id,
            created_at=created_at,
            metadata=metadata
        )

    async def save_thread(self, thread: ThreadMetadata, context: dict) -> None:
        existing_row = await Database.fetchrow(
            "SELECT metadata, user_id FROM chatkit_threads WHERE id = $1", thread.id
        )

        if existing_row:
            existing_metadata = json.loads(existing_row["metadata"])
            if "user_id" in existing_metadata and "user_id" not in thread.metadata:
                thread.metadata["user_id"] = existing_metadata["user_id"]
        elif "user_id" not in thread.metadata:
            thread.metadata["user_id"] = context.get("user_id")

        metadata_json = json.dumps(thread.metadata)
        owner_id = thread.metadata.get("user_id")

        await Database.execute(
            """
            INSERT INTO chatkit_threads (id, created_at, metadata, user_id)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (id) DO UPDATE SET metadata = $3
            """,
            thread.id, thread.created_at, metadata_json, owner_id
        )

    async def load_thread_items(
        self, thread_id: str, after: str | None, limit: int, order: str, context: dict
    ) -> Page[ThreadItem]:

        cutoff_date = None
        if after:
            row = await Database.fetchrow("SELECT created_at FROM chatkit_items WHERE id = $1", after)
            if row:
                cutoff_date = row["created_at"]

        query = "SELECT * FROM chatkit_items WHERE thread_id = $1"
        params = [thread_id]

        if cutoff_date:
            if order == "desc":
                query += " AND created_at < $2"
            else:
                query += " AND created_at > $2"
            params.append(cutoff_date)

        query += f" ORDER BY created_at {'DESC' if order == 'desc' else 'ASC'}"
        query += f" LIMIT ${len(params) + 1}"
        params.append(limit + 1)

        rows = await Database.fetch(query, *params)

        items = []
        adapter = TypeAdapter(ThreadItem)
        for r in rows:
            item_dict = json.loads(r["content"])
            item_dict["id"] = r["id"]
            item_dict["type"] = r["type"]
            items.append(adapter.validate_python(item_dict))

        has_more = len(items) > limit
        if has_more:
            items = items[:limit]

        return Page(
            data=items,
            has_more=has_more,
            after=items[-1].id if items else None
        )

    async def add_thread_item(self, thread_id: str, item: ThreadItem, context: dict) -> None:
        item_dict = item.model_dump(mode='json')
        item_type = item_dict.pop("type", "message")
        item_dict.pop("id", None)

        await Database.execute(
            """
            INSERT INTO chatkit_items (id, thread_id, type, content, created_at)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (id) DO UPDATE SET content = $4
            """,
            item.id, thread_id, item.type, json.dumps(item_dict), datetime.now(timezone.utc)
        )

    async def save_item(self, thread_id: str, item: ThreadItem, context: dict) -> None:
        await self.add_thread_item(thread_id, item, context)

    async def load_item(self, thread_id: str, item_id: str, context: dict) -> ThreadItem:
        row = await Database.fetchrow(
            "SELECT * FROM chatkit_items WHERE id = $1 AND thread_id = $2", item_id, thread_id
        )
        if not row:
            raise ValueError(f"Item {item_id} not found in thread {thread_id}")

        item_dict = json.loads(row["content"])
        item_dict["id"] = row["id"]
        item_dict["type"] = row["type"]
        return TypeAdapter(ThreadItem).validate_python(item_dict)

    async def delete_thread_item(self, thread_id: str, item_id: str, context: dict) -> None:
        await Database.execute(
            "DELETE FROM chatkit_items WHERE id = $1 AND thread_id = $2", item_id, thread_id
        )

    async def load_threads(self, limit: int, after: str | None, order: str, context: dict) -> Page[ThreadMetadata]:
        user_id = context.get("user_id")

        # Anonymous users get NO history
        if user_id is None:
            return Page(data=[], has_more=False)

        query = "SELECT * FROM chatkit_threads WHERE user_id = $1"
        params = [user_id]

        cutoff_date = None
        if after:
            row = await Database.fetchrow("SELECT created_at FROM chatkit_threads WHERE id = $1", after)
            if row:
                cutoff_date = row["created_at"]

        if cutoff_date:
            if order == "desc":
                query += " AND created_at < $2"
            else:
                query += " AND created_at > $2"
            params.append(cutoff_date)

        query += f" ORDER BY created_at {'DESC' if order == 'desc' else 'ASC'}"
        query += f" LIMIT ${len(params) + 1}"
        params.append(limit + 1)

        rows = await Database.fetch(query, *params)

        threads = []
        for r in rows:
            metadata = json.loads(r["metadata"])
            threads.append(ThreadMetadata(
                id=r["id"],
                created_at=r["created_at"],
                metadata=metadata,
                title=metadata.get("title")
            ))

        has_more = len(threads) > limit
        if has_more:
            threads = threads[:limit]

        return Page(
            data=threads,
            has_more=has_more,
            after=threads[-1].id if threads else None
        )

    async def delete_thread(self, thread_id: str, context: dict) -> None:
        await Database.execute("DELETE FROM chatkit_threads WHERE id = $1", thread_id)

    # Attachments - Memory Stub (extend for S3/blob storage)
    async def save_attachment(self, attachment: Any, context: dict) -> None:
        self._attachments[attachment.id] = attachment

    async def load_attachment(self, attachment_id: str, context: dict) -> Any:
        if attachment_id not in self._attachments:
            raise ValueError(f"Attachment {attachment_id} not found")
        return self._attachments[attachment_id]

    async def delete_attachment(self, attachment_id: str, context: dict) -> None:
        self._attachments.pop(attachment_id, None)

    # Synchronous ID generators (required by Store protocol)
    def generate_thread_id(self, context: dict) -> str:
        import uuid
        return f"thread_{uuid.uuid4().hex[:12]}"

    def generate_item_id(self, item_type: str, thread: ThreadMetadata, context: dict) -> str:
        import uuid
        return f"{item_type}_{uuid.uuid4().hex[:12]}"
```

---

### ChatKit Server & Routes

Create `api/chatkit_routes.py` with the main ChatKit server implementation:

```python
import os
import json
from datetime import datetime, timezone
from typing import Any, AsyncIterator
from fastapi import APIRouter, Request, Depends, BackgroundTasks
from fastapi.responses import StreamingResponse, Response
from agents import Agent, Runner
from agents.extensions.models.litellm_model import LitellmModel

from chatkit.server import ChatKitServer, StreamingResult
from chatkit.types import ThreadMetadata, ThreadItem, UserMessageItem
from chatkit.agents import AgentContext, ThreadItemConverter, stream_agent_response

from services.chatkit_postgres_store import PostgresStore
from services.query.retrieval import QueryService  # Optional: for RAG
from api.deps import get_optional_current_user, UserSession

router = APIRouter()

# Initialize Store (PostgresStore for persistence)
store = PostgresStore()

# Initialize LLM Model (Gemini via LiteLLM)
gemini_model = LitellmModel(
    model="gemini/gemini-2.5-flash",
    api_key=os.getenv("GEMINI_API_KEY"),
)


class RAGChatKitServer(ChatKitServer[dict]):
    """Custom ChatKit server with RAG support and auto-title generation"""

    def __init__(self, data_store: PostgresStore):
        super().__init__(data_store)
        self.converter = ThreadItemConverter()

    async def generate_title(self, thread_id: str, user_message: str):
        """Generates a short 3-5 word title for the chat thread."""
        try:
            print(f"Generating title for thread {thread_id}...")

            prompt = (
                f"Generate a very short, summarized title (maximum 5 words) for a conversation "
                f"that starts with this message:\n\n{user_message}\n\n"
                "Do not use quotes or anything else. Just the title."
            )

            title_agent = Agent(
                name="TitleGenerator",
                instructions="You are a helpful assistant.",
                model=gemini_model
            )

            result = await Runner.run(title_agent, input=prompt)
            new_title = getattr(result, "final_output", str(result)).strip()

            # Update thread metadata
            thread = await self.store.load_thread(thread_id, {})
            thread.metadata["title"] = new_title
            thread.title = new_title
            await self.store.save_thread(thread, {})
            print(f"Set title for thread {thread_id} to: {new_title}")
            return thread

        except Exception as e:
            print(f"Error generating title: {e}")
            return None

    async def respond(self, thread: ThreadMetadata, input: Any, context: dict) -> AsyncIterator:
        """
        Respond to user input with RAG-augmented streaming response.
        """
        # Initialize RAG service (optional)
        query_service = QueryService()

        # 1. Extract user message
        user_message = ""
        if isinstance(input, str):
            user_message = input
        elif isinstance(input, dict) and "text" in input:
            user_message = input["text"]
        elif hasattr(input, "text"):
            user_message = input.text

        if not user_message:
            page = await self.store.load_thread_items(
                thread.id, limit=1, order="desc", after=None, context=context
            )
            if page.data and isinstance(page.data[0], UserMessageItem):
                user_text_parts = [p.text for p in page.data[0].content if hasattr(p, 'text')]
                user_message = " ".join(user_text_parts)

        if not user_message:
            user_message = "Hello"

        # 2. Auto-generate title if needed
        current_title = thread.metadata.get("title")
        title_task = None
        if not current_title or current_title == "New Thread":
            import asyncio
            title_task = asyncio.create_task(self.generate_title(thread.id, user_message))

        # 3. RAG: Retrieve Context (optional - remove if not using RAG)
        from services.query.models import ValidationQuery
        search_response = query_service.search(
            ValidationQuery(query_text=user_message, top_k=5)
        )

        context_text = ""
        for i, res in enumerate(search_response.results):
            context_text += f"SOURCE [{i+1}] ({res.page_title}):\n{res.chunk_text}\n\n"

        # 4. Create Agent with Context
        instructions = f"""
        You are an expert assistant.
        Answer based ONLY on the provided context.

        CONTEXT:
        {context_text}

        If the answer is not in the context, say you don't know.
        """

        agent = Agent(
            name="Assistant",
            instructions=instructions,
            model=gemini_model
        )

        # 5. Stream Response
        agent_context = AgentContext(
            thread=thread,
            store=self.store,
            request_context=context,
        )

        streamed = Runner.run_streamed(agent, input=user_message)

        # ID remapping for unique persistent IDs
        id_map = {}

        async for event in stream_agent_response(agent_context, streamed):
            if hasattr(event, "item") and event.item:
                original_id = event.item.id
                if original_id not in id_map:
                    id_map[original_id] = self.store.generate_item_id("message", thread, context)
                event.item.id = id_map[original_id]

            if hasattr(event, "item_id") and event.item_id in id_map:
                event.item_id = id_map[event.item_id]

            yield event

        # Post-stream: Send title update if generated
        if title_task:
            try:
                updated_thread = await title_task
                if updated_thread:
                    from chatkit.types import ThreadUpdatedEvent, Thread

                    items_page = await self.store.load_thread_items(
                        updated_thread.id, limit=50, order="desc", after=None, context=context
                    )

                    full_thread = Thread(
                        **updated_thread.model_dump(),
                        items=items_page
                    )
                    yield ThreadUpdatedEvent(thread=full_thread)
            except Exception as e:
                print(f"Error awaiting title task: {e}")


# Instantiate Server
server = RAGChatKitServer(store)


@router.post("/chatkit")
async def chatkit_endpoint(
    request: Request,
    background_tasks: BackgroundTasks,
    user: UserSession | None = Depends(get_optional_current_user)
):
    """Main ChatKit endpoint - handles all ChatKit protocol requests"""
    try:
        body = await request.body()

        context = {
            "background_tasks": background_tasks,
            "user_id": user.id if user else None
        }

        result = await server.process(body, context)

        if isinstance(result, StreamingResult):
            return StreamingResponse(result, media_type="text/event-stream")

        return Response(content=result.json, media_type="application/json")
    except Exception as e:
        print(f"Error in chatkit_endpoint: {e}")
        return Response(
            content=json.dumps({"error": str(e)}),
            status_code=500,
            media_type="application/json"
        )
```

---

### Authentication Dependencies

Create `api/deps.py` for extracting user from JWT/session:

```python
import os
from datetime import datetime
from typing import Annotated, Optional
from pydantic import BaseModel
from fastapi import Depends, HTTPException, status, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from services.db import Database


class UserSession(BaseModel):
    """Represents the authenticated user"""
    id: str
    email: str | None = None
    name: str | None = None
    email_verified: bool = False


security = HTTPBearer(auto_error=False)


async def get_current_user_token(
    request: Request,
    token_auth: Annotated[Optional[HTTPAuthorizationCredentials], Depends(security)]
) -> str:
    """Extract session token from Bearer header OR cookie."""
    token = None

    # 1. Try Bearer Token
    if token_auth:
        token = token_auth.credentials

    # 2. Try Cookie
    if not token:
        token = request.cookies.get("better-auth.session_token")

    if not token:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Missing authentication credentials"
        )

    return token


async def get_current_user(
    token: Annotated[str, Depends(get_current_user_token)]
) -> UserSession:
    """Validate session token and return user."""
    session_row = await Database.fetchrow(
        'SELECT "userId", "expiresAt" FROM public."session" WHERE token = $1',
        token
    )

    if not session_row:
        raise HTTPException(status_code=401, detail="Invalid session token")

    expires_at = session_row['expiresAt']
    now = datetime.now(expires_at.tzinfo) if expires_at.tzinfo else datetime.now()

    if now > expires_at:
        raise HTTPException(status_code=401, detail="Session expired")

    user_id = session_row['userId']

    user_row = await Database.fetchrow(
        'SELECT name, email, "emailVerified" FROM public."user" WHERE id = $1',
        user_id
    )

    if not user_row:
        raise HTTPException(status_code=401, detail="User not found")

    return UserSession(
        id=user_id,
        email=user_row['email'],
        name=user_row['name'],
        email_verified=user_row['emailVerified']
    )


async def get_current_user_token_optional(
    request: Request,
    token_auth: Annotated[Optional[HTTPAuthorizationCredentials], Depends(security)]
) -> str | None:
    """Optional token extraction (for anonymous users)."""
    token = None
    if token_auth:
        token = token_auth.credentials
    if not token:
        token = request.cookies.get("better-auth.session_token")
    return token


async def get_optional_current_user(
    token: Annotated[str | None, Depends(get_current_user_token_optional)]
) -> UserSession | None:
    """Get user if authenticated, else None."""
    if not token:
        return None
    try:
        session_row = await Database.fetchrow(
            'SELECT "userId", "expiresAt" FROM public."session" WHERE token = $1', token
        )
        if not session_row:
            return None

        expires_at = session_row['expiresAt']
        now = datetime.now(expires_at.tzinfo) if expires_at.tzinfo else datetime.now()
        if now > expires_at:
            return None

        user_row = await Database.fetchrow(
            'SELECT name, email, "emailVerified" FROM public."user" WHERE id = $1',
            session_row['userId']
        )
        if not user_row:
            return None

        return UserSession(
            id=session_row['userId'],
            email=user_row['email'],
            name=user_row['name'],
            email_verified=user_row['emailVerified']
        )
    except Exception:
        return None
```

---

### FastAPI Main Application

Update `main.py` to register routes:

```python
from dotenv import load_dotenv
from fastapi import FastAPI
from contextlib import asynccontextmanager
from api import chatkit_routes
from services.db import Database

load_dotenv()


@asynccontextmanager
async def lifespan(app: FastAPI):
    await Database.connect()
    yield
    await Database.disconnect()


app = FastAPI(title="ChatKit API", version="0.1.0", lifespan=lifespan)

# CORS Configuration
from fastapi.middleware.cors import CORSMiddleware
import os

default_origins = ["http://localhost:3000", "http://localhost:3001"]
env_origins = os.getenv("ALLOWED_ORIGINS", "").split(",")
allow_origins = default_origins + [o.strip() for o in env_origins if o.strip()]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allow_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Register ChatKit router
app.include_router(chatkit_routes.router, tags=["ChatKit"])


@app.get("/health")
def health_check():
    return {"status": "ok"}
```

---

## Frontend Implementation

### Auth Client Setup

Create `src/lib/auth-client-factory.ts`:

```typescript
import { createAuthClient } from "better-auth/react";

// For Next.js, use env variable. For Docusaurus, use customFields
const BETTER_AUTH_URL = process.env.NEXT_PUBLIC_BETTER_AUTH_URL
  || (typeof window !== 'undefined' ? (window as any).__CONFIG__?.betterAuthUrl : null)
  || "http://localhost:3001";

export const authClient = createAuthClient({
    baseURL: BETTER_AUTH_URL
});

export const { signIn, signUp, useSession, signOut } = authClient;
```

---

### ChatBot Component

Create `src/components/ChatBot.tsx`:

```tsx
import React, { useState, useEffect } from 'react';
import { ChatKit, useChatKit } from '@openai/chatkit-react';
import { useSession } from '../lib/auth-client-factory';
import styles from '../css/chat.module.css';
import { TbMessageChatbot } from 'react-icons/tb';

// Configuration - adjust for your app
const CHATKIT_URL = process.env.NEXT_PUBLIC_CHATKIT_URL || 'http://localhost:8000/chatkit';
const CHATKIT_DOMAIN_KEY = process.env.NEXT_PUBLIC_CHATKIT_DOMAIN_KEY || 'localhost';


// Inner component handling a specific chat session
const ChatKitSession: React.FC<{
    storageKey: string;
    config: { url: string; domainKey: string };
    token?: string;
}> = ({ storageKey, config, token }) => {

    // Initialize with thread ID from localStorage if available
    const [initialThread] = useState<string | null>(() => {
        if (typeof window === 'undefined') return null;
        return localStorage.getItem(storageKey) || null;
    });

    const [isChatOpen, setIsChatOpen] = useState(false);

    // Custom fetch to inject Authorization header
    const customFetch: typeof fetch = React.useCallback(async (input, init) => {
        const headers = new Headers(init?.headers);
        if (token) {
            headers.set("Authorization", `Bearer ${token}`);
        }
        return fetch(input, { ...init, headers });
    }, [token]);

    const { control, setThreadId } = useChatKit({
        api: {
            ...config,
            fetch: customFetch
        },
        initialThread,
        theme: {
            colorScheme: 'light',
            radius: 'pill',
            density: 'spacious',
            color: {
                grayscale: { hue: 317, tint: 9, shade: 3 },
                accent: { primary: '#FCA5A5', level: 2 }
            },
            typography: {
                baseSize: 17,
                fontFamily: 'Inter, sans-serif',
            }
        },
        startScreen: {
            greeting: 'Hello! How can I help you today?',
            prompts: [
                { icon: 'book-open', label: 'Learn', prompt: 'What can you teach me?' },
                { icon: 'write', label: 'Help', prompt: 'I need help with...' },
                { icon: 'desktop', label: 'Features', prompt: 'What features do you have?' }
            ],
        },
        composer: {
            placeholder: 'Type your message...',
            attachments: { enabled: false },
            tools: [],
        },
        onThreadChange: ({ threadId }) => {
            if (typeof window !== 'undefined') {
                localStorage.setItem(storageKey, threadId);
            }
        },
    });

    // Restore latest thread for authenticated users
    useEffect(() => {
        const restoreLatestThread = async () => {
            if (!token || initialThread) return;

            try {
                const response = await customFetch(config.url, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        type: 'threads.list',
                        params: { limit: 1, order: 'desc' }
                    })
                });

                if (!response.ok) return;

                const data = await response.json();
                if (data.data && data.data.length > 0) {
                    const latestThreadId = data.data[0].id;
                    setThreadId(latestThreadId);
                    localStorage.setItem(storageKey, latestThreadId);
                }
            } catch (e) {
                console.error('[ChatKitSession] Failed to restore thread:', e);
            }
        };

        restoreLatestThread();
    }, [token, initialThread, config.url, customFetch, storageKey, setThreadId]);


    return (
        <>
            {/* Floating Launcher Button */}
            {!isChatOpen && (
                <button
                    onClick={() => setIsChatOpen(true)}
                    className={styles.launcherBtn}
                    aria-label="Open Chat"
                >
                    <TbMessageChatbot className={styles.chatIcon} />
                </button>
            )}

            {/* Chat Widget */}
            {isChatOpen && (
                <>
                    <div
                        onClick={() => setIsChatOpen(false)}
                        className={styles.backdrop}
                    />
                    <div className={styles.widgetContainer}>
                        <div className={styles.widgetContent}>
                            <ChatKit control={control} className="chatkit-full" />
                        </div>
                    </div>
                </>
            )}
        </>
    );
};


const ChatBot: React.FC = () => {
    // Use session hook - don't block render
    const { data: session, isPending } = useSession();

    // Don't render until we know user state
    if (isPending) return null;

    // Distinct storage key per user
    const storageKey = session?.user?.id
        ? `chatkit_thread_${session.user.id}`
        : 'chatkit_thread_anonymous';

    // Extract token from session
    const token = session?.session?.token;

    return (
        <ChatKitSession
            key={storageKey}  // Forces remount when user changes
            storageKey={storageKey}
            config={{ url: CHATKIT_URL, domainKey: CHATKIT_DOMAIN_KEY }}
            token={token}
        />
    );
};

export default ChatBot;
```

---

### Styling

Create `src/css/chat.module.css`:

```css
/* Chat Widget Container */
.widgetContainer {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  width: 420px;
  height: 600px;
  max-width: calc(100vw - 4rem);
  max-height: calc(100vh - 4rem);
  border-radius: 24px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  z-index: 1000;
  animation: popupIn 0.3s cubic-bezier(0.16, 1, 0.3, 1);
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.1);
}

.widgetContent {
  flex: 1;
  overflow: hidden;
  position: relative;
}

/* Backdrop */
.backdrop {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.1);
  backdrop-filter: blur(4px);
  -webkit-backdrop-filter: blur(4px);
  z-index: 999;
}

/* Launcher Button */
.launcherBtn {
  position: fixed;
  bottom: 2rem;
  right: 2rem;
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: #ffffff;
  border: 2px solid rgba(100, 100, 240, 0.3);
  cursor: pointer;
  box-shadow: 0 8px 30px rgba(100, 100, 240, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  z-index: 100;
  animation: pulseGlow 3s infinite;
}

.launcherBtn:hover {
  transform: translateY(-2px) scale(1.1);
  box-shadow: 0 12px 40px rgba(100, 100, 240, 0.5);
  border-color: #6464f0;
}

.chatIcon {
  color: #6464f0;
  font-size: 34px;
  filter: drop-shadow(0 0 2px rgba(100, 100, 240, 0.4));
  transition: transform 0.5s ease;
}

.launcherBtn:hover .chatIcon {
  transform: rotate(15deg) scale(1.1);
}

/* Animations */
@keyframes pulseGlow {
  0% { box-shadow: 0 0 0 0 rgba(100, 100, 240, 0.4); }
  70% { box-shadow: 0 0 0 15px rgba(100, 100, 240, 0); }
  100% { box-shadow: 0 0 0 0 rgba(100, 100, 240, 0); }
}

@keyframes popupIn {
  from { opacity: 0; transform: scale(0.95) translateY(20px); }
  to { opacity: 1; transform: scale(1) translateY(0); }
}

/* Responsive */
@media (max-width: 480px) {
  .widgetContainer {
    bottom: 0;
    right: 0;
    width: 100vw;
    height: 100vh;
    max-width: 100vw;
    max-height: 100vh;
    border-radius: 0;
  }
}
```

---

### Configuration

**For Next.js apps**, add to `.env.local`:
```env
NEXT_PUBLIC_CHATKIT_URL=http://localhost:8000/chatkit
NEXT_PUBLIC_CHATKIT_DOMAIN_KEY=localhost
NEXT_PUBLIC_BETTER_AUTH_URL=http://localhost:3001
```

**For Docusaurus**, update `docusaurus.config.ts`:
```typescript
const config: Config = {
  // ...
  customFields: {
    chatKitUrl: process.env.CHATKIT_URL || 'http://localhost:8000/chatkit',
    chatKitDomainKey: process.env.CHATKIT_DOMAIN_KEY || 'localhost',
    betterAuthUrl: process.env.BETTER_AUTH_URL || 'http://localhost:3001',
  },
  // Optional: Load ChatKit CDN script
  scripts: [
    {
      src: 'https://cdn.platform.openai.com/deployments/chatkit/chatkit.js',
      async: true,
    },
  ],
  // ...
};
```

---

## Environment Variables

### Backend `.env`

```env
# Database
DATABASE_URL=postgresql://user:password@host.neon.tech/database?sslmode=require

# LLM Provider (Gemini via LiteLLM)
GEMINI_API_KEY=your_gemini_api_key

# Optional: RAG Services
QDRANT_URL=https://your-qdrant-instance.qdrant.io
QDRANT_API_KEY=your_qdrant_api_key
COHERE_API_KEY=your_cohere_api_key

# CORS
ALLOWED_ORIGINS=http://localhost:3000,https://your-production-domain.com
```

### Frontend `.env`

```env
# ChatKit API
CHATKIT_URL=http://localhost:8000/chatkit
CHATKIT_DOMAIN_KEY=localhost

# Auth Service
BETTER_AUTH_URL=http://localhost:3001

# Production
SITE_URL=https://your-production-domain.com
```

---

## Features & Capabilities

| Feature | Implementation | Status |
|---------|----------------|--------|
| **Streaming Responses** | SSE via `StreamingResponse` | ✅ |
| **Thread Persistence** | PostgreSQL or In-Memory Store | ✅ |
| **User Isolation** | `user_id` in thread metadata | ✅ |
| **Anonymous Users** | Separate localStorage key | ✅ |
| **Auth Integration** | Bearer token + Cookie support | ✅ |
| **Auto Title Generation** | Async LLM-based generation | ✅ |
| **RAG Context** | Qdrant + Cohere embeddings | ✅ |
| **Thread History** | `threads.list` API | ✅ |
| **Thread Restoration** | On login, restore latest | ✅ |
| **Custom Theming** | ChatKit theme config | ✅ |
| **Responsive Design** | Mobile fullscreen mode | ✅ |

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/chatkit` | Main ChatKit protocol endpoint |
| `GET` | `/health` | Health check |

### ChatKit Protocol Requests

The `/chatkit` endpoint handles all ChatKit protocol messages:

```typescript
// Create/continue thread
{ type: "thread.respond", params: { threadId: "...", input: "Hello" } }

// List threads
{ type: "threads.list", params: { limit: 10, order: "desc" } }

// Load thread
{ type: "thread.load", params: { threadId: "..." } }

// Delete thread
{ type: "thread.delete", params: { threadId: "..." } }
```

---

## Complete Flow Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant FE as Frontend (ChatBot.tsx)
    participant LS as localStorage
    participant BE as Backend (/chatkit)
    participant DB as PostgreSQL
    participant LLM as Gemini (LiteLLM)
    participant RAG as QueryService

    U->>FE: Opens chat widget
    FE->>LS: Check for existing thread_id

    alt Has thread_id
        FE->>BE: POST /chatkit {type: "thread.load", threadId}
        BE->>DB: Load thread + items
        DB-->>BE: Thread data
        BE-->>FE: Thread with history
    else No thread_id
        Note over FE: New thread will be created on first message
    end

    U->>FE: Types message and sends
    FE->>BE: POST /chatkit {type: "thread.respond", input: "..."}

    Note over BE: Extract user_id from JWT/Cookie
    BE->>DB: Load/Create thread
    BE->>RAG: Search for context (top_k=5)
    RAG-->>BE: Relevant chunks

    BE->>LLM: Stream response with context

    loop Streaming
        LLM-->>BE: Token chunks
        BE-->>FE: SSE events (ThreadItemAdded, Updated, Done)
        FE->>U: Renders tokens as they arrive
    end

    Note over BE: Background: Generate title
    BE->>LLM: Generate title prompt
    LLM-->>BE: "Your Title Here"
    BE->>DB: Update thread metadata
    BE-->>FE: ThreadUpdatedEvent

    FE->>LS: Save thread_id
    FE->>U: Display complete response
```

---

## Quick Start Checklist

1. **Backend Setup**
   - [ ] Install dependencies: `uv add "openai-chatkit>=1.4.0" "openai-agents[litellm]>=0.6.3"`
   - [ ] Create `services/db.py`
   - [ ] Create `services/chatkit_postgres_store.py`
   - [ ] Create `api/chatkit_routes.py`
   - [ ] Create `api/deps.py`
   - [ ] Update `main.py` with lifespan and router
   - [ ] Run `python scripts/init_chat_db.py`
   - [ ] Set environment variables

2. **Frontend Setup**
   - [ ] Install dependencies: `npm install @openai/chatkit-react better-auth react-icons`
   - [ ] Create `src/lib/auth-client-factory.ts`
   - [ ] Create `src/components/ChatBot.tsx`
   - [ ] Create `src/css/chat.module.css`
   - [ ] Add ChatBot component to your layout
   - [ ] Set environment variables

3. **Testing**
   - [ ] Start backend: `uvicorn main:app --reload`
   - [ ] Start frontend: `npm run dev`
   - [ ] Open browser and test chat widget
   - [ ] Verify streaming responses
   - [ ] Test anonymous → authenticated flow

---

> **Note**: This guide is based on the implementation in the Physical AI & Humanoid Robotics Textbook project. Adapt the RAG integration, theming, and auth configuration to match your specific application requirements.
