# ベストプラクティス

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/best-practices/

## 目次

- [概要](#概要)
- [接続管理](#接続管理)
- [メッセージ設計](#メッセージ設計)
- [パフォーマンス最適化](#パフォーマンス最適化)
- [エラー処理](#エラー処理)
- [テスト](#テスト)
- [デプロイメント](#デプロイメント)

---

## 概要

双方向ストリーミングの実装において、信頼性、パフォーマンス、保守性を確保するためのベストプラクティスをまとめる。

## 接続管理

### 接続のライフサイクル管理

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator

@asynccontextmanager
async def managed_connection(websocket: WebSocket, session_id: str) -> AsyncGenerator:
    """接続のライフサイクルを管理"""
    try:
        # 接続時の処理
        await websocket.accept()
        await register_connection(session_id, websocket)
        
        yield websocket
        
    except WebSocketDisconnect:
        # 正常な切断
        pass
    except Exception as e:
        # エラー時の処理
        await log_error(session_id, e)
        raise
    finally:
        # クリーンアップ
        await unregister_connection(session_id)
        await cleanup_session(session_id)

@app.websocket("/ws/{session_id}")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    async with managed_connection(websocket, session_id) as ws:
        await handle_messages(ws)
```

### ハートビートの実装

```python
import asyncio

class HeartbeatManager:
    def __init__(self, interval: int = 30, timeout: int = 10):
        self.interval = interval
        self.timeout = timeout
        self.last_pong = {}
    
    async def start(self, websocket: WebSocket, session_id: str):
        """ハートビートを開始"""
        while True:
            await asyncio.sleep(self.interval)
            
            try:
                await websocket.send_json({"type": "ping"})
                
                # Pong を待つ
                await asyncio.wait_for(
                    self._wait_for_pong(session_id),
                    timeout=self.timeout
                )
            except asyncio.TimeoutError:
                # タイムアウト - 接続を閉じる
                await websocket.close(code=1002)
                break
    
    async def _wait_for_pong(self, session_id: str):
        while session_id not in self.last_pong:
            await asyncio.sleep(0.1)
        del self.last_pong[session_id]
    
    def record_pong(self, session_id: str):
        self.last_pong[session_id] = True
```

### グレースフルシャットダウン

```python
import signal
import asyncio

class GracefulShutdown:
    def __init__(self):
        self.shutdown_event = asyncio.Event()
        self.active_connections = set()
    
    def register_connection(self, websocket: WebSocket):
        self.active_connections.add(websocket)
    
    def unregister_connection(self, websocket: WebSocket):
        self.active_connections.discard(websocket)
    
    async def shutdown(self):
        """グレースフルシャットダウン"""
        self.shutdown_event.set()
        
        # すべての接続に終了を通知
        for ws in self.active_connections:
            try:
                await ws.send_json({
                    "type": "server_shutdown",
                    "message": "Server is shutting down"
                })
                await ws.close(code=1001)
            except Exception:
                pass
        
        # 接続が閉じるのを待つ
        while self.active_connections:
            await asyncio.sleep(0.1)

shutdown_manager = GracefulShutdown()

@app.on_event("shutdown")
async def app_shutdown():
    await shutdown_manager.shutdown()
```

## メッセージ設計

### 一貫したメッセージ形式

```python
from dataclasses import dataclass, asdict
from typing import Any, Optional
from datetime import datetime
import uuid

@dataclass
class Message:
    type: str
    data: Any
    id: str = None
    timestamp: str = None
    
    def __post_init__(self):
        if self.id is None:
            self.id = str(uuid.uuid4())
        if self.timestamp is None:
            self.timestamp = datetime.utcnow().isoformat()
    
    def to_dict(self) -> dict:
        return asdict(self)

# メッセージタイプを定数で定義
class MessageType:
    CONTENT = "content"
    TOOL_USE = "tool_use"
    TOOL_RESULT = "tool_result"
    ERROR = "error"
    END = "end"
    PING = "ping"
    PONG = "pong"

# 使用例
message = Message(
    type=MessageType.CONTENT,
    data={"text": "こんにちは"}
)
await websocket.send_json(message.to_dict())
```

### メッセージのバッチ処理

```python
import asyncio
from typing import List

class MessageBatcher:
    def __init__(self, max_size: int = 10, max_wait: float = 0.1):
        self.max_size = max_size
        self.max_wait = max_wait
        self.batch: List[dict] = []
        self.lock = asyncio.Lock()
    
    async def add(self, message: dict):
        async with self.lock:
            self.batch.append(message)
            
            if len(self.batch) >= self.max_size:
                return await self._flush()
    
    async def _flush(self) -> List[dict]:
        batch = self.batch
        self.batch = []
        return batch
    
    async def flush_periodically(self, websocket: WebSocket):
        while True:
            await asyncio.sleep(self.max_wait)
            
            async with self.lock:
                if self.batch:
                    batch = await self._flush()
                    await websocket.send_json({
                        "type": "batch",
                        "messages": batch
                    })
```

## パフォーマンス最適化

### 接続プーリング

```python
from typing import Dict, Optional
import asyncio

class ConnectionPool:
    def __init__(self, max_connections_per_user: int = 5):
        self.max_per_user = max_connections_per_user
        self.connections: Dict[str, list[WebSocket]] = {}
        self.lock = asyncio.Lock()
    
    async def add(self, user_id: str, websocket: WebSocket) -> bool:
        async with self.lock:
            if user_id not in self.connections:
                self.connections[user_id] = []
            
            if len(self.connections[user_id]) >= self.max_per_user:
                return False
            
            self.connections[user_id].append(websocket)
            return True
    
    async def remove(self, user_id: str, websocket: WebSocket):
        async with self.lock:
            if user_id in self.connections:
                self.connections[user_id] = [
                    ws for ws in self.connections[user_id]
                    if ws != websocket
                ]
    
    async def broadcast_to_user(self, user_id: str, message: dict):
        async with self.lock:
            connections = self.connections.get(user_id, [])
        
        await asyncio.gather(*[
            ws.send_json(message)
            for ws in connections
        ])
```

### メモリ管理

```python
import sys
from typing import Deque
from collections import deque

class BoundedMessageHistory:
    """メモリ使用量を制限したメッセージ履歴"""
    
    def __init__(self, max_messages: int = 100, max_bytes: int = 1_000_000):
        self.max_messages = max_messages
        self.max_bytes = max_bytes
        self.messages: Deque[dict] = deque(maxlen=max_messages)
        self.total_bytes = 0
    
    def add(self, message: dict):
        msg_size = sys.getsizeof(str(message))
        
        # 古いメッセージを削除してスペースを確保
        while self.total_bytes + msg_size > self.max_bytes and self.messages:
            old_msg = self.messages.popleft()
            self.total_bytes -= sys.getsizeof(str(old_msg))
        
        self.messages.append(message)
        self.total_bytes += msg_size
    
    def get_all(self) -> list[dict]:
        return list(self.messages)
```

### 非同期処理の最適化

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# CPU バウンドタスク用のスレッドプール
thread_pool = ThreadPoolExecutor(max_workers=4)

async def process_cpu_intensive(data: dict) -> dict:
    """CPU 集約型タスクをスレッドプールで実行"""
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(
        thread_pool,
        _sync_process,
        data
    )

def _sync_process(data: dict) -> dict:
    # CPU 集約型処理
    return {"result": "processed"}

# 並列処理
async def process_multiple(items: list[dict]) -> list[dict]:
    """複数のアイテムを並列処理"""
    tasks = [process_item(item) for item in items]
    return await asyncio.gather(*tasks)
```

## エラー処理

### 構造化エラーハンドリング

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional

class ErrorSeverity(Enum):
    WARNING = "warning"
    ERROR = "error"
    CRITICAL = "critical"

@dataclass
class StreamError:
    code: str
    message: str
    severity: ErrorSeverity
    recoverable: bool
    details: Optional[dict] = None
    
    def to_dict(self) -> dict:
        return {
            "type": "error",
            "code": self.code,
            "message": self.message,
            "severity": self.severity.value,
            "recoverable": self.recoverable,
            "details": self.details
        }

# エラーファクトリ
class Errors:
    @staticmethod
    def rate_limited() -> StreamError:
        return StreamError(
            code="RATE_LIMITED",
            message="レート制限に達しました",
            severity=ErrorSeverity.WARNING,
            recoverable=True
        )
    
    @staticmethod
    def context_exhausted() -> StreamError:
        return StreamError(
            code="CONTEXT_EXHAUSTED",
            message="コンテキストウィンドウを超えました",
            severity=ErrorSeverity.ERROR,
            recoverable=True
        )
    
    @staticmethod
    def internal_error(details: dict = None) -> StreamError:
        return StreamError(
            code="INTERNAL_ERROR",
            message="内部エラーが発生しました",
            severity=ErrorSeverity.CRITICAL,
            recoverable=False,
            details=details
        )
```

## テスト

### WebSocket テスト

```python
import pytest
from fastapi.testclient import TestClient
from fastapi.websockets import WebSocket

@pytest.fixture
def client():
    return TestClient(app)

def test_websocket_connection(client):
    with client.websocket_connect("/ws/test-session") as websocket:
        # メッセージを送信
        websocket.send_json({
            "type": "chat",
            "content": "テスト"
        })
        
        # 応答を受信
        responses = []
        while True:
            data = websocket.receive_json()
            responses.append(data)
            
            if data["type"] == "end":
                break
        
        # 検証
        assert len(responses) > 0
        assert any(r["type"] == "content" for r in responses)

def test_error_handling(client):
    with client.websocket_connect("/ws/test-session") as websocket:
        # 無効なメッセージを送信
        websocket.send_json({
            "invalid": "message"
        })
        
        # エラー応答を確認
        data = websocket.receive_json()
        assert data["type"] == "error"
```

### 負荷テスト

```python
import asyncio
import websockets
import time

async def load_test(url: str, num_connections: int, messages_per_connection: int):
    """負荷テストを実行"""
    
    async def client_session(client_id: int):
        start = time.time()
        messages_received = 0
        
        async with websockets.connect(url) as ws:
            for i in range(messages_per_connection):
                await ws.send(json.dumps({
                    "type": "chat",
                    "content": f"Test message {i}"
                }))
                
                async for message in ws:
                    data = json.loads(message)
                    messages_received += 1
                    
                    if data["type"] == "end":
                        break
        
        elapsed = time.time() - start
        return {
            "client_id": client_id,
            "messages_received": messages_received,
            "elapsed": elapsed
        }
    
    # 並列でクライアントを実行
    tasks = [client_session(i) for i in range(num_connections)]
    results = await asyncio.gather(*tasks)
    
    # 統計を計算
    total_messages = sum(r["messages_received"] for r in results)
    total_time = sum(r["elapsed"] for r in results)
    
    print(f"Total connections: {num_connections}")
    print(f"Total messages: {total_messages}")
    print(f"Average time per connection: {total_time / num_connections:.2f}s")
    print(f"Messages per second: {total_messages / max(r['elapsed'] for r in results):.2f}")

# 実行
asyncio.run(load_test("ws://localhost:8000/ws", 100, 10))
```

## デプロイメント

### 本番環境チェックリスト

```yaml
# deployment-checklist.yaml
deployment:
  security:
    - TLS/SSL 証明書の設定
    - 認証の有効化
    - レート制限の設定
    - 入力検証の実装
  
  reliability:
    - ヘルスチェックの設定
    - 自動再起動の設定
    - ログの設定
    - モニタリングの設定
  
  performance:
    - 接続数の制限
    - タイムアウトの設定
    - キープアライブの設定
    - ロードバランサーの設定
  
  scaling:
    - 水平スケーリングの準備
    - セッション共有（Redis）
    - Pub/Sub の設定
```

### Docker 設定

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 非 root ユーザーで実行
RUN useradd -m appuser
USER appuser

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Kubernetes 設定

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-streaming
spec:
  replicas: 3
  selector:
    matchLabels:
      app: agent-streaming
  template:
    spec:
      containers:
      - name: agent
        image: agent-streaming:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 20
```
