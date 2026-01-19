# Streaming

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/streaming/

## 目次

- [概要](#概要)
- [ストリーミングの方法](#ストリーミングの方法)
- [ストリーミングイベント](#ストリーミングイベント)
- [次のステップ](#次のステップ)

---

## 概要

ストリーミングにより、Agent は応答が完全に生成される前に、応答の一部をリアルタイムで提供できる。これにより、ユーザーエクスペリエンスが向上し、応答待ち時間が短縮される。

Strands Agents は、Agent の実行中に発生するイベントをストリーミングするための複数の方法を提供する：

1. **Async Iterators** - 非同期イテレータを使用してイベントを処理
2. **Callback Handlers** - コールバック関数を使用してイベントをキャプチャ

## ストリーミングの方法

### 1. Async Iterators（非同期イテレータ）

`stream_async` メソッドを使用して、Agent の実行中にイベントを非同期で取得できる：

```python
import asyncio
from strands import Agent

async def main():
    agent = Agent()
    
    async for event in agent.stream_async("ストーリーを書いてください"):
        if "data" in event:
            print(event["data"], end="", flush=True)

asyncio.run(main())
```

### 2. Callback Handlers（コールバックハンドラー）

`callback_handler` を使用して、特定のイベントタイプを処理できる：

```python
from strands import Agent
from strands.handlers import PrintingCallbackHandler

# 組み込みの PrintingCallbackHandler を使用
agent = Agent(callback_handler=PrintingCallbackHandler())
agent("ストーリーを書いてください")
```

## ストリーミングイベント

Agent の実行中に様々なタイプのイベントが発生する：

| イベントタイプ | 説明 |
|--------------|------|
| `data` | モデルからのテキストチャンク |
| `current_tool_use` | 現在実行中の Tool の情報 |
| `tool_stream_event` | Tool からのストリーミング更新 |
| `message` | 完了したメッセージ |
| `complete` | Agent の実行完了 |
| `error` | エラー情報 |

### イベントの処理例

```python
import asyncio
from strands import Agent

async def process_events():
    agent = Agent()
    
    async for event in agent.stream_async("タスクを実行してください"):
        if "data" in event:
            # テキストチャンクを処理
            print(f"テキスト: {event['data']}")
        
        elif "current_tool_use" in event:
            # Tool の使用を追跡
            tool = event["current_tool_use"]
            print(f"Tool 使用中: {tool['name']}")
        
        elif "tool_stream_event" in event:
            # Tool からのストリーミング更新
            update = event["tool_stream_event"]
            print(f"Tool 更新: {update.get('data')}")
        
        elif "message" in event:
            # 完了したメッセージ
            print("メッセージが完了しました")
        
        elif "complete" in event:
            # 実行完了
            print("Agent の実行が完了しました")

asyncio.run(process_events())
```

## 使用ケース

ストリーミングは以下のような場合に特に有用：

- **チャットボット**: ユーザーに即座にフィードバックを提供
- **長時間実行タスク**: 進捗状況をリアルタイムで表示
- **インタラクティブアプリケーション**: レスポンシブな UI を構築
- **デバッグ**: Agent の実行を詳細に追跡

## 次のステップ

詳細な実装については、以下を参照：

- [Async Iterators](./02-async-iterators.md) - 非同期イテレータの詳細な使用方法
- [Callback Handlers](./03-callback-handlers.md) - コールバックハンドラーの詳細な使用方法
