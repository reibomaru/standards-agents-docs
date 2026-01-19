# エラーハンドリング

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/error-handling/

## 目次

- [概要](#概要)
- [エラータイプ](#エラータイプ)
- [サーバー側エラーハンドリング](#サーバー側エラーハンドリング)
- [クライアント側エラーハンドリング](#クライアント側エラーハンドリング)
- [再試行戦略](#再試行戦略)
- [グレースフルデグラデーション](#グレースフルデグラデーション)

---

## 概要

双方向ストリーミングでは、接続エラー、タイムアウト、Agent エラーなど、様々なエラーが発生する可能性がある。適切なエラーハンドリングは、堅牢なアプリケーションに不可欠。

## エラータイプ

### 接続エラー

```python
class ConnectionError(Exception):
    """接続関連のエラー"""
    pass

class ConnectionRefusedError(ConnectionError):
    """接続拒否"""
    pass

class ConnectionTimeoutError(ConnectionError):
    """接続タイムアウト"""
    pass

class ConnectionClosedError(ConnectionError):
    """予期しない接続切断"""
    pass
```

### プロトコルエラー

```python
class ProtocolError(Exception):
    """プロトコル関連のエラー"""
    pass

class InvalidMessageError(ProtocolError):
    """無効なメッセージ形式"""
    pass

class MessageTooLargeError(ProtocolError):
    """メッセージサイズ超過"""
    pass
```

### Agent エラー

```python
class AgentError(Exception):
    """Agent 関連のエラー"""
    pass

class ContextWindowExhaustedError(AgentError):
    """コンテキストウィンドウ超過"""
    pass

class ToolExecutionError(AgentError):
    """ツール実行エラー"""
    pass

class RateLimitError(AgentError):
    """レート制限"""
    pass
```

### エラーコード

```python
from enum import Enum

class ErrorCode(Enum):
    # 接続エラー (1xxx)
    CONNECTION_REFUSED = 1001
    CONNECTION_TIMEOUT = 1002
    CONNECTION_CLOSED = 1003
    
    # 認証エラー (2xxx)
    UNAUTHORIZED = 2001
    TOKEN_EXPIRED = 2002
    
    # プロトコルエラー (3xxx)
    INVALID_MESSAGE = 3001
    MESSAGE_TOO_LARGE = 3002
    
    # Agent エラー (4xxx)
    AGENT_ERROR = 4001
    CONTEXT_EXHAUSTED = 4002
    TOOL_ERROR = 4003
    RATE_LIMITED = 4004
    
    # サーバーエラー (5xxx)
    INTERNAL_ERROR = 5001
    SERVICE_UNAVAILABLE = 5002
```

## サーバー側エラーハンドリング

### FastAPI での実装

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.websockets import WebSocketState
import traceback

app = FastAPI()

async def send_error(
    websocket: WebSocket,
    code: ErrorCode,
    message: str,
    recoverable: bool = True
):
    """エラーメッセージを送信"""
    if websocket.client_state == WebSocketState.CONNECTED:
        await websocket.send_json({
            "type": "error",
            "code": code.value,
            "message": message,
            "recoverable": recoverable
        })

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            try:
                data = await websocket.receive_json()
                
                # メッセージを検証
                if "type" not in data:
                    await send_error(
                        websocket,
                        ErrorCode.INVALID_MESSAGE,
                        "Missing 'type' field"
                    )
                    continue
                
                # 処理
                if data["type"] == "chat":
                    await process_chat(websocket, data)
                    
            except json.JSONDecodeError:
                await send_error(
                    websocket,
                    ErrorCode.INVALID_MESSAGE,
                    "Invalid JSON"
                )
            
            except AgentError as e:
                await send_error(
                    websocket,
                    ErrorCode.AGENT_ERROR,
                    str(e),
                    recoverable=True
                )
            
            except Exception as e:
                # 予期しないエラー
                traceback.print_exc()
                await send_error(
                    websocket,
                    ErrorCode.INTERNAL_ERROR,
                    "Internal server error",
                    recoverable=False
                )
                
    except WebSocketDisconnect:
        # 正常な切断
        pass
    
    except Exception as e:
        # 致命的なエラー
        traceback.print_exc()
```

### エラーリカバリー

```python
async def process_with_recovery(websocket: WebSocket, data: dict):
    """エラーリカバリー付き処理"""
    max_retries = 3
    retry_count = 0
    
    while retry_count < max_retries:
        try:
            async for event in agent.stream_async(data["content"]):
                await websocket.send_json(event)
            
            return  # 成功
            
        except RateLimitError:
            retry_count += 1
            wait_time = 2 ** retry_count
            
            await websocket.send_json({
                "type": "retry",
                "message": f"Rate limited. Retrying in {wait_time}s...",
                "attempt": retry_count,
                "max_attempts": max_retries
            })
            
            await asyncio.sleep(wait_time)
        
        except ContextWindowExhaustedError:
            # 会話履歴を削減して再試行
            agent.trim_conversation_history()
            retry_count += 1
    
    # すべてのリトライが失敗
    await send_error(
        websocket,
        ErrorCode.AGENT_ERROR,
        "Failed after maximum retries",
        recoverable=False
    )
```

## クライアント側エラーハンドリング

### JavaScript 実装

```javascript
class RobustAgentClient {
    constructor(url, options = {}) {
        this.url = url;
        this.maxRetries = options.maxRetries || 3;
        this.retryDelay = options.retryDelay || 1000;
        this.socket = null;
        this.errorHandlers = [];
    }
    
    onError(handler) {
        this.errorHandlers.push(handler);
        return () => {
            this.errorHandlers = this.errorHandlers.filter(h => h !== handler);
        };
    }
    
    handleError(error) {
        this.errorHandlers.forEach(handler => handler(error));
    }
    
    async connect() {
        let retries = 0;
        
        while (retries < this.maxRetries) {
            try {
                await this._connect();
                return;
            } catch (error) {
                retries++;
                
                if (retries >= this.maxRetries) {
                    this.handleError({
                        type: 'connection',
                        message: 'Failed to connect after maximum retries',
                        recoverable: false
                    });
                    throw error;
                }
                
                const delay = this.retryDelay * Math.pow(2, retries - 1);
                await this._sleep(delay);
            }
        }
    }
    
    _connect() {
        return new Promise((resolve, reject) => {
            this.socket = new WebSocket(this.url);
            
            this.socket.onopen = () => resolve();
            this.socket.onerror = (e) => reject(e);
            
            this.socket.onmessage = (event) => {
                const data = JSON.parse(event.data);
                
                if (data.type === 'error') {
                    this.handleError(data);
                    
                    if (!data.recoverable) {
                        this.socket.close();
                    }
                }
            };
            
            this.socket.onclose = (event) => {
                if (!event.wasClean) {
                    this.handleError({
                        type: 'connection',
                        message: 'Connection closed unexpectedly',
                        code: event.code,
                        recoverable: true
                    });
                }
            };
        });
    }
    
    _sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// 使用例
const client = new RobustAgentClient('ws://localhost:8000/ws');

client.onError((error) => {
    console.error('Error:', error.message);
    
    if (error.recoverable) {
        // リカバリー可能なエラー
        showRetryPrompt();
    } else {
        // 致命的なエラー
        showFatalError(error.message);
    }
});

try {
    await client.connect();
} catch (e) {
    console.error('Failed to establish connection');
}
```

### TypeScript 実装

```typescript
interface StreamError {
    type: string;
    code: number;
    message: string;
    recoverable: boolean;
}

class ErrorHandler {
    private handlers: Map<number, (error: StreamError) => void> = new Map();
    private defaultHandler: (error: StreamError) => void;
    
    constructor() {
        this.defaultHandler = (error) => {
            console.error(`Unhandled error: ${error.message}`);
        };
    }
    
    register(code: number, handler: (error: StreamError) => void): void {
        this.handlers.set(code, handler);
    }
    
    setDefault(handler: (error: StreamError) => void): void {
        this.defaultHandler = handler;
    }
    
    handle(error: StreamError): void {
        const handler = this.handlers.get(error.code) || this.defaultHandler;
        handler(error);
    }
}

// 使用例
const errorHandler = new ErrorHandler();

// 特定のエラーコードにハンドラを登録
errorHandler.register(4004, (error) => {
    // レート制限
    showRateLimitDialog(error);
});

errorHandler.register(4002, (error) => {
    // コンテキスト超過
    promptToStartNewConversation();
});

errorHandler.setDefault((error) => {
    showErrorNotification(error.message);
});
```

## 再試行戦略

### 指数バックオフ

```python
import asyncio
from typing import TypeVar, Callable

T = TypeVar('T')

async def retry_with_backoff(
    func: Callable[[], T],
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential: float = 2.0,
    retryable_exceptions: tuple = (Exception,)
) -> T:
    """指数バックオフでリトライ"""
    last_exception = None
    
    for attempt in range(max_retries):
        try:
            return await func()
        except retryable_exceptions as e:
            last_exception = e
            
            if attempt == max_retries - 1:
                raise
            
            delay = min(base_delay * (exponential ** attempt), max_delay)
            await asyncio.sleep(delay)
    
    raise last_exception

# 使用例
async def send_message(content: str):
    return await retry_with_backoff(
        lambda: agent.stream_async(content),
        max_retries=3,
        retryable_exceptions=(RateLimitError, ConnectionError)
    )
```

### サーキットブレーカー

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"    # 正常
    OPEN = "open"        # エラー状態
    HALF_OPEN = "half_open"  # 回復試行中

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 30.0
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failures = 0
        self.state = CircuitState.CLOSED
        self.last_failure_time = 0
    
    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        
        return True  # HALF_OPEN
    
    def record_success(self):
        self.failures = 0
        self.state = CircuitState.CLOSED
    
    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        
        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# 使用例
circuit_breaker = CircuitBreaker()

async def protected_call():
    if not circuit_breaker.can_execute():
        raise Exception("Circuit breaker is open")
    
    try:
        result = await agent.stream_async(prompt)
        circuit_breaker.record_success()
        return result
    except Exception as e:
        circuit_breaker.record_failure()
        raise
```

## グレースフルデグラデーション

```python
async def degraded_response(websocket: WebSocket, original_prompt: str):
    """グレースフルデグラデーション"""
    
    # 1. キャッシュからの応答を試す
    cached = await get_cached_response(original_prompt)
    if cached:
        await websocket.send_json({
            "type": "content",
            "data": cached,
            "source": "cache"
        })
        return
    
    # 2. 簡易モデルにフォールバック
    try:
        async for event in simple_agent.stream_async(original_prompt):
            await websocket.send_json(event)
        return
    except Exception:
        pass
    
    # 3. 静的応答
    await websocket.send_json({
        "type": "content",
        "data": "申し訳ありません。現在サービスが利用できません。しばらくしてからもう一度お試しください。",
        "source": "fallback"
    })
```
