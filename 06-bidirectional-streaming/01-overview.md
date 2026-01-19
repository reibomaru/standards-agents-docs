# Bidirectional Streaming 概要

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/bidirectional-streaming/

## 目次

- [概要](#概要)
- [Bidirectional Streaming とは](#bidirectional-streaming-とは)
- [ユースケース](#ユースケース)
- [アーキテクチャ](#アーキテクチャ)
- [サポートされるプロトコル](#サポートされるプロトコル)
- [次のステップ](#次のステップ)

---

## 概要

Bidirectional Streaming は、Agent とクライアント間でリアルタイムの双方向通信を実現するメカニズム。従来の単方向ストリーミングとは異なり、両者が同時にデータを送受信できる。

## Bidirectional Streaming とは

### 単方向 vs 双方向

**単方向ストリーミング**：
- サーバーからクライアントへの一方的なデータフロー
- クライアントはリクエストを送信し、サーバーの応答を待つ

**双方向ストリーミング**：
- 両方向での同時データ転送
- リアルタイムのインタラクション
- より動的な対話が可能

```
単方向:
Client ─────Request─────> Server
Client <────Response───── Server

双方向:
Client <────────────────> Server
       ↑ 同時に双方向通信 ↓
```

### 主な特徴

1. **リアルタイム通信**: 遅延なしで即座にデータを送受信
2. **インタラクティブ**: ユーザーが途中で入力可能
3. **効率的**: 単一のコネクションで複数のメッセージを処理
4. **柔軟性**: 様々なプロトコルをサポート

## ユースケース

### 1. チャットアプリケーション

```python
# ユーザーがリアルタイムで入力しながら
# Agent の応答も同時に受信
async for event in agent.bidirectional_stream():
    if event.type == "user_input":
        # ユーザーの新しい入力を処理
        await process_user_input(event.data)
    elif event.type == "agent_response":
        # Agent の応答を表示
        display_response(event.data)
```

### 2. コラボレーティブ編集

複数のユーザーと Agent が同時にドキュメントを編集：

```python
from strands import Agent
from strands.streaming import BidirectionalStream

async def collaborative_editing():
    stream = BidirectionalStream(agent)
    
    async for event in stream:
        if event.type == "user_edit":
            await stream.send_to_agent(event.data)
        elif event.type == "agent_suggestion":
            await broadcast_to_users(event.data)
```

### 3. リアルタイム翻訳

音声入力を受けながら翻訳結果をリアルタイムで出力：

```python
async def realtime_translation():
    async for event in bidirectional_stream:
        if event.type == "audio_chunk":
            # 音声をテキストに変換
            text = await transcribe(event.data)
            await stream.send(text)
        elif event.type == "translation":
            # 翻訳結果を出力
            yield event.data
```

### 4. Human-in-the-Loop

Agent の処理中にユーザーが介入可能：

```python
async def human_in_loop_flow():
    stream = BidirectionalStream(agent)
    
    async for event in stream:
        if event.type == "approval_request":
            # ユーザーに承認を求める
            approval = await get_user_approval(event.data)
            await stream.send({"approved": approval})
        elif event.type == "progress":
            # 進捗を表示
            update_progress_bar(event.data)
```

## アーキテクチャ

### コンポーネント

```
┌─────────────────────────────────────────────────────────┐
│                     Client                               │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐ │
│  │ Input Queue │    │ Output Queue │    │ Event Loop  │ │
│  └──────┬──────┘    └──────┬──────┘    └──────┬───────┘ │
│         │                   │                   │        │
└─────────┼───────────────────┼───────────────────┼────────┘
          │                   │                   │
          ▼                   ▲                   ▼
┌─────────────────────────────────────────────────────────┐
│                   Transport Layer                        │
│         (WebSocket / gRPC / SSE + POST)                 │
└─────────────────────────────────────────────────────────┘
          │                   │                   │
          ▼                   ▲                   ▼
┌─────────────────────────────────────────────────────────┐
│                      Server                              │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐ │
│  │ Request     │    │ Response    │    │ Agent        │ │
│  │ Handler     │────│ Handler     │────│ Runtime      │ │
│  └─────────────┘    └─────────────┘    └──────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### データフロー

1. **クライアント → サーバー**: ユーザー入力、制御信号
2. **サーバー → クライアント**: Agent 応答、イベント通知
3. **双方向**: 状態同期、ハートビート

## サポートされるプロトコル

### WebSocket

最も一般的な双方向通信プロトコル：

```python
from strands.streaming import WebSocketStream

stream = WebSocketStream(
    url="ws://localhost:8080/agent",
    agent=agent
)

await stream.connect()
```

### gRPC Streaming

高パフォーマンスな双方向 RPC：

```python
from strands.streaming import GrpcStream

stream = GrpcStream(
    host="localhost",
    port=50051,
    agent=agent
)
```

### SSE + HTTP POST

Server-Sent Events とPOST リクエストの組み合わせ：

```python
from strands.streaming import SsePostStream

stream = SsePostStream(
    sse_url="http://localhost:8080/events",
    post_url="http://localhost:8080/messages",
    agent=agent
)
```

## 次のステップ

- [Quick Start](./02-quick-start.md): 双方向ストリーミングを始める
- [WebSocket Integration](./03-websocket.md): WebSocket の設定と使用
- [Event Types](./08-event-types.md): イベントの種類と処理
- [Error Handling](./09-error-handling.md): エラー処理
