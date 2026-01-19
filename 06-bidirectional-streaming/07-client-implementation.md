# クライアント実装

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/client/

## 目次

- [概要](#概要)
- [Python クライアント](#python-クライアント)
- [JavaScript クライアント](#javascript-クライアント)
- [React フック](#react-フック)
- [接続管理](#接続管理)
- [エラーハンドリング](#エラーハンドリング)

---

## 概要

双方向ストリーミングクライアントは、サーバーとの持続的な接続を維持し、メッセージの送受信を非同期で処理する。

## Python クライアント

### 基本クライアント

```python
import asyncio
import websockets
import json
from typing import AsyncGenerator, Callable, Optional

class AgentClient:
    def __init__(self, url: str):
        self.url = url
        self.websocket = None
        self.connected = False
    
    async def connect(self):
        """サーバーに接続"""
        self.websocket = await websockets.connect(self.url)
        self.connected = True
    
    async def disconnect(self):
        """接続を切断"""
        if self.websocket:
            await self.websocket.close()
            self.connected = False
    
    async def send(self, message: str):
        """メッセージを送信"""
        await self.websocket.send(json.dumps({
            "type": "chat",
            "content": message
        }))
    
    async def receive(self) -> AsyncGenerator[dict, None]:
        """メッセージを受信"""
        async for message in self.websocket:
            data = json.loads(message)
            yield data
            
            if data["type"] == "end":
                break
    
    async def chat(self, message: str) -> AsyncGenerator[str, None]:
        """チャットメッセージを送信し、応答をストリーム"""
        await self.send(message)
        
        async for data in self.receive():
            if data["type"] == "content":
                yield data["data"]

# 使用例
async def main():
    client = AgentClient("ws://localhost:8000/ws/session123")
    await client.connect()
    
    try:
        async for chunk in client.chat("こんにちは"):
            print(chunk, end="", flush=True)
        print()
    finally:
        await client.disconnect()

asyncio.run(main())
```

### 高度なクライアント

```python
import asyncio
import websockets
import json
from dataclasses import dataclass
from typing import Optional, Callable, Dict, Any
from enum import Enum

class EventType(Enum):
    CONTENT = "content"
    TOOL_USE = "tool_use"
    TOOL_RESULT = "tool_result"
    ERROR = "error"
    END = "end"

@dataclass
class StreamEvent:
    type: EventType
    data: Any

class AdvancedAgentClient:
    def __init__(
        self,
        url: str,
        reconnect: bool = True,
        max_retries: int = 5
    ):
        self.url = url
        self.reconnect = reconnect
        self.max_retries = max_retries
        self.websocket = None
        self.connected = False
        self.handlers: Dict[EventType, Callable] = {}
    
    def on(self, event_type: EventType, handler: Callable):
        """イベントハンドラを登録"""
        self.handlers[event_type] = handler
    
    async def connect(self):
        """接続（再試行付き）"""
        retries = 0
        
        while retries < self.max_retries:
            try:
                self.websocket = await websockets.connect(
                    self.url,
                    ping_interval=30,
                    ping_timeout=10
                )
                self.connected = True
                return
            except Exception as e:
                retries += 1
                wait = min(2 ** retries, 30)
                print(f"接続失敗、{wait}秒後にリトライ...")
                await asyncio.sleep(wait)
        
        raise ConnectionError("接続に失敗しました")
    
    async def _handle_message(self, message: str):
        """メッセージを処理"""
        data = json.loads(message)
        event_type = EventType(data["type"])
        event = StreamEvent(type=event_type, data=data.get("data"))
        
        if event_type in self.handlers:
            await self.handlers[event_type](event)
    
    async def listen(self):
        """メッセージをリッスン"""
        while self.connected:
            try:
                message = await self.websocket.recv()
                await self._handle_message(message)
            except websockets.ConnectionClosed:
                self.connected = False
                if self.reconnect:
                    await self.connect()
                else:
                    break
    
    async def send_chat(self, content: str):
        """チャットメッセージを送信"""
        await self.websocket.send(json.dumps({
            "type": "chat",
            "content": content
        }))
    
    async def cancel(self):
        """現在の処理をキャンセル"""
        await self.websocket.send(json.dumps({
            "type": "cancel"
        }))

# 使用例
async def main():
    client = AdvancedAgentClient("ws://localhost:8000/ws/session123")
    
    client.on(EventType.CONTENT, lambda e: print(e.data, end="", flush=True))
    client.on(EventType.END, lambda e: print())
    client.on(EventType.ERROR, lambda e: print(f"Error: {e.data}"))
    
    await client.connect()
    
    # リスナーを開始
    listen_task = asyncio.create_task(client.listen())
    
    # メッセージを送信
    await client.send_chat("こんにちは")
    
    # 応答を待つ
    await asyncio.sleep(10)
    listen_task.cancel()

asyncio.run(main())
```

## JavaScript クライアント

### 基本クライアント

```javascript
class AgentClient {
    constructor(url) {
        this.url = url;
        this.socket = null;
        this.connected = false;
        this.messageHandlers = [];
    }
    
    connect() {
        return new Promise((resolve, reject) => {
            this.socket = new WebSocket(this.url);
            
            this.socket.onopen = () => {
                this.connected = true;
                resolve();
            };
            
            this.socket.onerror = (error) => {
                reject(error);
            };
            
            this.socket.onclose = () => {
                this.connected = false;
            };
            
            this.socket.onmessage = (event) => {
                const data = JSON.parse(event.data);
                this.messageHandlers.forEach(handler => handler(data));
            };
        });
    }
    
    disconnect() {
        if (this.socket) {
            this.socket.close();
        }
    }
    
    onMessage(handler) {
        this.messageHandlers.push(handler);
        return () => {
            this.messageHandlers = this.messageHandlers.filter(h => h !== handler);
        };
    }
    
    send(message) {
        this.socket.send(JSON.stringify({
            type: 'chat',
            content: message
        }));
    }
    
    async chat(message) {
        return new Promise((resolve) => {
            let response = '';
            
            const unsubscribe = this.onMessage((data) => {
                if (data.type === 'content') {
                    response += data.data;
                } else if (data.type === 'end') {
                    unsubscribe();
                    resolve(response);
                }
            });
            
            this.send(message);
        });
    }
}

// 使用例
async function main() {
    const client = new AgentClient('ws://localhost:8000/ws/session123');
    await client.connect();
    
    client.onMessage((data) => {
        if (data.type === 'content') {
            process.stdout.write(data.data);
        } else if (data.type === 'end') {
            console.log();
        }
    });
    
    await client.chat('こんにちは');
    client.disconnect();
}

main();
```

### TypeScript クライアント

```typescript
interface StreamEvent {
    type: 'content' | 'tool_use' | 'tool_result' | 'error' | 'end';
    data?: any;
}

type EventHandler = (event: StreamEvent) => void;

class TypedAgentClient {
    private url: string;
    private socket: WebSocket | null = null;
    private connected: boolean = false;
    private handlers: Map<string, EventHandler[]> = new Map();
    
    constructor(url: string) {
        this.url = url;
    }
    
    async connect(): Promise<void> {
        return new Promise((resolve, reject) => {
            this.socket = new WebSocket(this.url);
            
            this.socket.onopen = () => {
                this.connected = true;
                resolve();
            };
            
            this.socket.onerror = reject;
            
            this.socket.onmessage = (event) => {
                const data: StreamEvent = JSON.parse(event.data);
                this.emit(data.type, data);
            };
        });
    }
    
    on(eventType: string, handler: EventHandler): () => void {
        if (!this.handlers.has(eventType)) {
            this.handlers.set(eventType, []);
        }
        this.handlers.get(eventType)!.push(handler);
        
        return () => {
            const handlers = this.handlers.get(eventType);
            if (handlers) {
                const index = handlers.indexOf(handler);
                if (index > -1) {
                    handlers.splice(index, 1);
                }
            }
        };
    }
    
    private emit(eventType: string, event: StreamEvent): void {
        const handlers = this.handlers.get(eventType);
        if (handlers) {
            handlers.forEach(handler => handler(event));
        }
    }
    
    send(content: string): void {
        this.socket?.send(JSON.stringify({
            type: 'chat',
            content
        }));
    }
}
```

## React フック

### useAgent フック

```typescript
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseAgentOptions {
    url: string;
    autoConnect?: boolean;
}

interface UseAgentReturn {
    connected: boolean;
    loading: boolean;
    response: string;
    error: string | null;
    send: (message: string) => void;
    connect: () => Promise<void>;
    disconnect: () => void;
}

export function useAgent({ url, autoConnect = true }: UseAgentOptions): UseAgentReturn {
    const [connected, setConnected] = useState(false);
    const [loading, setLoading] = useState(false);
    const [response, setResponse] = useState('');
    const [error, setError] = useState<string | null>(null);
    
    const socketRef = useRef<WebSocket | null>(null);
    
    const connect = useCallback(async () => {
        return new Promise<void>((resolve, reject) => {
            const socket = new WebSocket(url);
            
            socket.onopen = () => {
                setConnected(true);
                socketRef.current = socket;
                resolve();
            };
            
            socket.onerror = (e) => {
                setError('Connection failed');
                reject(e);
            };
            
            socket.onclose = () => {
                setConnected(false);
            };
            
            socket.onmessage = (event) => {
                const data = JSON.parse(event.data);
                
                if (data.type === 'content') {
                    setResponse(prev => prev + data.data);
                } else if (data.type === 'end') {
                    setLoading(false);
                } else if (data.type === 'error') {
                    setError(data.data);
                    setLoading(false);
                }
            };
        });
    }, [url]);
    
    const disconnect = useCallback(() => {
        socketRef.current?.close();
    }, []);
    
    const send = useCallback((message: string) => {
        setResponse('');
        setError(null);
        setLoading(true);
        
        socketRef.current?.send(JSON.stringify({
            type: 'chat',
            content: message
        }));
    }, []);
    
    useEffect(() => {
        if (autoConnect) {
            connect();
        }
        
        return () => {
            disconnect();
        };
    }, [autoConnect, connect, disconnect]);
    
    return {
        connected,
        loading,
        response,
        error,
        send,
        connect,
        disconnect
    };
}

// 使用例
function ChatComponent() {
    const { connected, loading, response, error, send } = useAgent({
        url: 'ws://localhost:8000/ws/session123'
    });
    
    const [input, setInput] = useState('');
    
    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();
        send(input);
        setInput('');
    };
    
    return (
        <div>
            <div>{connected ? '接続中' : '切断'}</div>
            
            {error && <div style={{ color: 'red' }}>{error}</div>}
            
            <div>{response}</div>
            
            <form onSubmit={handleSubmit}>
                <input
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    disabled={!connected || loading}
                />
                <button type="submit" disabled={!connected || loading}>
                    {loading ? '送信中...' : '送信'}
                </button>
            </form>
        </div>
    );
}
```

## 接続管理

### 再接続ロジック

```typescript
class ReconnectingClient {
    private url: string;
    private socket: WebSocket | null = null;
    private reconnectAttempts = 0;
    private maxReconnectAttempts = 5;
    private reconnectDelay = 1000;
    
    constructor(url: string) {
        this.url = url;
    }
    
    connect(): void {
        this.socket = new WebSocket(this.url);
        
        this.socket.onclose = () => {
            this.handleDisconnect();
        };
    }
    
    private handleDisconnect(): void {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts);
            
            setTimeout(() => {
                this.reconnectAttempts++;
                this.connect();
            }, delay);
        }
    }
}
```

## エラーハンドリング

```typescript
class RobustAgentClient {
    async send(message: string): Promise<void> {
        try {
            this.socket?.send(JSON.stringify({
                type: 'chat',
                content: message
            }));
        } catch (error) {
            if (error instanceof DOMException) {
                // WebSocket エラー
                await this.reconnect();
                await this.send(message);
            } else {
                throw error;
            }
        }
    }
    
    private async reconnect(): Promise<void> {
        this.disconnect();
        await this.connect();
    }
}
```
