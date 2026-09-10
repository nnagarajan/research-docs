# Deep Agents + assistant-ui: Self-Hosted Build Guide

A complete reference for building a chat UI over a LangChain **deep agent** backend, without LangSmith or LangGraph Platform. Covers two paths (managed and manual), then goes deep on the manual one: your own FastAPI endpoints, session auth, multi-user scoping, structured output, and the full frontend.

---

## Table of contents

1. [Architecture overview](#1-architecture-overview)
2. [Path A — managed LangGraph server (the short version)](#2-path-a--managed-langgraph-server)
3. [Path B — manual endpoints (recommended here)](#3-path-b--manual-endpoints)
4. [Backend: the agent](#4-backend-the-agent)
5. [Backend: serialization](#5-backend-serialization)
6. [Backend: FastAPI server](#6-backend-fastapi-server)
7. [Auth: sessions vs JWT](#7-auth-sessions-vs-jwt)
8. [Multi-user scoping](#8-multi-user-scoping)
9. [Structured output](#9-structured-output)
10. [Frontend: full code](#10-frontend-full-code)
11. [assistant-ui vs building your own](#11-assistant-ui-vs-building-your-own)
12. [Gotchas](#12-gotchas)

---

## 1. Architecture overview

A deep agent is just a compiled LangGraph graph. That means you have two ways to serve it:

| | Path A: LangGraph Agent Server | Path B: manual FastAPI |
|---|---|---|
| Threads / checkpoints | provisioned for you | you build them |
| Streaming protocol | LangGraph SDK events | your own SSE format |
| Frontend runtime | `useLangGraphRuntime` | `useExternalStoreRuntime` |
| Free features | branching, thread list, generative UI | none |
| Control | opinionated | total |

Both terminate in **assistant-ui** on the frontend, which is a React chat-UI library with pluggable runtimes.

The layer diagram either way:

```
React + assistant-ui
        │  (runtime adapter — the seam)
   your state hook
        │  (HTTP + SSE)
   FastAPI / LangGraph server
        │
   deep agent (LangGraph graph)
        │
   checkpointer (SQLite/Postgres) + store
```

---

## 2. Path A — managed LangGraph server

Included for completeness; skip to Path B if self-hosting.

**`agent.py`**

```python
from deepagents import create_deep_agent
agent = create_deep_agent(tools=[...], system_prompt="...")
```

**`langgraph.json`**

```json
{
  "dependencies": ["."],
  "graphs": { "agent": "./agent.py:agent" },
  "env": ".env"
}
```

Run `langgraph dev` → `http://localhost:2024`. Threads, runs, store, and checkpointer are provisioned automatically. Do **not** pass a checkpointer in code on this path — the server supplies one.

**Frontend**

```bash
npx create-assistant-ui@latest -t langchain my-app
```

```
NEXT_PUBLIC_LANGGRAPH_API_URL=http://localhost:2024
NEXT_PUBLIC_LANGGRAPH_ASSISTANT_ID=agent
```

The generated component wires `useLangGraphRuntime` with `unstable_createLangGraphStream({ client, assistantId })`, plus `create` (calls `client.threads.create()`) and `load` (reads `state.values.messages` and `state.tasks[0]?.interrupts`). The runtime requires the graph state to include a `messages` key with LangChain-alike messages — deep agents satisfy this.

Extras this path gives you for free: interrupt handling, subgraph streaming for sub-agent activity, message editing/regeneration via `getCheckpointId`, and agent-state reads for todos and the virtual filesystem.

For production, proxy through your own backend (a Next.js catch-all route) so the API key never reaches the client.

---

## 3. Path B — manual endpoints

### Protocol

```
POST   /threads                  -> { thread_id }
GET    /threads                  -> { threads: [...] }
GET    /threads/{id}             -> { messages, todos, structured, interrupt }
POST   /threads/{id}/messages    -> SSE stream
POST   /threads/{id}/resume      -> SSE stream
DELETE /threads/{id}
```

SSE frames are single-line JSON with a `type` field:

`message_start` · `text_delta` · `tool_call` · `tool_result` · `todos` · `structured` · `interrupt` · `error` · `done`

Cancellation is client-driven: abort the `fetch`, the server sees the disconnect, the checkpointer keeps whatever completed, and the client re-syncs with a `GET`.

---

## 4. Backend: the agent

**`agent.py`**

```python
import os
from deepagents import create_deep_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel, Field
from tavily import TavilyClient

_tavily = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])


def search_web(query: str, max_results: int = 5) -> list[dict]:
    """Search the web."""
    res = _tavily.search(query, max_results=max_results)
    return [
        {"title": r["title"], "url": r["url"], "content": r["content"]}
        for r in res["results"]
    ]


class Source(BaseModel):
    title: str
    url: str


class ResearchBrief(BaseModel):
    """Final structured answer."""
    summary: str = Field(description="Two-sentence answer.")
    confidence: float = Field(ge=0, le=1)
    sources: list[Source]
    open_questions: list[str] = []


def build_agent(checkpointer):
    return create_deep_agent(
        model=ChatAnthropic(model="claude-sonnet-4-5-20250929", max_tokens=8000),
        tools=[search_web],
        subagents=[{
            "name": "researcher",
            "description": "Deep research on one narrow question.",
            "system_prompt": "You are a thorough researcher. Cite sources.",
            "tools": [search_web],
        }],
        system_prompt="You are a research assistant. Plan with write_todos first.",
        response_format=ResearchBrief,
        middleware=[HumanInTheLoopMiddleware(interrupt_on={"search_web": True})],
        checkpointer=checkpointer,
    )
```

Built-in deep agent tools you may want to render specially: `write_todos`, `task` (sub-agent dispatch), `ls` / `read_file` / `write_file` / `edit_file` / `glob` / `grep`, and `execute`.

**Backends** control where the agent's virtual filesystem lives:

- `StateBackend` — ephemeral, in graph state, per-thread (default)
- `StoreBackend` — persistent across threads, requires a `store`
- `FilesystemBackend` — real disk
- `CompositeBackend` — route paths to different backends

---

## 5. Backend: serialization

**`serialize.py`**

```python
from langchain_core.messages import AIMessage, HumanMessage, ToolMessage


def text_of(content) -> str:
    if isinstance(content, str):
        return content
    return "".join(
        b.get("text", "") for b in content
        if isinstance(b, dict) and b.get("type") == "text"
    )


def serialize_messages(messages) -> list[dict]:
    """LangChain messages -> [{id, role, text, tool_calls:[...]}]"""
    out: list[dict] = []
    results: dict[str, str] = {}

    for m in messages:
        if isinstance(m, ToolMessage):
            results[m.tool_call_id] = text_of(m.content)

    for m in messages:
        if isinstance(m, HumanMessage):
            out.append({"id": m.id, "role": "user",
                        "text": text_of(m.content), "tool_calls": []})
        elif isinstance(m, AIMessage):
            calls = [{
                "id": tc["id"],
                "name": tc["name"],
                "args": tc["args"],
                "result": results.get(tc["id"]),
            } for tc in (m.tool_calls or [])]
            body = text_of(m.content)
            if body or calls:
                out.append({"id": m.id, "role": "assistant",
                            "text": body, "tool_calls": calls})
    return out


def serialize_structured(value) -> dict | None:
    if value is None:
        return None
    if hasattr(value, "model_dump"):      # pydantic
        return value.model_dump(mode="json")
    return value                           # already a dict
```

---

## 6. Backend: FastAPI server

**`server.py`**

```python
import json
import uuid
from contextlib import asynccontextmanager
from typing import AsyncIterator

from fastapi import Depends, FastAPI, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import StreamingResponse
from langchain_core.messages import AIMessageChunk, HumanMessage
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
from langgraph.types import Command
from pydantic import BaseModel

import threads
from agent import build_agent
from auth import current_user
from serialize import serialize_messages, serialize_structured, text_of

STATE: dict = {}
ALLOWED_ORIGINS = ["http://localhost:3000"]


@asynccontextmanager
async def lifespan(app: FastAPI):
    await threads.init()
    async with AsyncSqliteSaver.from_conn_string("checkpoints.sqlite") as saver:
        STATE["agent"] = build_agent(saver)
        yield


app = FastAPI(lifespan=lifespan)
app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,          # required for cookie auth
    allow_methods=["*"],
    allow_headers=["*"],
)


def config_for(user_id: str, thread_id: str) -> dict:
    return {
        "configurable": {"thread_id": thread_id, "user_id": user_id},
        "recursion_limit": 100,
    }


async def owned_thread(thread_id: str,
                       user_id: str = Depends(current_user)) -> tuple[str, str]:
    if not await threads.owns(user_id, thread_id):
        raise HTTPException(404, "thread not found")   # 404, not 403
    return thread_id, user_id


def sse(payload: dict) -> str:
    return f"data: {json.dumps(payload)}\n\n"


async def run(agent, inp, config, request: Request) -> AsyncIterator[str]:
    """Shared streaming loop for /messages and /resume."""
    open_msg_id: str | None = None
    try:
        async for mode, chunk in agent.astream(
            inp, config, stream_mode=["messages", "updates"], subgraphs=True
        ):
            if await request.is_disconnected():
                break

            if mode == "messages":
                msg, meta = chunk
                if not isinstance(msg, AIMessageChunk):
                    continue
                delta = text_of(msg.content)
                if not delta:
                    continue
                if open_msg_id != msg.id:
                    open_msg_id = msg.id
                    yield sse({"type": "message_start", "id": msg.id,
                               "node": meta.get("langgraph_node"),
                               "subagent": meta.get("subgraph_name")})
                yield sse({"type": "text_delta", "id": msg.id, "delta": delta})

            elif mode == "updates":
                for node, update in (chunk or {}).items():
                    if node == "__interrupt__":
                        yield sse({"type": "interrupt", "value": update[0].value})
                        continue
                    if not isinstance(update, dict):
                        continue
                    if "todos" in update:
                        yield sse({"type": "todos", "todos": update["todos"]})
                    if "structured_response" in update:
                        yield sse({"type": "structured", "message_id": open_msg_id,
                                   "data": serialize_structured(
                                       update["structured_response"])})
                    for m in update.get("messages", []) or []:
                        for tc in getattr(m, "tool_calls", None) or []:
                            yield sse({"type": "tool_call", "message_id": m.id,
                                       "id": tc["id"], "name": tc["name"],
                                       "args": tc["args"]})
                        if getattr(m, "tool_call_id", None):
                            yield sse({"type": "tool_result", "id": m.tool_call_id,
                                       "result": text_of(m.content)})
    except Exception as exc:
        yield sse({"type": "error", "message": str(exc)})
    finally:
        yield sse({"type": "done"})


class NewMessage(BaseModel):
    content: str


class Resume(BaseModel):
    decision: dict     # {"type": "accept"} / {"type": "reject"} / {"type": "edit", "args": {...}}


@app.post("/threads")
async def create_thread(user_id: str = Depends(current_user)):
    return {"thread_id": await threads.create(user_id)}


@app.get("/threads")
async def list_threads(user_id: str = Depends(current_user)):
    return {"threads": await threads.list_for(user_id)}


@app.get("/threads/{thread_id}")
async def get_thread(ctx: tuple = Depends(owned_thread)):
    thread_id, user_id = ctx
    snap = await STATE["agent"].aget_state(config_for(user_id, thread_id))
    interrupt = None
    if snap.tasks and snap.tasks[0].interrupts:
        interrupt = snap.tasks[0].interrupts[0].value
    return {
        "messages": serialize_messages(snap.values.get("messages", [])),
        "todos": snap.values.get("todos", []),
        "structured": serialize_structured(snap.values.get("structured_response")),
        "interrupt": interrupt,
    }


@app.post("/threads/{thread_id}/messages")
async def post_message(body: NewMessage, request: Request,
                       ctx: tuple = Depends(owned_thread)):
    thread_id, user_id = ctx
    return StreamingResponse(
        run(STATE["agent"], {"messages": [HumanMessage(content=body.content)]},
            config_for(user_id, thread_id), request),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )


@app.post("/threads/{thread_id}/resume")
async def resume(body: Resume, request: Request,
                 ctx: tuple = Depends(owned_thread)):
    thread_id, user_id = ctx
    cfg = config_for(user_id, thread_id)
    snap = await STATE["agent"].aget_state(cfg)
    if not (snap.tasks and snap.tasks[0].interrupts):
        raise HTTPException(409, "thread is not interrupted")
    return StreamingResponse(
        run(STATE["agent"], Command(resume=[body.decision]), cfg, request),
        media_type="text/event-stream",
    )
```

Run: `uvicorn server:app --reload --port 8000`

`X-Accel-Buffering: no` matters if nginx sits in front — otherwise it buffers the SSE stream and nothing appears until the run ends.

---

## 7. Auth: sessions vs JWT

**Sessions are the better fit here.** Reasons:

- **Revocation** — delete the row, user is out. A JWT stays valid until expiry; a denylist is just a session table with extra steps.
- **Long runs** — a deep agent run can exceed 5 minutes. A short-lived JWT expiring mid-stream forces a refresh dance. Sessions don't have this problem.
- **You already have a DB** — the thread-ownership table. A `sessions` table costs nothing extra.
- **Nothing sensitive in the browser** — `HttpOnly; Secure; SameSite` means XSS can't read the credential.

**Costs:**

- **CSRF becomes real** — cookies auto-attach. Need `SameSite=Lax` plus an origin check on mutating routes.
- **Cross-origin friction** — `localhost:3000` → `localhost:8000` is cross-origin, so you need `credentials: "include"`, `allow_credentials=True`, and an explicit origin list (wildcard `*` is rejected with credentials). Easiest fix: proxy the API under the Next.js origin.
- **One DB round-trip per request** — irrelevant next to a multi-second LLM call.

Cookies also let you use native `EventSource` (which can't set headers but does send cookies). Not needed here, since POST-based `fetch` streaming is more flexible.

**`auth.py`**

```python
import secrets
import time

import aiosqlite
from fastapi import HTTPException, Request

DB = "threads.sqlite"
ALLOWED_ORIGINS = {"http://localhost:3000"}


async def current_user(request: Request) -> str:
    sid = request.cookies.get("sid")
    if not sid:
        raise HTTPException(401, "no session")
    async with aiosqlite.connect(DB) as db:
        cur = await db.execute(
            "SELECT user_id FROM sessions WHERE sid=? AND expires_at > ?",
            (sid, time.time()),
        )
        row = await cur.fetchone()
    if not row:
        raise HTTPException(401, "expired session")
    return row[0]


def check_origin(request: Request):
    if request.headers.get("origin") not in ALLOWED_ORIGINS:
        raise HTTPException(403, "bad origin")
```

On login:

```python
resp.set_cookie("sid", secrets.token_urlsafe(32), httponly=True,
                secure=True, samesite="lax", max_age=60 * 60 * 24 * 14)
```

Store a hash of the sid rather than the raw value for DB-leak resistance.

Everything downstream is auth-agnostic: `current_user` returns a user id, and `owns()`, `config_for()`, and store namespacing don't care where it came from. Swapping to JWT later is a one-function change.

---

## 8. Multi-user scoping

Three separate concerns: **authenticating** the request, **owning** the thread, **scoping** the memory.

The checkpointer has no notion of users — `thread_id` is a flat global namespace. Anyone who guesses or steals a UUID reads that conversation. Ownership is on you.

**`threads.py`**

```python
import time
import uuid

import aiosqlite

DB = "threads.sqlite"


async def init():
    async with aiosqlite.connect(DB) as db:
        await db.execute("""
          CREATE TABLE IF NOT EXISTS threads (
            thread_id  TEXT PRIMARY KEY,
            user_id    TEXT NOT NULL,
            title      TEXT,
            created_at REAL NOT NULL
          )""")
        await db.execute("""
          CREATE TABLE IF NOT EXISTS sessions (
            sid        TEXT PRIMARY KEY,
            user_id    TEXT NOT NULL,
            expires_at REAL NOT NULL
          )""")
        await db.execute(
            "CREATE INDEX IF NOT EXISTS idx_user ON threads(user_id, created_at DESC)")
        await db.commit()


async def create(user_id: str) -> str:
    tid = str(uuid.uuid4())
    async with aiosqlite.connect(DB) as db:
        await db.execute("INSERT INTO threads VALUES (?,?,?,?)",
                         (tid, user_id, None, time.time()))
        await db.commit()
    return tid


async def owns(user_id: str, thread_id: str) -> bool:
    async with aiosqlite.connect(DB) as db:
        cur = await db.execute(
            "SELECT 1 FROM threads WHERE thread_id=? AND user_id=?", (thread_id, user_id))
        return await cur.fetchone() is not None


async def list_for(user_id: str) -> list[dict]:
    async with aiosqlite.connect(DB) as db:
        db.row_factory = aiosqlite.Row
        cur = await db.execute(
            "SELECT thread_id, title, created_at FROM threads "
            "WHERE user_id=? ORDER BY created_at DESC", (user_id,))
        return [dict(r) for r in await cur.fetchall()]
```

**Rules:**

- `thread_id` comes from the path, but is only usable after `owns()` passes.
- Never let the client hand you a `user_id`.
- Return **404, not 403**, on a foreign thread — don't confirm existence.
- The thread list comes from `list_for(user_id)` only. Never enumerate the checkpointer.

**Tools read the user from the runtime**, not a global:

```python
from langgraph.runtime import get_runtime


def get_my_calendar(day: str) -> list[dict]:
    """Fetch the current user's calendar."""
    user_id = get_runtime().config["configurable"]["user_id"]
    return calendar_api.fetch(user_id, day)
```

**Long-term memory must be namespaced** or everyone shares a filesystem:

```python
from deepagents import CompositeBackend, StateBackend, StoreBackend


def backend(rt):
    user_id = rt.config["configurable"]["user_id"]
    return CompositeBackend(
        StateBackend(rt),                                   # per-thread scratch
        {"/memories/": StoreBackend(rt, namespace=("memories", user_id))},
    )


agent = create_deep_agent(..., backend=backend, store=store)
```

Same rule for direct `store.aput` / `astore.asearch` calls: first namespace element is the user id.

**Frontend:** reset all state when the user changes. Simplest is `<RuntimeProvider key={userId}>`, which remounts everything and prevents the previous user's messages leaking into the new session.

The `agent` object itself is shared across users — that's fine, it's stateless. Anything you cache in module scope keyed only by `thread_id` is **not**.

---

## 9. Structured output

`response_format` puts the parsed object in the `structured_response` state key.

**Strategies:**

- bare type (`response_format=ResearchBrief`) — auto-selects
- `ProviderStrategy(ResearchBrief)` — provider-native, most reliable when available
- `ToolStrategy(ResearchBrief)` — tool-calling fallback

Sub-agents accept their own `response_format`; the result is JSON-serialized into the `ToolMessage` the parent sees, replacing the usual last-message extraction. Good for making a `researcher` hand back typed findings instead of prose.

**Three gotchas:**

1. **`structured_response` is a single state slot, not per-message.** Turn 3 overwrites turn 2. A `GET` after several turns returns only the latest. To attach a brief to each assistant message, capture it from the stream and key it by message id client-side — don't reconstruct history from state alone. Your post-stream reconcile must *merge*, not overwrite, the brief map.

2. **It is not guaranteed to appear.** There's an open report of `structured_response` intermittently missing when tools are in play and prompts get long — the result comes back with only `messages`. Treat `structured: null` as a normal branch, not an error. Fall back to the text response.

3. **Structured output and streaming interact badly.** With `ToolStrategy` the object arrives as a tool call, so the user sees no prose streaming at the end of the turn — just a pause, then the card. Either emit a placeholder when the structured tool call begins, or keep the agent's prose answer and treat the structured object as metadata.

---

## 10. Frontend: full code

```bash
npx assistant-ui init      # copies thread.tsx primitives into your repo
```

```
NEXT_PUBLIC_AGENT_API=http://localhost:8000
```

### `lib/api.ts`

```ts
const BASE = process.env.NEXT_PUBLIC_AGENT_API ?? "http://localhost:8000";

export type Todo = { content: string; status: "pending" | "in_progress" | "completed" };
export type Brief = {
  summary: string;
  confidence: number;
  sources: { title: string; url: string }[];
  open_questions: string[];
};

export type ServerEvent =
  | { type: "message_start"; id: string; node?: string; subagent?: string }
  | { type: "text_delta"; id: string; delta: string }
  | { type: "tool_call"; message_id: string; id: string; name: string; args: any }
  | { type: "tool_result"; id: string; result: string }
  | { type: "todos"; todos: Todo[] }
  | { type: "structured"; message_id: string | null; data: Brief }
  | { type: "interrupt"; value: any }
  | { type: "error"; message: string }
  | { type: "done" };

const json = { "Content-Type": "application/json" };

async function req(path: string, init: RequestInit = {}) {
  const res = await fetch(`${BASE}${path}`, { ...init, credentials: "include" });
  if (res.status === 401) { window.location.href = "/login"; throw new Error("unauthorized"); }
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res;
}

export const createThread = () => req("/threads", { method: "POST" }).then((r) => r.json());
export const listThreads  = () => req("/threads").then((r) => r.json());
export const getThread    = (id: string) => req(`/threads/${id}`).then((r) => r.json());

export async function* streamPost(
  path: string,
  body: unknown,
  signal: AbortSignal,
): AsyncGenerator<ServerEvent> {
  const res = await req(path, {
    method: "POST", headers: json, body: JSON.stringify(body), signal,
  });
  const reader = res.body!.pipeThrough(new TextDecoderStream()).getReader();
  let buf = "";
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buf += value;
    const frames = buf.split("\n\n");
    buf = frames.pop() ?? "";
    for (const frame of frames) {
      const line = frame.split("\n").find((l) => l.startsWith("data: "));
      if (line) yield JSON.parse(line.slice(6)) as ServerEvent;
    }
  }
}
```

### `lib/useDeepAgent.ts`

```ts
"use client";
import { useCallback, useEffect, useRef, useState } from "react";
import {
  createThread, getThread, listThreads, streamPost,
  type Brief, type ServerEvent, type Todo,
} from "./api";

export type ToolCall = { id: string; name: string; args: any; result?: string };
export type Msg = { id: string; role: "user" | "assistant"; text: string; toolCalls: ToolCall[] };
export type ThreadMeta = { thread_id: string; title: string | null; created_at: number };

export function useDeepAgent() {
  const [messages, setMessages] = useState<Msg[]>([]);
  const [todos, setTodos] = useState<Todo[]>([]);
  const [briefs, setBriefs] = useState<Record<string, Brief>>({});
  const [interrupt, setInterrupt] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);
  const [isRunning, setIsRunning] = useState(false);
  const [threadId, setThreadId] = useState<string | null>(null);
  const [threadList, setThreadList] = useState<ThreadMeta[]>([]);

  const abort = useRef<AbortController | null>(null);
  const currentMsg = useRef<string | null>(null);

  useEffect(() => {
    listThreads().then((r) => setThreadList(r.threads)).catch(() => {});
  }, []);

  const apply = (e: ServerEvent) => {
    switch (e.type) {
      case "message_start":
        currentMsg.current = e.id;
        setMessages((p) => p.some((m) => m.id === e.id) ? p
          : [...p, { id: e.id, role: "assistant", text: "", toolCalls: [] }]);
        break;
      case "text_delta":
        setMessages((p) => p.map((m) => m.id === e.id ? { ...m, text: m.text + e.delta } : m));
        break;
      case "tool_call":
        setMessages((p) => {
          const base = p.some((m) => m.id === e.message_id) ? p
            : [...p, { id: e.message_id, role: "assistant" as const, text: "", toolCalls: [] }];
          return base.map((m) =>
            m.id === e.message_id && !m.toolCalls.some((t) => t.id === e.id)
              ? { ...m, toolCalls: [...m.toolCalls, { id: e.id, name: e.name, args: e.args }] }
              : m);
        });
        break;
      case "tool_result":
        setMessages((p) => p.map((m) => ({
          ...m,
          toolCalls: m.toolCalls.map((t) => t.id === e.id ? { ...t, result: e.result } : t),
        })));
        break;
      case "todos": setTodos(e.todos); break;
      case "structured":
        setBriefs((p) => ({ ...p, [e.message_id ?? currentMsg.current ?? "latest"]: e.data }));
        break;
      case "interrupt": setInterrupt(e.value); break;
      case "error": setError(e.message); break;
    }
  };

  const hydrate = useCallback(async (id: string) => {
    const s = await getThread(id);
    setMessages(s.messages.map((m: any) => ({
      id: m.id, role: m.role, text: m.text, toolCalls: m.tool_calls,
    })));
    setTodos(s.todos ?? []);
    setInterrupt(s.interrupt ?? null);
    // structured_response is a single slot server-side — merge, never replace
    if (s.structured) setBriefs((p) => ({ ...p, latest: s.structured }));
  }, []);

  const consume = useCallback(async (id: string, path: string, body: unknown) => {
    abort.current = new AbortController();
    setIsRunning(true); setInterrupt(null); setError(null);
    try {
      for await (const e of streamPost(path, body, abort.current.signal)) apply(e);
    } catch (err) {
      if ((err as Error).name !== "AbortError") setError((err as Error).message);
    } finally {
      setIsRunning(false);
      currentMsg.current = null;
      await hydrate(id).catch(() => {});
    }
  }, [hydrate]);

  const send = useCallback(async (text: string) => {
    let id = threadId;
    if (!id) {
      id = (await createThread()).thread_id;
      setThreadId(id);
      listThreads().then((r) => setThreadList(r.threads)).catch(() => {});
    }
    setMessages((p) => [...p, { id: crypto.randomUUID(), role: "user", text, toolCalls: [] }]);
    await consume(id!, `/threads/${id}/messages`, { content: text });
  }, [threadId, consume]);

  const respond = useCallback(
    (decision: { type: "accept" | "reject" | "edit"; args?: unknown }) => {
      if (!threadId) return;
      return consume(threadId, `/threads/${threadId}/resume`, { decision });
    }, [threadId, consume]);

  const openThread = useCallback(async (id: string) => {
    abort.current?.abort();
    setThreadId(id); setBriefs({}); setMessages([]);
    await hydrate(id);
  }, [hydrate]);

  const newThread = useCallback(() => {
    abort.current?.abort();
    setThreadId(null); setMessages([]); setTodos([]); setBriefs({}); setInterrupt(null);
  }, []);

  return {
    messages, todos, briefs, interrupt, error, isRunning,
    threadId, threadList, send, respond, openThread, newThread,
    cancel: () => abort.current?.abort(),
  };
}
```

### `components/Runtime.tsx`

```tsx
"use client";
import { createContext, useContext, type ReactNode } from "react";
import {
  AssistantRuntimeProvider, useExternalStoreRuntime,
  type AppendMessage, type ThreadMessageLike,
} from "@assistant-ui/react";
import { useDeepAgent, type Msg } from "@/lib/useDeepAgent";

const convertMessage = (m: Msg): ThreadMessageLike => ({
  id: m.id,
  role: m.role,
  content: [
    ...(m.text ? [{ type: "text" as const, text: m.text }] : []),
    ...m.toolCalls.map((t) => ({
      type: "tool-call" as const,
      toolCallId: t.id,
      toolName: t.name,
      args: t.args,
      result: t.result,
    })),
  ],
});

const Ctx = createContext<ReturnType<typeof useDeepAgent> | null>(null);
export const useAgent = () => useContext(Ctx)!;

export function RuntimeProvider({ children }: { children: ReactNode }) {
  const agent = useDeepAgent();
  const runtime = useExternalStoreRuntime({
    messages: agent.messages,
    isRunning: agent.isRunning,
    convertMessage,
    onNew: async (m: AppendMessage) => {
      const text = m.content.filter((c) => c.type === "text")
        .map((c) => (c as any).text).join("");
      await agent.send(text);
    },
    onCancel: async () => agent.cancel(),
  });

  return (
    <Ctx.Provider value={agent}>
      <AssistantRuntimeProvider runtime={runtime}>{children}</AssistantRuntimeProvider>
    </Ctx.Provider>
  );
}
```

### `components/SearchToolUI.tsx`

```tsx
"use client";
import { makeAssistantToolUI } from "@assistant-ui/react";

export const SearchToolUI = makeAssistantToolUI<{ query: string }, string>({
  toolName: "search_web",
  render: ({ args, result, status }) => {
    if (status.type === "running" || status.type === "requires_action")
      return <div className="my-1 text-sm opacity-60">Searching “{args?.query}”…</div>;
    if (status.type === "incomplete")
      return <div className="my-1 text-sm text-red-600">Search failed.</div>;
    return (
      <details className="my-1 rounded border px-2 py-1 text-sm">
        <summary className="cursor-pointer opacity-70">Searched “{args?.query}”</summary>
        <pre className="mt-1 max-h-48 overflow-auto text-xs">{result}</pre>
      </details>
    );
  },
});

export const TaskToolUI = makeAssistantToolUI<
  { description: string; subagent_type: string }, string
>({
  toolName: "task",
  render: ({ args, status }) => (
    <div className="my-1 text-sm opacity-70">
      {status.type === "running" ? "▶" : "✓"} {args?.subagent_type}: {args?.description}
    </div>
  ),
});
```

### `components/TodoPanel.tsx`

```tsx
"use client";
import { useAgent } from "./Runtime";

const mark = { pending: "○", in_progress: "◐", completed: "●" } as const;

export function TodoPanel() {
  const { todos } = useAgent();
  if (!todos.length) return null;
  return (
    <aside className="w-72 shrink-0 border-l p-4">
      <h2 className="mb-2 text-xs font-semibold uppercase tracking-wide opacity-60">Plan</h2>
      <ul className="space-y-1 text-sm">
        {todos.map((t, i) => (
          <li key={i} className={t.status === "completed" ? "opacity-50 line-through" : ""}>
            <span className="mr-1">{mark[t.status]}</span>{t.content}
          </li>
        ))}
      </ul>
    </aside>
  );
}
```

### `components/BriefCard.tsx`

```tsx
"use client";
import { useAgent } from "./Runtime";

export function BriefCard() {
  const { briefs, isRunning } = useAgent();
  const brief = Object.values(briefs).at(-1);
  if (!brief || isRunning) return null;   // structured may legitimately be absent

  return (
    <div className="mx-auto my-3 max-w-2xl rounded-xl border p-4">
      <div className="mb-1 flex items-baseline justify-between">
        <h3 className="font-medium">Brief</h3>
        <span className="text-xs opacity-60">
          confidence {(brief.confidence * 100).toFixed(0)}%
        </span>
      </div>
      <p className="text-sm">{brief.summary}</p>
      {brief.sources.length > 0 && (
        <ul className="mt-2 text-sm">
          {brief.sources.map((s) => (
            <li key={s.url}>
              <a className="underline" href={s.url} target="_blank" rel="noreferrer">
                {s.title}
              </a>
            </li>
          ))}
        </ul>
      )}
      {brief.open_questions.length > 0 && (
        <div className="mt-2 text-xs opacity-70">
          Open: {brief.open_questions.join(" · ")}
        </div>
      )}
    </div>
  );
}
```

### `components/ApprovalCard.tsx`

```tsx
"use client";
import { useState } from "react";
import { useAgent } from "./Runtime";

export function ApprovalCard() {
  const { interrupt, respond } = useAgent();
  const [busy, setBusy] = useState(false);
  if (!interrupt) return null;

  const req = interrupt?.action_requests?.[0] ?? interrupt;
  const act = async (type: "accept" | "reject") => {
    setBusy(true);
    try { await respond({ type }); } finally { setBusy(false); }
  };

  return (
    <div className="fixed bottom-28 left-1/2 z-50 -translate-x-1/2 rounded-xl border bg-white p-4 shadow-xl">
      <p className="font-medium">Run <code>{req?.action ?? "tool"}</code>?</p>
      <pre className="my-2 max-h-40 max-w-md overflow-auto rounded bg-neutral-50 p-2 text-xs">
        {JSON.stringify(req?.args, null, 2)}
      </pre>
      <div className="flex gap-2">
        <button disabled={busy} className="rounded bg-black px-3 py-1 text-sm text-white"
          onClick={() => act("accept")}>Approve</button>
        <button disabled={busy} className="rounded border px-3 py-1 text-sm"
          onClick={() => act("reject")}>Reject</button>
      </div>
    </div>
  );
}
```

### `components/ThreadSidebar.tsx`

```tsx
"use client";
import { useAgent } from "./Runtime";

export function ThreadSidebar() {
  const { threadList, threadId, openThread, newThread } = useAgent();
  return (
    <nav className="w-60 shrink-0 border-r p-3">
      <button onClick={newThread} className="mb-3 w-full rounded border px-2 py-1 text-sm">
        New chat
      </button>
      <ul className="space-y-1 text-sm">
        {threadList.map((t) => (
          <li key={t.thread_id}>
            <button
              onClick={() => openThread(t.thread_id)}
              className={`w-full truncate rounded px-2 py-1 text-left ${
                t.thread_id === threadId ? "bg-neutral-100 font-medium" : ""}`}>
              {t.title ?? "Untitled"}
            </button>
          </li>
        ))}
      </ul>
    </nav>
  );
}
```

### `app/page.tsx`

```tsx
import { Thread } from "@/components/assistant-ui/thread";
import { RuntimeProvider } from "@/components/Runtime";
import { ThreadSidebar } from "@/components/ThreadSidebar";
import { TodoPanel } from "@/components/TodoPanel";
import { BriefCard } from "@/components/BriefCard";
import { ApprovalCard } from "@/components/ApprovalCard";
import { SearchToolUI, TaskToolUI } from "@/components/SearchToolUI";

export default function Page() {
  return (
    <RuntimeProvider>
      <main className="flex h-dvh">
        <ThreadSidebar />
        <div className="flex min-w-0 flex-1 flex-col">
          <div className="min-h-0 flex-1"><Thread /></div>
          <BriefCard />
        </div>
        <TodoPanel />
      </main>
      <ApprovalCard />
      <SearchToolUI />
      <TaskToolUI />
    </RuntimeProvider>
  );
}
```

---

## 11. assistant-ui vs building your own

**Recommendation: use assistant-ui, exactly as above — `ExternalStoreRuntime` plus your own hook.**

The reasoning: `useDeepAgent` is the asset. It's yours, it's in your format, it doesn't know assistant-ui exists. Everything downstream is replaceable in a day. So take the free transcript rendering and spend your time on the parts that are actually your product — the plan panel, sub-agent visibility, approval flow, structured brief. Those are what make a deep agent UI feel different from a chat box, and no library gives you them.

**What you get for that `convertMessage` function:**

- Autoscroll that respects the user having scrolled up (more annoying than it sounds)
- Streaming markdown that doesn't break on half-written code fences
- Composer behavior: shift-enter, paste, disabled-while-running, focus management
- Message actions, copy, branch switching, keyboard nav, ARIA roles
- Tool call rendering slots via `makeAssistantToolUI`

**Honest caveat:** on the manual path you've already opted out of most of what makes the library valuable — LangGraph runtime, generative UI, checkpoint branching, thread persistence. You're getting a transcript renderer and a hook contract. Fair trade, but a narrower slice than the docs imply.

**When to roll your own instead:**

- Your UI isn't shaped like a chat. If plan/sub-agent/filesystem structure is the main surface and chat is a sidebar, you'll fight the abstraction for the 80% that isn't a transcript.
- You want primitives, not opinions — use `ThreadPrimitive` / `ComposerPrimitive` directly and gut the generated `thread.tsx`.

**Practical rules:**

- Pin `@assistant-ui/react` to an exact version. `unstable_` prefixes appear in the API surface and people have patched the package for subgraph interrupt handling. It's a young library moving fast.
- Treat the generated `thread.tsx` as your code. Rewrite freely — that's why the CLI copies it in.
- Keep `convertMessage` as the only place assistant-ui types appear outside components. That's your seam. Leaving means deleting `Runtime.tsx` and writing `<Transcript messages={messages} />`.
- Don't adopt anything else that touches state — no thread list adapter, no cloud persistence. You own threads server-side; a second source of truth is where this gets painful.

The decision is cheap either way: the runtime surface is ~30 lines, and your message state is already in your own format. Migrating in either direction is about a day. Revisit if the product turns out not to be chat-shaped.

---

## 12. Gotchas

**Backend**

- Don't pass a checkpointer when deploying to a managed LangGraph server — it supplies one. Do pass one on the manual path.
- `interrupt_on` payload shapes have shifted between deepagents releases. Log `intr.value` once and match your approval UI to what you actually get. Same for whether `resume` wants a list or a bare dict.
- `structured_response` is one slot; it does not accumulate per message.
- `structured_response` can be silently absent when tools are in play and prompts are long. Handle `null`.
- `X-Accel-Buffering: no` on SSE responses if nginx is in front.
- Sub-agents run as subgraphs — pass `subgraphs=True` to `astream` or you won't see their activity.

**Auth / multi-tenancy**

- The checkpointer does not know about users. Thread ownership is entirely your responsibility.
- 404 (not 403) on foreign threads.
- Never enumerate the checkpointer to build a thread list.
- Namespace `StoreBackend` by user id, or memory leaks across accounts.
- Module-scope caches keyed only by `thread_id` are a cross-user leak.
- Cookie auth + cross-origin needs `credentials: "include"`, `allow_credentials=True`, and an explicit origin list — `*` is rejected.

**Frontend**

- Tool UI components render nothing themselves; they must be mounted inside `RuntimeProvider` or registration never happens.
- Hide the structured card while `isRunning` — with `ToolStrategy` the object lands at the very end and a half-filled card looks broken.
- The post-stream reconcile must merge the brief map, not overwrite it from the `GET`.
- Reset all state on user change (`<RuntimeProvider key={userId}>`).
- Cancellation is client-abort-driven; the checkpointer keeps completed work and the reconcile resyncs.

**Not implemented here (deliberately)**

- Message editing / branching — needs a checkpoint-forking endpoint over `aget_state_history` with `checkpoint_id` in config.
- Thread titles — currently always `null`; generate one from the first user message.
- Attachments, file-tree viewer for the agent's virtual filesystem.
