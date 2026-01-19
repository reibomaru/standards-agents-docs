# Async Iterators

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/streaming/async-iterators/

## 目次

- [概要](#概要)
- [基本的な使用法](#基本的な使用法)
- [イベントタイプ](#イベントタイプ)
- [高度な使用法](#高度な使用法)

---

## 概要

Async Iterators（非同期イテレータ）は、Agent からのストリーミングイベントを処理するための Python の標準的な方法を提供する。`stream_async` メソッドを使用して、Agent の実行中に発生するイベントを非同期的に取得できる。

この方法は、非同期プログラミングパターンに慣れている開発者や、より細かい制御が必要な場合に最適。

## 基本的な使用法

### シンプルなテキストストリーミング

```python
import asyncio
from strands import Agent

async def stream_text():
    agent = Agent()
    
    async for event in agent.stream_async("物語を書いてください"):
        if "data" in event:
            # テキストチャンクを出力
            print(event["data"], end="", flush=True)
    
    print()  # 改行

asyncio.run(stream_text())
```

### 完全なイベント処理

```python
import asyncio
from strands import Agent

async def process_all_events():
    agent = Agent()
    
    async for event in agent.stream_async("タスクを実行してください"):
        # イベントの種類に応じて処理
        event_type = next(iter(event.keys()), None)
        
        match event_type:
            case "data":
                print(f"[TEXT] {event['data']}", end="")
            case "current_tool_use":
                tool = event["current_tool_use"]
                print(f"\n[TOOL] Using: {tool['name']}")
            case "tool_stream_event":
                update = event["tool_stream_event"]
                print(f"[TOOL UPDATE] {update.get('data')}")
            case "message":
                print("\n[MESSAGE] Complete")
            case "complete":
                print("[DONE] Agent finished")
            case "error":
                print(f"[ERROR] {event['error']}")

asyncio.run(process_all_events())
```

## イベントタイプ

### data イベント

モデルからのテキストチャンクを含む：

```python
async for event in agent.stream_async(prompt):
    if "data" in event:
        text_chunk = event["data"]
        # テキストを処理
```

### current_tool_use イベント

現在実行中の Tool に関する情報：

```python
async for event in agent.stream_async(prompt):
    if "current_tool_use" in event:
        tool_use = event["current_tool_use"]
        tool_name = tool_use.get("name")
        tool_input = tool_use.get("input")
        print(f"Tool: {tool_name}, Input: {tool_input}")
```

### tool_stream_event イベント

Tool からのストリーミング更新（Tool が yield する場合）：

```python
async for event in agent.stream_async(prompt):
    if "tool_stream_event" in event:
        stream_event = event["tool_stream_event"]
        tool_use = stream_event.get("tool_use")
        data = stream_event.get("data")
        print(f"Tool update from {tool_use.get('name')}: {data}")
```

### message イベント

完了したメッセージ：

```python
async for event in agent.stream_async(prompt):
    if "message" in event:
        message = event["message"]
        # 完了したメッセージを処理
```

### complete イベント

Agent の実行完了：

```python
async for event in agent.stream_async(prompt):
    if "complete" in event:
        result = event["complete"]
        print("Agent の実行が完了しました")
```

### error イベント

エラー情報：

```python
async for event in agent.stream_async(prompt):
    if "error" in event:
        error = event["error"]
        print(f"エラーが発生しました: {error}")
```

## 高度な使用法

### タイムアウト処理

```python
import asyncio
from strands import Agent

async def stream_with_timeout():
    agent = Agent()
    
    try:
        async with asyncio.timeout(30):  # 30秒のタイムアウト
            async for event in agent.stream_async("長いタスクを実行"):
                if "data" in event:
                    print(event["data"], end="")
    except asyncio.TimeoutError:
        print("\n処理がタイムアウトしました")

asyncio.run(stream_with_timeout())
```

### イベントの収集

```python
import asyncio
from strands import Agent

async def collect_events():
    agent = Agent()
    
    events = []
    full_response = ""
    
    async for event in agent.stream_async("質問に答えてください"):
        events.append(event)
        
        if "data" in event:
            full_response += event["data"]
    
    print(f"合計 {len(events)} イベント")
    print(f"完全な応答: {full_response}")
    
    return events, full_response

asyncio.run(collect_events())
```

### 複数の Agent の並列ストリーミング

```python
import asyncio
from strands import Agent

async def stream_multiple():
    agent1 = Agent(system_prompt="あなたは詩人です")
    agent2 = Agent(system_prompt="あなたは科学者です")
    
    async def process_agent(agent, name, prompt):
        print(f"\n{name} の応答:")
        async for event in agent.stream_async(prompt):
            if "data" in event:
                print(event["data"], end="")
        print()
    
    await asyncio.gather(
        process_agent(agent1, "詩人", "春について詩を書いてください"),
        process_agent(agent2, "科学者", "春について科学的に説明してください")
    )

asyncio.run(stream_multiple())
```

### Web フレームワークとの統合

FastAPI での SSE (Server-Sent Events) の例：

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from strands import Agent
import asyncio

app = FastAPI()

async def generate_stream(prompt: str):
    agent = Agent()
    
    async for event in agent.stream_async(prompt):
        if "data" in event:
            yield f"data: {event['data']}\n\n"
    
    yield "data: [DONE]\n\n"

@app.get("/stream")
async def stream_endpoint(prompt: str):
    return StreamingResponse(
        generate_stream(prompt),
        media_type="text/event-stream"
    )
```
