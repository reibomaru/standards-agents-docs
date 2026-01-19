# Server-Sent Events (SSE)

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/sse/

## 目次

- [概要](#概要)
- [SSE の基本](#sse-の基本)
- [サーバー実装](#サーバー実装)
- [クライアント実装](#クライアント実装)
- [SSE + POST による双方向通信](#sse--post-による双方向通信)
- [制限事項と対策](#制限事項と対策)

---

## 概要

Server-Sent Events (SSE) は、サーバーからクライアントへの単方向ストリーミングを実現する標準的な Web 技術。HTTP/1.1 上で動作し、シンプルな実装が可能。

### SSE vs WebSocket

| 特徴 | SSE | WebSocket |
|------|-----|-----------|
| 方向 | サーバー → クライアント | 双方向 |
| プロトコル | HTTP | WebSocket |
| 再接続 | 自動 | 手動実装 |
| ブラウザサポート | 良好 | 良好 |
| ファイアウォール | 通過しやすい | ブロックされる場合あり |

## SSE の基本

### イベントフォーマット

```
event: message
id: 1
data: {"content": "こんにちは"}

event: tool_use
id: 2
data: {"tool": "search", "input": {"query": "天気"}}

event: end
id: 3
data: {}
```

### フィールド

- `event`: イベントタイプ（オプション）
- `id`: イベント ID（再接続時に使用）
- `data`: イベントデータ
- `retry`: 再接続間隔（ミリ秒）

## サーバー実装

### FastAPI での SSE

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from strands import Agent
import json
import asyncio

app = FastAPI()

agent = Agent(
    system_prompt="あなたは親切なアシスタントです。"
)

async def event_generator(prompt: str):
    """SSE イベントを生成"""
    async for event in agent.stream_async(prompt):
        if "data" in event:
            yield f"event: content\n"
            yield f"data: {json.dumps({'content': event['data']})}\n\n"
        
        if "tool_use" in event:
            yield f"event: tool_use\n"
            yield f"data: {json.dumps(event['tool_use'])}\n\n"
    
    yield f"event: end\n"
    yield f"data: {{}}\n\n"

@app.get("/stream")
async def stream_agent(prompt: str):
    return StreamingResponse(
        event_generator(prompt),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"
        }
    )
```

### Starlette での SSE

```python
from starlette.applications import Starlette
from starlette.responses import StreamingResponse
from starlette.routing import Route
from strands import Agent
import json

agent = Agent(system_prompt="あなたはアシスタントです。")

async def sse_endpoint(request):
    prompt = request.query_params.get("prompt", "")
    
    async def generate():
        async for event in agent.stream_async(prompt):
            if "data" in event:
                yield f"data: {json.dumps({'type': 'content', 'data': event['data']})}\n\n"
        
        yield f"data: {json.dumps({'type': 'end'})}\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )

app = Starlette(routes=[
    Route("/stream", sse_endpoint)
])
```

### Flask での SSE

```python
from flask import Flask, Response, request
from strands import Agent
import json

app = Flask(__name__)
agent = Agent(system_prompt="あなたはアシスタントです。")

def generate_events(prompt: str):
    for event in agent.stream(prompt):
        if "data" in event:
            yield f"data: {json.dumps({'type': 'content', 'data': event['data']})}\n\n"
    
    yield f"data: {json.dumps({'type': 'end'})}\n\n"

@app.route('/stream')
def stream():
    prompt = request.args.get('prompt', '')
    return Response(
        generate_events(prompt),
        mimetype='text/event-stream'
    )
```

## クライアント実装

### JavaScript (ブラウザ)

```javascript
class SSEClient {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
        this.eventSource = null;
    }
    
    connect(prompt, handlers) {
        const url = `${this.baseUrl}/stream?prompt=${encodeURIComponent(prompt)}`;
        this.eventSource = new EventSource(url);
        
        // 通常のメッセージ
        this.eventSource.onmessage = (event) => {
            const data = JSON.parse(event.data);
            if (handlers.onMessage) {
                handlers.onMessage(data);
            }
        };
        
        // 名前付きイベント
        this.eventSource.addEventListener('content', (event) => {
            const data = JSON.parse(event.data);
            if (handlers.onContent) {
                handlers.onContent(data.content);
            }
        });
        
        this.eventSource.addEventListener('tool_use', (event) => {
            const data = JSON.parse(event.data);
            if (handlers.onToolUse) {
                handlers.onToolUse(data);
            }
        });
        
        this.eventSource.addEventListener('end', () => {
            if (handlers.onEnd) {
                handlers.onEnd();
            }
            this.close();
        });
        
        // エラーハンドリング
        this.eventSource.onerror = (error) => {
            if (handlers.onError) {
                handlers.onError(error);
            }
        };
    }
    
    close() {
        if (this.eventSource) {
            this.eventSource.close();
            this.eventSource = null;
        }
    }
}

// 使用例
const client = new SSEClient('http://localhost:8000');

client.connect('こんにちは', {
    onContent: (content) => {
        document.getElementById('output').textContent += content;
    },
    onEnd: () => {
        console.log('ストリーミング完了');
    },
    onError: (error) => {
        console.error('エラー:', error);
    }
});
```

### Python クライアント

```python
import aiohttp
import asyncio
import json

class SSEClient:
    def __init__(self, base_url: str):
        self.base_url = base_url
    
    async def stream(self, prompt: str):
        url = f"{self.base_url}/stream?prompt={prompt}"
        
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                async for line in response.content:
                    line = line.decode('utf-8').strip()
                    
                    if line.startswith('data:'):
                        data = json.loads(line[5:].strip())
                        yield data

# 使用例
async def main():
    client = SSEClient("http://localhost:8000")
    
    async for event in client.stream("こんにちは"):
        if event["type"] == "content":
            print(event["data"], end="", flush=True)
        elif event["type"] == "end":
            print()
            break

asyncio.run(main())
```

## SSE + POST による双方向通信

SSE は単方向のため、双方向通信には POST リクエストと組み合わせる：

### サーバー実装

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from strands import Agent
import json
import asyncio
from uuid import uuid4

app = FastAPI()

# セッション管理
sessions = {}

class Session:
    def __init__(self):
        self.agent = Agent(system_prompt="あなたはアシスタントです。")
        self.message_queue = asyncio.Queue()
        self.response_queue = asyncio.Queue()

@app.post("/session")
async def create_session():
    session_id = str(uuid4())
    sessions[session_id] = Session()
    return {"session_id": session_id}

@app.post("/message/{session_id}")
async def send_message(session_id: str, request: Request):
    data = await request.json()
    session = sessions.get(session_id)
    
    if not session:
        return {"error": "Session not found"}
    
    await session.message_queue.put(data["content"])
    return {"status": "ok"}

@app.get("/stream/{session_id}")
async def stream_session(session_id: str):
    session = sessions.get(session_id)
    
    if not session:
        return {"error": "Session not found"}
    
    async def generate():
        while True:
            # メッセージを待つ
            message = await session.message_queue.get()
            
            # Agent で処理
            async for event in session.agent.stream_async(message):
                if "data" in event:
                    yield f"data: {json.dumps({'type': 'content', 'data': event['data']})}\n\n"
            
            yield f"data: {json.dumps({'type': 'message_end'})}\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )
```

### クライアント実装

```javascript
class BidirectionalSSEClient {
    constructor(baseUrl) {
        this.baseUrl = baseUrl;
        this.sessionId = null;
        this.eventSource = null;
    }
    
    async createSession() {
        const response = await fetch(`${this.baseUrl}/session`, {
            method: 'POST'
        });
        const data = await response.json();
        this.sessionId = data.session_id;
        return this.sessionId;
    }
    
    connect(handlers) {
        const url = `${this.baseUrl}/stream/${this.sessionId}`;
        this.eventSource = new EventSource(url);
        
        this.eventSource.onmessage = (event) => {
            const data = JSON.parse(event.data);
            
            if (data.type === 'content' && handlers.onContent) {
                handlers.onContent(data.data);
            } else if (data.type === 'message_end' && handlers.onMessageEnd) {
                handlers.onMessageEnd();
            }
        };
        
        this.eventSource.onerror = handlers.onError;
    }
    
    async send(message) {
        await fetch(`${this.baseUrl}/message/${this.sessionId}`, {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({ content: message })
        });
    }
    
    close() {
        if (this.eventSource) {
            this.eventSource.close();
        }
    }
}

// 使用例
async function main() {
    const client = new BidirectionalSSEClient('http://localhost:8000');
    
    await client.createSession();
    
    client.connect({
        onContent: (content) => console.log(content),
        onMessageEnd: () => console.log('\n--- メッセージ終了 ---')
    });
    
    await client.send('こんにちは');
}

main();
```

## 制限事項と対策

### 接続数の制限

ブラウザは同一ドメインへの SSE 接続数を制限（通常6接続）：

```javascript
// HTTP/2 を使用することで制限を緩和
// サーバー側で HTTP/2 を有効化

// または、共有接続を使用
class SharedSSEConnection {
    static instance = null;
    
    static getInstance(url) {
        if (!SharedSSEConnection.instance) {
            SharedSSEConnection.instance = new EventSource(url);
        }
        return SharedSSEConnection.instance;
    }
}
```

### タイムアウト対策

```python
async def event_generator_with_keepalive(prompt: str):
    last_event_time = time.time()
    
    async for event in agent.stream_async(prompt):
        yield f"data: {json.dumps(event)}\n\n"
        last_event_time = time.time()
    
    # キープアライブ
    while True:
        if time.time() - last_event_time > 15:
            yield f": keepalive\n\n"
            last_event_time = time.time()
        await asyncio.sleep(1)
```
