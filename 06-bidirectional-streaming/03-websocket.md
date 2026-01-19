# WebSocket 統合

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/websocket/

## 目次

- [概要](#概要)
- [WebSocket の基本](#websocket-の基本)
- [サーバー実装](#サーバー実装)
- [クライアント実装](#クライアント実装)
- [メッセージプロトコル](#メッセージプロトコル)
- [接続管理](#接続管理)
- [スケーリング](#スケーリング)

---

## 概要

WebSocket は、HTTP コネクションをアップグレードして全二重通信を実現するプロトコル。Agent との双方向リアルタイム通信に最適。

### WebSocket の利点

- **持続的接続**: 一度接続すれば、複数のメッセージを送受信可能
- **低レイテンシ**: HTTP のオーバーヘッドなし
- **全二重通信**: 同時に送信と受信が可能
- **広範なサポート**: すべてのモダンブラウザと環境でサポート

## WebSocket の基本

### プロトコルフロー

```
1. HTTP Handshake
   Client: GET /ws HTTP/1.1
           Upgrade: websocket
           Connection: Upgrade
   
   Server: HTTP/1.1 101 Switching Protocols
           Upgrade: websocket
           Connection: Upgrade

2. WebSocket Connection Established
   Client <────────────────> Server
          双方向メッセージング
```

### 接続状態

```python
from enum import Enum

class ConnectionState(Enum):
    CONNECTING = 0  # 接続中
    OPEN = 1        # 接続済み
    CLOSING = 2     # 切断中
    CLOSED = 3      # 切断済み
```

## サーバー実装

### FastAPI での実装

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from strands import Agent
from typing import List
import json

app = FastAPI()

# 接続管理
class ConnectionManager:
    def __init__(self):
        self.active_connections: List[WebSocket] = []
    
    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)
    
    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)
    
    async def send_personal(self, message: str, websocket: WebSocket):
        await websocket.send_text(message)
    
    async def broadcast(self, message: str):
        for connection in self.active_connections:
            await connection.send_text(message)

manager = ConnectionManager()

# Agent インスタンス
agent = Agent(
    system_prompt="あなたは親切なアシスタントです。"
)

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    
    try:
        while True:
            # メッセージを受信
            data = await websocket.receive_text()
            message = json.loads(data)
            
            # Agent で処理
            async for event in agent.stream_async(message["content"]):
                if "data" in event:
                    await manager.send_personal(
                        json.dumps({"type": "content", "data": event["data"]}),
                        websocket
                    )
            
            # 完了通知
            await manager.send_personal(
                json.dumps({"type": "complete"}),
                websocket
            )
            
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

### Starlette での実装

```python
from starlette.applications import Starlette
from starlette.routing import WebSocketRoute
from starlette.websockets import WebSocket
from strands import Agent
import json

agent = Agent(system_prompt="あなたはアシスタントです。")

async def agent_websocket(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_json()
            
            async for event in agent.stream_async(data["content"]):
                if "data" in event:
                    await websocket.send_json({
                        "type": "content",
                        "data": event["data"]
                    })
            
            await websocket.send_json({"type": "end"})
            
    except Exception as e:
        print(f"Error: {e}")
    finally:
        await websocket.close()

app = Starlette(routes=[
    WebSocketRoute("/ws", agent_websocket)
])
```

## クライアント実装

### Python クライアント

```python
import asyncio
import websockets
import json

class AgentWebSocketClient:
    def __init__(self, url: str):
        self.url = url
        self.websocket = None
    
    async def connect(self):
        self.websocket = await websockets.connect(self.url)
    
    async def disconnect(self):
        if self.websocket:
            await self.websocket.close()
    
    async def send(self, message: str):
        await self.websocket.send(json.dumps({
            "content": message
        }))
    
    async def receive(self):
        async for message in self.websocket:
            data = json.loads(message)
            yield data
    
    async def chat(self, message: str):
        await self.send(message)
        
        response = ""
        async for data in self.receive():
            if data["type"] == "content":
                response += data["data"]
                yield data["data"]
            elif data["type"] == "end":
                break
        
        return response

# 使用例
async def main():
    client = AgentWebSocketClient("ws://localhost:8000/ws/user123")
    await client.connect()
    
    try:
        async for chunk in client.chat("こんにちは"):
            print(chunk, end="", flush=True)
        print()
    finally:
        await client.disconnect()

asyncio.run(main())
```

### JavaScript クライアント

```javascript
class AgentWebSocketClient {
    constructor(url) {
        this.url = url;
        this.socket = null;
        this.messageCallbacks = [];
    }
    
    connect() {
        return new Promise((resolve, reject) => {
            this.socket = new WebSocket(this.url);
            
            this.socket.onopen = () => resolve();
            this.socket.onerror = (error) => reject(error);
            
            this.socket.onmessage = (event) => {
                const data = JSON.parse(event.data);
                this.messageCallbacks.forEach(cb => cb(data));
            };
        });
    }
    
    disconnect() {
        if (this.socket) {
            this.socket.close();
        }
    }
    
    send(message) {
        this.socket.send(JSON.stringify({
            content: message
        }));
    }
    
    onMessage(callback) {
        this.messageCallbacks.push(callback);
    }
    
    async chat(message) {
        return new Promise((resolve) => {
            let response = '';
            
            const handler = (data) => {
                if (data.type === 'content') {
                    response += data.data;
                } else if (data.type === 'end') {
                    this.messageCallbacks = this.messageCallbacks
                        .filter(cb => cb !== handler);
                    resolve(response);
                }
            };
            
            this.onMessage(handler);
            this.send(message);
        });
    }
}

// 使用例
async function main() {
    const client = new AgentWebSocketClient('ws://localhost:8000/ws/user123');
    await client.connect();
    
    client.onMessage((data) => {
        if (data.type === 'content') {
            process.stdout.write(data.data);
        }
    });
    
    await client.chat('こんにちは');
    console.log();
    
    client.disconnect();
}

main();
```

## メッセージプロトコル

### メッセージ形式

```typescript
// クライアント → サーバー
interface ClientMessage {
    type: "chat" | "control" | "ping";
    content?: string;
    metadata?: Record<string, any>;
}

// サーバー → クライアント
interface ServerMessage {
    type: "content" | "tool_use" | "tool_result" | "error" | "end" | "pong";
    data?: string;
    metadata?: Record<string, any>;
}
```

### メッセージタイプ

```python
from enum import Enum

class ClientMessageType(Enum):
    CHAT = "chat"           # 通常のチャットメッセージ
    CONTROL = "control"     # 制御メッセージ（キャンセルなど）
    PING = "ping"           # 生存確認

class ServerMessageType(Enum):
    CONTENT = "content"     # テキストコンテンツ
    TOOL_USE = "tool_use"   # ツール呼び出し
    TOOL_RESULT = "tool_result"  # ツール結果
    ERROR = "error"         # エラー
    END = "end"             # メッセージ終了
    PONG = "pong"           # ping への応答
```

## 接続管理

### 再接続ロジック

```python
import asyncio
import websockets
from websockets.exceptions import ConnectionClosed

class ReconnectingWebSocket:
    def __init__(self, url: str, max_retries: int = 5):
        self.url = url
        self.max_retries = max_retries
        self.websocket = None
        self.connected = False
    
    async def connect(self):
        retries = 0
        
        while retries < self.max_retries:
            try:
                self.websocket = await websockets.connect(self.url)
                self.connected = True
                return True
            except Exception as e:
                retries += 1
                wait_time = min(2 ** retries, 30)  # 指数バックオフ
                print(f"接続失敗、{wait_time}秒後にリトライ...")
                await asyncio.sleep(wait_time)
        
        return False
    
    async def ensure_connected(self):
        if not self.connected:
            await self.connect()
    
    async def send(self, message: str):
        await self.ensure_connected()
        try:
            await self.websocket.send(message)
        except ConnectionClosed:
            self.connected = False
            await self.ensure_connected()
            await self.websocket.send(message)
```

### ハートビート

```python
import asyncio

class HeartbeatWebSocket:
    def __init__(self, websocket, interval: int = 30):
        self.websocket = websocket
        self.interval = interval
        self.heartbeat_task = None
    
    async def start_heartbeat(self):
        self.heartbeat_task = asyncio.create_task(self._heartbeat_loop())
    
    async def stop_heartbeat(self):
        if self.heartbeat_task:
            self.heartbeat_task.cancel()
    
    async def _heartbeat_loop(self):
        while True:
            try:
                await asyncio.sleep(self.interval)
                await self.websocket.send('{"type": "ping"}')
            except Exception:
                break
```

## スケーリング

### Redis を使用した Pub/Sub

```python
import aioredis
from fastapi import FastAPI, WebSocket

app = FastAPI()

@app.on_event("startup")
async def startup():
    app.state.redis = await aioredis.create_redis_pool("redis://localhost")

@app.websocket("/ws/{channel}")
async def websocket_endpoint(websocket: WebSocket, channel: str):
    await websocket.accept()
    redis = app.state.redis
    
    # チャンネルを購読
    channel_sub = (await redis.subscribe(channel))[0]
    
    async def reader():
        async for message in channel_sub.iter():
            await websocket.send_text(message.decode())
    
    reader_task = asyncio.create_task(reader())
    
    try:
        while True:
            data = await websocket.receive_text()
            await redis.publish(channel, data)
    finally:
        reader_task.cancel()
        await redis.unsubscribe(channel)
```
