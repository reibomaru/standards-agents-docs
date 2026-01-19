# Bidirectional Streaming クイックスタート

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/quick-start/

## 目次

- [前提条件](#前提条件)
- [基本的な実装](#基本的な実装)
- [サーバーのセットアップ](#サーバーのセットアップ)
- [クライアントのセットアップ](#クライアントのセットアップ)
- [実行例](#実行例)

---

## 前提条件

```bash
pip install strands[streaming]
```

必要なパッケージ：
- `strands` (コアライブラリ)
- `websockets` (WebSocket サポート)
- `aiohttp` (非同期 HTTP)

## 基本的な実装

### 最小構成

```python
import asyncio
from strands import Agent
from strands.streaming import BidirectionalStream

async def main():
    # Agent を作成
    agent = Agent(
        system_prompt="あなたは親切なアシスタントです。"
    )
    
    # 双方向ストリームを作成
    stream = BidirectionalStream(agent)
    
    # ストリームを開始
    async with stream:
        # メッセージを送信
        await stream.send("こんにちは！")
        
        # 応答を受信
        async for event in stream:
            if event.type == "content":
                print(event.data, end="", flush=True)
            elif event.type == "end":
                break
        
        print()  # 改行

asyncio.run(main())
```

## サーバーのセットアップ

### FastAPI を使用したサーバー

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from strands import Agent
from strands.streaming import WebSocketHandler

app = FastAPI()

# Agent インスタンス
agent = Agent(
    system_prompt="あなたは親切なアシスタントです。"
)

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    # WebSocket ハンドラーを作成
    handler = WebSocketHandler(agent, websocket)
    
    try:
        await handler.run()
    except WebSocketDisconnect:
        print("クライアントが切断しました")
```

### 起動

```bash
uvicorn server:app --host 0.0.0.0 --port 8000
```

## クライアントのセットアップ

### 基本的なクライアント

```python
import asyncio
import websockets

async def client():
    uri = "ws://localhost:8000/ws"
    
    async with websockets.connect(uri) as websocket:
        # メッセージを送信
        await websocket.send("今日の天気は？")
        
        # 応答を受信
        while True:
            try:
                message = await asyncio.wait_for(
                    websocket.recv(),
                    timeout=30.0
                )
                print(message, end="", flush=True)
            except asyncio.TimeoutError:
                break
        
        print()

asyncio.run(client())
```

### インタラクティブクライアント

```python
import asyncio
import websockets
import sys

async def interactive_client():
    uri = "ws://localhost:8000/ws"
    
    async with websockets.connect(uri) as websocket:
        # 受信タスク
        async def receiver():
            while True:
                try:
                    message = await websocket.recv()
                    print(f"\nAgent: {message}")
                    print("You: ", end="", flush=True)
                except websockets.exceptions.ConnectionClosed:
                    break
        
        # 受信タスクを開始
        recv_task = asyncio.create_task(receiver())
        
        # 送信ループ
        print("チャットを開始します。'quit' で終了。")
        print("You: ", end="", flush=True)
        
        while True:
            try:
                # 標準入力から読み取り
                line = await asyncio.get_event_loop().run_in_executor(
                    None, sys.stdin.readline
                )
                line = line.strip()
                
                if line.lower() == 'quit':
                    break
                
                if line:
                    await websocket.send(line)
                    
            except KeyboardInterrupt:
                break
        
        recv_task.cancel()

asyncio.run(interactive_client())
```

## 実行例

### ターミナル 1（サーバー）

```bash
$ uvicorn server:app --host 0.0.0.0 --port 8000
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

### ターミナル 2（クライアント）

```bash
$ python client.py
チャットを開始します。'quit' で終了。
You: こんにちは

Agent: こんにちは！何かお手伝いできることはありますか？
You: Pythonについて教えて

Agent: Pythonは、読みやすく書きやすいことで知られるプログラミング言語です。
主な特徴は：
1. シンプルな構文
2. 豊富なライブラリ
3. 多目的（Web、AI、データ分析など）
詳しく知りたい分野はありますか？
You: quit
```

### 完全なサンプルコード

**server.py**:

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from strands import Agent
import json

app = FastAPI()

agent = Agent(
    system_prompt="""あなたは親切なアシスタントです。
    質問に丁寧に答えてください。"""
)

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    
    try:
        while True:
            # クライアントからメッセージを受信
            data = await websocket.recv_text()
            message = json.loads(data)
            
            # Agent を呼び出し
            async for event in agent.stream_async(message["content"]):
                if "data" in event:
                    await websocket.send_text(json.dumps({
                        "type": "content",
                        "data": event["data"]
                    }))
            
            # 終了を通知
            await websocket.send_text(json.dumps({
                "type": "end"
            }))
            
    except WebSocketDisconnect:
        print("クライアントが切断しました")
```

**client.py**:

```python
import asyncio
import websockets
import json

async def main():
    uri = "ws://localhost:8000/ws"
    
    async with websockets.connect(uri) as websocket:
        while True:
            # ユーザー入力
            user_input = input("You: ")
            
            if user_input.lower() == 'quit':
                break
            
            # メッセージを送信
            await websocket.send(json.dumps({
                "content": user_input
            }))
            
            # 応答を受信
            print("Agent: ", end="", flush=True)
            while True:
                response = await websocket.recv()
                data = json.loads(response)
                
                if data["type"] == "content":
                    print(data["data"], end="", flush=True)
                elif data["type"] == "end":
                    print()
                    break

asyncio.run(main())
```
