# イベントタイプ

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/events/

## 目次

- [概要](#概要)
- [基本イベント](#基本イベント)
- [Agent イベント](#agent-イベント)
- [制御イベント](#制御イベント)
- [カスタムイベント](#カスタムイベント)
- [イベント処理パターン](#イベント処理パターン)

---

## 概要

双方向ストリーミングでは、様々なイベントタイプを使用してクライアントとサーバー間の通信を構造化する。

## 基本イベント

### イベント構造

```typescript
interface BaseEvent {
    type: string;          // イベントタイプ
    id?: string;           // イベント ID
    timestamp?: string;    // タイムスタンプ
    metadata?: Record<string, any>;
}
```

### クライアント → サーバー

```typescript
// チャットメッセージ
interface ChatEvent extends BaseEvent {
    type: 'chat';
    content: string;
    attachments?: Attachment[];
}

// 制御メッセージ
interface ControlEvent extends BaseEvent {
    type: 'control';
    action: 'cancel' | 'pause' | 'resume';
}

// Ping
interface PingEvent extends BaseEvent {
    type: 'ping';
}

// ツール結果（Human-in-the-Loop）
interface ToolResultEvent extends BaseEvent {
    type: 'tool_result';
    tool_id: string;
    result: any;
}
```

### サーバー → クライアント

```typescript
// コンテンツチャンク
interface ContentEvent extends BaseEvent {
    type: 'content';
    data: string;
    role?: 'assistant' | 'system';
}

// ツール使用
interface ToolUseEvent extends BaseEvent {
    type: 'tool_use';
    tool_name: string;
    tool_id: string;
    input: any;
}

// ツール結果
interface ToolResultEvent extends BaseEvent {
    type: 'tool_result';
    tool_id: string;
    output: any;
}

// エラー
interface ErrorEvent extends BaseEvent {
    type: 'error';
    code: string;
    message: string;
    recoverable: boolean;
}

// 終了
interface EndEvent extends BaseEvent {
    type: 'end';
    reason?: 'complete' | 'cancelled' | 'error';
}

// Pong
interface PongEvent extends BaseEvent {
    type: 'pong';
}
```

## Agent イベント

### コンテンツ生成

```python
# サーバー側
async def stream_agent_events(prompt: str):
    async for event in agent.stream_async(prompt):
        if "data" in event:
            yield {
                "type": "content",
                "data": event["data"],
                "timestamp": datetime.now().isoformat()
            }
```

### ツール実行

```python
async def stream_with_tools(prompt: str):
    async for event in agent.stream_async(prompt):
        # テキストコンテンツ
        if "data" in event:
            yield {
                "type": "content",
                "data": event["data"]
            }
        
        # ツール呼び出し開始
        if "tool_use" in event:
            tool = event["tool_use"]
            yield {
                "type": "tool_use",
                "tool_name": tool["name"],
                "tool_id": tool["id"],
                "input": tool["input"]
            }
        
        # ツール結果
        if "tool_result" in event:
            result = event["tool_result"]
            yield {
                "type": "tool_result",
                "tool_id": result["id"],
                "output": result["output"]
            }
```

### 思考プロセス

```python
async def stream_with_thinking(prompt: str):
    async for event in agent.stream_async(prompt, include_thinking=True):
        if "thinking" in event:
            yield {
                "type": "thinking",
                "data": event["thinking"]
            }
        
        if "data" in event:
            yield {
                "type": "content",
                "data": event["data"]
            }
```

## 制御イベント

### キャンセル

```typescript
// クライアント側
function cancelCurrentRequest() {
    socket.send(JSON.stringify({
        type: 'control',
        action: 'cancel'
    }));
}

// サーバー側
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    cancel_event = asyncio.Event()
    
    async def handle_messages():
        while True:
            data = await websocket.receive_json()
            
            if data["type"] == "control" and data["action"] == "cancel":
                cancel_event.set()
            elif data["type"] == "chat":
                await process_chat(data, cancel_event)
    
    async def process_chat(data, cancel_event):
        async for event in agent.stream_async(data["content"]):
            if cancel_event.is_set():
                await websocket.send_json({
                    "type": "end",
                    "reason": "cancelled"
                })
                cancel_event.clear()
                return
            
            await websocket.send_json(event)
```

### 一時停止と再開

```python
class StreamController:
    def __init__(self):
        self.paused = asyncio.Event()
        self.paused.set()  # 初期状態は非一時停止
    
    def pause(self):
        self.paused.clear()
    
    def resume(self):
        self.paused.set()
    
    async def wait_if_paused(self):
        await self.paused.wait()

controller = StreamController()

async def controlled_stream(prompt: str):
    async for event in agent.stream_async(prompt):
        await controller.wait_if_paused()
        yield event
```

### ハートビート

```python
import asyncio

async def heartbeat_handler(websocket: WebSocket):
    """ハートビート処理"""
    while True:
        try:
            await asyncio.sleep(30)
            await websocket.send_json({"type": "ping"})
            
            # Pong を待つ
            response = await asyncio.wait_for(
                websocket.receive_json(),
                timeout=10
            )
            
            if response.get("type") != "pong":
                # 予期しないメッセージは処理キューに戻す
                pass
                
        except asyncio.TimeoutError:
            # タイムアウト - 接続を閉じる
            await websocket.close()
            break
```

## カスタムイベント

### 進捗イベント

```python
# 進捗を報告
async def stream_with_progress(prompt: str, websocket: WebSocket):
    total_steps = 5
    
    for step in range(total_steps):
        await websocket.send_json({
            "type": "progress",
            "current": step + 1,
            "total": total_steps,
            "message": f"ステップ {step + 1} を処理中..."
        })
        
        # 処理を実行
        await process_step(step)
    
    await websocket.send_json({
        "type": "progress",
        "current": total_steps,
        "total": total_steps,
        "message": "完了"
    })
```

### 承認リクエスト

```python
async def stream_with_approval(prompt: str, websocket: WebSocket):
    async for event in agent.stream_async(prompt):
        if event.get("requires_approval"):
            # 承認を要求
            await websocket.send_json({
                "type": "approval_request",
                "action": event["action"],
                "details": event["details"],
                "request_id": event["id"]
            })
            
            # 承認を待つ
            response = await websocket.receive_json()
            
            if response.get("type") == "approval_response":
                if response["approved"]:
                    # 処理を続行
                    continue
                else:
                    # キャンセル
                    break
```

## イベント処理パターン

### イベントディスパッチャー

```python
from typing import Callable, Dict

class EventDispatcher:
    def __init__(self):
        self.handlers: Dict[str, list[Callable]] = {}
    
    def on(self, event_type: str, handler: Callable):
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)
    
    async def dispatch(self, event: dict):
        event_type = event.get("type")
        if event_type in self.handlers:
            for handler in self.handlers[event_type]:
                await handler(event)

# 使用例
dispatcher = EventDispatcher()

@dispatcher.on("content")
async def handle_content(event):
    print(event["data"], end="", flush=True)

@dispatcher.on("tool_use")
async def handle_tool_use(event):
    print(f"\n[Tool: {event['tool_name']}]")

@dispatcher.on("error")
async def handle_error(event):
    print(f"\nError: {event['message']}")
```

### イベントフィルター

```python
class EventFilter:
    def __init__(self, include: list[str] = None, exclude: list[str] = None):
        self.include = include
        self.exclude = exclude or []
    
    def should_process(self, event: dict) -> bool:
        event_type = event.get("type")
        
        if self.include and event_type not in self.include:
            return False
        
        if event_type in self.exclude:
            return False
        
        return True

# 使用例
filter = EventFilter(exclude=["thinking", "tool_use"])

async for event in stream:
    if filter.should_process(event):
        await process(event)
```

### イベント変換

```python
class EventTransformer:
    def transform(self, event: dict) -> dict:
        # タイムスタンプを追加
        event["received_at"] = datetime.now().isoformat()
        
        # 型を正規化
        if event["type"] == "text":
            event["type"] = "content"
        
        return event

transformer = EventTransformer()

async for event in stream:
    transformed = transformer.transform(event)
    await process(transformed)
```
