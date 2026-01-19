# サーバー実装

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/server/

## 目次

- [概要](#概要)
- [基本アーキテクチャ](#基本アーキテクチャ)
- [FastAPI 実装](#fastapi-実装)
- [Django 実装](#django-実装)
- [セッション管理](#セッション管理)
- [認証と認可](#認証と認可)
- [スケーリング](#スケーリング)

---

## 概要

双方向ストリーミングサーバーは、クライアントとの持続的な接続を管理し、Agent のリアルタイム応答を配信する。

## 基本アーキテクチャ

```
┌────────────────────────────────────────────────────────┐
│                    Load Balancer                        │
└───────────────────────┬────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌───────────┐   ┌───────────┐   ┌───────────┐
│  Server 1 │   │  Server 2 │   │  Server 3 │
│  ┌─────┐  │   │  ┌─────┐  │   │  ┌─────┐  │
│  │Agent│  │   │  │Agent│  │   │  │Agent│  │
│  └─────┘  │   │  └─────┘  │   │  └─────┘  │
└───────────┘   └───────────┘   └───────────┘
        │               │               │
        └───────────────┼───────────────┘
                        │
                        ▼
            ┌───────────────────┐
            │   Message Queue   │
            │   (Redis/RabbitMQ)│
            └───────────────────┘
```

## FastAPI 実装

### 基本的なサーバー

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from strands import Agent
from typing import Dict
import json
import asyncio
from uuid import uuid4

app = FastAPI(title="Agent Streaming Server")

# CORS 設定
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Agent プール
agent_pool: Dict[str, Agent] = {}

def get_or_create_agent(session_id: str) -> Agent:
    if session_id not in agent_pool:
        agent_pool[session_id] = Agent(
            system_prompt="あなたは親切なアシスタントです。"
        )
    return agent_pool[session_id]

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    await websocket.accept()
    
    agent = get_or_create_agent(session_id)
    
    try:
        while True:
            # クライアントからメッセージを受信
            data = await websocket.receive_json()
            
            if data["type"] == "chat":
                # Agent でストリーミング処理
                async for event in agent.stream_async(data["content"]):
                    if "data" in event:
                        await websocket.send_json({
                            "type": "content",
                            "data": event["data"]
                        })
                    
                    if "tool_use" in event:
                        await websocket.send_json({
                            "type": "tool_use",
                            "data": event["tool_use"]
                        })
                
                await websocket.send_json({"type": "end"})
            
            elif data["type"] == "cancel":
                # キャンセル処理
                await websocket.send_json({
                    "type": "cancelled"
                })
                
    except WebSocketDisconnect:
        # クリーンアップ
        if session_id in agent_pool:
            del agent_pool[session_id]
```

### 高度な機能

```python
from fastapi import FastAPI, WebSocket, Depends, Query
from fastapi.security import OAuth2PasswordBearer
import jwt
from datetime import datetime, timedelta

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class ConnectionManager:
    def __init__(self):
        self.active_connections: Dict[str, WebSocket] = {}
        self.user_sessions: Dict[str, str] = {}
    
    async def connect(self, websocket: WebSocket, session_id: str, user_id: str):
        await websocket.accept()
        self.active_connections[session_id] = websocket
        self.user_sessions[session_id] = user_id
    
    def disconnect(self, session_id: str):
        if session_id in self.active_connections:
            del self.active_connections[session_id]
        if session_id in self.user_sessions:
            del self.user_sessions[session_id]
    
    async def send_to_session(self, session_id: str, message: dict):
        if session_id in self.active_connections:
            await self.active_connections[session_id].send_json(message)
    
    async def broadcast_to_user(self, user_id: str, message: dict):
        for session_id, uid in self.user_sessions.items():
            if uid == user_id:
                await self.send_to_session(session_id, message)

manager = ConnectionManager()

async def verify_token(token: str = Query(...)):
    try:
        payload = jwt.decode(token, "SECRET_KEY", algorithms=["HS256"])
        return payload["user_id"]
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(
    websocket: WebSocket,
    session_id: str,
    user_id: str = Depends(verify_token)
):
    await manager.connect(websocket, session_id, user_id)
    
    agent = get_or_create_agent(session_id)
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # メッセージ処理...
            
    except WebSocketDisconnect:
        manager.disconnect(session_id)
```

## Django 実装

### Django Channels

```python
# consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from strands import Agent

class AgentConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.session_id = self.scope['url_route']['kwargs']['session_id']
        self.agent = Agent(
            system_prompt="あなたは親切なアシスタントです。"
        )
        
        await self.accept()
    
    async def disconnect(self, close_code):
        pass
    
    async def receive(self, text_data):
        data = json.loads(text_data)
        
        if data["type"] == "chat":
            async for event in self.agent.stream_async(data["content"]):
                if "data" in event:
                    await self.send(json.dumps({
                        "type": "content",
                        "data": event["data"]
                    }))
            
            await self.send(json.dumps({"type": "end"}))

# routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/agent/(?P<session_id>\w+)/$', consumers.AgentConsumer.as_asgi()),
]
```

## セッション管理

### Redis ベースのセッション

```python
import aioredis
import json
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class SessionData:
    user_id: str
    created_at: str
    conversation_history: list
    agent_state: dict

class SessionManager:
    def __init__(self, redis_url: str = "redis://localhost"):
        self.redis_url = redis_url
        self.redis = None
    
    async def connect(self):
        self.redis = await aioredis.from_url(self.redis_url)
    
    async def create_session(self, session_id: str, user_id: str) -> SessionData:
        session = SessionData(
            user_id=user_id,
            created_at=datetime.now().isoformat(),
            conversation_history=[],
            agent_state={}
        )
        
        await self.redis.set(
            f"session:{session_id}",
            json.dumps(asdict(session)),
            ex=3600  # 1時間で有効期限切れ
        )
        
        return session
    
    async def get_session(self, session_id: str) -> Optional[SessionData]:
        data = await self.redis.get(f"session:{session_id}")
        
        if data:
            return SessionData(**json.loads(data))
        return None
    
    async def update_session(self, session_id: str, session: SessionData):
        await self.redis.set(
            f"session:{session_id}",
            json.dumps(asdict(session)),
            ex=3600
        )
    
    async def delete_session(self, session_id: str):
        await self.redis.delete(f"session:{session_id}")

# 使用例
session_manager = SessionManager()

@app.on_event("startup")
async def startup():
    await session_manager.connect()

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    session = await session_manager.get_session(session_id)
    
    if not session:
        await websocket.close(code=4001, reason="Session not found")
        return
    
    # ...
```

## 認証と認可

### JWT 認証

```python
from fastapi import WebSocket, HTTPException
from fastapi.security import HTTPBearer
import jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

def create_token(user_id: str) -> str:
    expire = datetime.utcnow() + timedelta(hours=24)
    payload = {
        "user_id": user_id,
        "exp": expire
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

async def authenticate_websocket(websocket: WebSocket) -> str:
    # クエリパラメータからトークンを取得
    token = websocket.query_params.get("token")
    
    if not token:
        await websocket.close(code=4001, reason="Missing token")
        return None
    
    try:
        payload = verify_token(token)
        return payload["user_id"]
    except HTTPException:
        await websocket.close(code=4001, reason="Invalid token")
        return None
```

## スケーリング

### Pub/Sub パターン

```python
import aioredis

class PubSubManager:
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.pubsub = None
    
    async def connect(self):
        redis = await aioredis.from_url(self.redis_url)
        self.pubsub = redis.pubsub()
    
    async def subscribe(self, channel: str):
        await self.pubsub.subscribe(channel)
    
    async def publish(self, channel: str, message: dict):
        redis = await aioredis.from_url(self.redis_url)
        await redis.publish(channel, json.dumps(message))
    
    async def listen(self):
        async for message in self.pubsub.listen():
            if message["type"] == "message":
                yield json.loads(message["data"])

# WebSocket エンドポイントでの使用
@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    await websocket.accept()
    
    pubsub = PubSubManager("redis://localhost")
    await pubsub.connect()
    await pubsub.subscribe(f"session:{session_id}")
    
    async def receive_messages():
        async for message in pubsub.listen():
            await websocket.send_json(message)
    
    receive_task = asyncio.create_task(receive_messages())
    
    try:
        while True:
            data = await websocket.receive_json()
            # メッセージを他のサーバーにも配信
            await pubsub.publish(f"session:{session_id}", data)
    finally:
        receive_task.cancel()
```
