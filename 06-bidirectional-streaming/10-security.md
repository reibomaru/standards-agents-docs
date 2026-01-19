# セキュリティ

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/security/

## 目次

- [概要](#概要)
- [認証](#認証)
- [認可](#認可)
- [通信の暗号化](#通信の暗号化)
- [入力検証](#入力検証)
- [レート制限](#レート制限)
- [セキュリティベストプラクティス](#セキュリティベストプラクティス)

---

## 概要

双方向ストリーミングでは、持続的な接続を維持するため、適切なセキュリティ対策が重要。認証、認可、暗号化、入力検証などを適切に実装する必要がある。

## 認証

### JWT トークン認証

```python
from fastapi import FastAPI, WebSocket, HTTPException
from fastapi.security import HTTPBearer
import jwt
from datetime import datetime, timedelta

app = FastAPI()

SECRET_KEY = "your-secret-key-here"
ALGORITHM = "HS256"

def create_access_token(user_id: str, expires_delta: timedelta = None) -> str:
    """アクセストークンを生成"""
    expire = datetime.utcnow() + (expires_delta or timedelta(hours=24))
    payload = {
        "sub": user_id,
        "exp": expire,
        "iat": datetime.utcnow()
    }
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)

def verify_token(token: str) -> dict:
    """トークンを検証"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # クエリパラメータからトークンを取得
    token = websocket.query_params.get("token")
    
    if not token:
        await websocket.close(code=4001, reason="Missing token")
        return
    
    try:
        payload = verify_token(token)
        user_id = payload["sub"]
    except HTTPException as e:
        await websocket.close(code=4001, reason=e.detail)
        return
    
    await websocket.accept()
    
    # 認証成功、処理を続行
    await handle_authenticated_connection(websocket, user_id)
```

### API キー認証

```python
from fastapi import WebSocket
from hashlib import sha256

VALID_API_KEYS = {
    sha256(b"api-key-1").hexdigest(): "user1",
    sha256(b"api-key-2").hexdigest(): "user2",
}

def verify_api_key(api_key: str) -> str:
    """API キーを検証してユーザーIDを返す"""
    key_hash = sha256(api_key.encode()).hexdigest()
    
    if key_hash not in VALID_API_KEYS:
        raise ValueError("Invalid API key")
    
    return VALID_API_KEYS[key_hash]

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    api_key = websocket.headers.get("X-API-Key")
    
    if not api_key:
        await websocket.close(code=4001, reason="Missing API key")
        return
    
    try:
        user_id = verify_api_key(api_key)
    except ValueError:
        await websocket.close(code=4001, reason="Invalid API key")
        return
    
    await websocket.accept()
    await handle_authenticated_connection(websocket, user_id)
```

### OAuth 2.0

```python
import httpx
from fastapi import WebSocket

OAUTH_INTROSPECTION_URL = "https://auth.example.com/oauth/introspect"
CLIENT_ID = "your-client-id"
CLIENT_SECRET = "your-client-secret"

async def verify_oauth_token(token: str) -> dict:
    """OAuth トークンを検証"""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            OAUTH_INTROSPECTION_URL,
            data={
                "token": token,
                "client_id": CLIENT_ID,
                "client_secret": CLIENT_SECRET
            }
        )
        
        if response.status_code != 200:
            raise ValueError("Token introspection failed")
        
        data = response.json()
        
        if not data.get("active"):
            raise ValueError("Token is not active")
        
        return data

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    auth_header = websocket.headers.get("Authorization")
    
    if not auth_header or not auth_header.startswith("Bearer "):
        await websocket.close(code=4001, reason="Missing or invalid authorization")
        return
    
    token = auth_header[7:]  # "Bearer " を除去
    
    try:
        token_data = await verify_oauth_token(token)
    except ValueError as e:
        await websocket.close(code=4001, reason=str(e))
        return
    
    await websocket.accept()
    await handle_authenticated_connection(websocket, token_data["sub"])
```

## 認可

### ロールベースアクセス制御 (RBAC)

```python
from enum import Enum
from typing import Set

class Role(Enum):
    USER = "user"
    ADMIN = "admin"
    PREMIUM = "premium"

class Permission(Enum):
    CHAT = "chat"
    USE_TOOLS = "use_tools"
    USE_PREMIUM_MODELS = "use_premium_models"
    ADMIN_ACTIONS = "admin_actions"

ROLE_PERMISSIONS: dict[Role, Set[Permission]] = {
    Role.USER: {Permission.CHAT},
    Role.PREMIUM: {Permission.CHAT, Permission.USE_TOOLS, Permission.USE_PREMIUM_MODELS},
    Role.ADMIN: {Permission.CHAT, Permission.USE_TOOLS, Permission.USE_PREMIUM_MODELS, Permission.ADMIN_ACTIONS},
}

class AuthorizationManager:
    def __init__(self, user_role: Role):
        self.role = user_role
        self.permissions = ROLE_PERMISSIONS.get(user_role, set())
    
    def has_permission(self, permission: Permission) -> bool:
        return permission in self.permissions
    
    def require_permission(self, permission: Permission):
        if not self.has_permission(permission):
            raise PermissionError(f"Missing permission: {permission.value}")

# 使用例
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    user = await authenticate(websocket)
    auth_manager = AuthorizationManager(user.role)
    
    await websocket.accept()
    
    while True:
        data = await websocket.receive_json()
        
        if data["type"] == "chat":
            auth_manager.require_permission(Permission.CHAT)
            await process_chat(data)
        
        elif data["type"] == "use_tool":
            auth_manager.require_permission(Permission.USE_TOOLS)
            await process_tool(data)
```

## 通信の暗号化

### TLS/SSL 設定

```python
import ssl
from fastapi import FastAPI
import uvicorn

app = FastAPI()

# SSL コンテキストを作成
ssl_context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ssl_context.load_cert_chain(
    certfile="path/to/cert.pem",
    keyfile="path/to/key.pem"
)

# 最小 TLS バージョンを設定
ssl_context.minimum_version = ssl.TLSVersion.TLSv1_2

# 弱い暗号を無効化
ssl_context.set_ciphers("HIGH:!aNULL:!MD5")

if __name__ == "__main__":
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=443,
        ssl_keyfile="path/to/key.pem",
        ssl_certfile="path/to/cert.pem"
    )
```

### クライアント側

```javascript
// wss:// を使用して暗号化された接続
const socket = new WebSocket('wss://example.com/ws?token=xxx');
```

## 入力検証

### メッセージ検証

```python
from pydantic import BaseModel, validator, constr
from typing import Optional, Literal

class ChatMessage(BaseModel):
    type: Literal["chat"]
    content: constr(min_length=1, max_length=10000)  # 最大10000文字
    metadata: Optional[dict] = None
    
    @validator('content')
    def sanitize_content(cls, v):
        # 危険な文字を除去
        v = v.strip()
        # 追加のサニタイゼーション
        return v

class ControlMessage(BaseModel):
    type: Literal["control"]
    action: Literal["cancel", "pause", "resume"]

def validate_message(data: dict) -> BaseModel:
    """メッセージを検証"""
    msg_type = data.get("type")
    
    if msg_type == "chat":
        return ChatMessage(**data)
    elif msg_type == "control":
        return ControlMessage(**data)
    else:
        raise ValueError(f"Unknown message type: {msg_type}")

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        raw_data = await websocket.receive_json()
        
        try:
            message = validate_message(raw_data)
        except Exception as e:
            await websocket.send_json({
                "type": "error",
                "message": f"Invalid message: {str(e)}"
            })
            continue
        
        # 検証済みメッセージを処理
        await process_message(message)
```

### コンテンツフィルタリング

```python
import re

class ContentFilter:
    def __init__(self):
        # 禁止パターン
        self.banned_patterns = [
            re.compile(r'<script.*?>.*?</script>', re.IGNORECASE | re.DOTALL),
            re.compile(r'javascript:', re.IGNORECASE),
        ]
        
        # 禁止ワード
        self.banned_words = ['spam', 'abuse']
    
    def filter(self, content: str) -> str:
        """コンテンツをフィルタリング"""
        # パターンを除去
        for pattern in self.banned_patterns:
            content = pattern.sub('', content)
        
        # 禁止ワードをチェック
        content_lower = content.lower()
        for word in self.banned_words:
            if word in content_lower:
                raise ValueError(f"Banned content detected")
        
        return content

content_filter = ContentFilter()
```

## レート制限

### トークンバケット

```python
import time
from dataclasses import dataclass

@dataclass
class TokenBucket:
    capacity: int
    refill_rate: float  # トークン/秒
    tokens: float
    last_refill: float

class RateLimiter:
    def __init__(self):
        self.buckets: dict[str, TokenBucket] = {}
    
    def get_bucket(self, key: str, capacity: int = 10, refill_rate: float = 1.0) -> TokenBucket:
        if key not in self.buckets:
            self.buckets[key] = TokenBucket(
                capacity=capacity,
                refill_rate=refill_rate,
                tokens=capacity,
                last_refill=time.time()
            )
        return self.buckets[key]
    
    def consume(self, key: str, tokens: int = 1) -> bool:
        """トークンを消費。成功したら True を返す"""
        bucket = self.get_bucket(key)
        
        # トークンを補充
        now = time.time()
        elapsed = now - bucket.last_refill
        bucket.tokens = min(
            bucket.capacity,
            bucket.tokens + elapsed * bucket.refill_rate
        )
        bucket.last_refill = now
        
        # トークンを消費
        if bucket.tokens >= tokens:
            bucket.tokens -= tokens
            return True
        
        return False

rate_limiter = RateLimiter()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await websocket.accept()
    
    while True:
        data = await websocket.receive_json()
        
        if not rate_limiter.consume(user_id):
            await websocket.send_json({
                "type": "error",
                "code": "rate_limited",
                "message": "Rate limit exceeded. Please wait."
            })
            continue
        
        await process_message(data)
```

## セキュリティベストプラクティス

### チェックリスト

```python
"""
セキュリティチェックリスト:

1. 認証
   - [ ] すべての接続で認証を要求
   - [ ] トークンの有効期限を設定
   - [ ] トークンの自動更新メカニズム

2. 認可
   - [ ] ロールベースアクセス制御を実装
   - [ ] 最小権限の原則を適用
   - [ ] 権限チェックをすべての操作で実行

3. 暗号化
   - [ ] TLS 1.2 以上を使用
   - [ ] wss:// を強制
   - [ ] 証明書の定期更新

4. 入力検証
   - [ ] すべての入力を検証
   - [ ] サイズ制限を設定
   - [ ] 危険なコンテンツをフィルタリング

5. レート制限
   - [ ] ユーザーごとのレート制限
   - [ ] IP ベースのレート制限
   - [ ] グローバルレート制限

6. ログとモニタリング
   - [ ] 認証失敗をログ
   - [ ] 異常なアクティビティを検出
   - [ ] アラートを設定
"""
```
