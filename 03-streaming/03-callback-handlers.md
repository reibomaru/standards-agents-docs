# Callback Handlers

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/streaming/callback-handlers/

## 目次

- [概要](#概要)
- [組み込み Callback Handler](#組み込み-callback-handler)
- [カスタム Callback Handler](#カスタム-callback-handler)
- [コールバックメソッド](#コールバックメソッド)
- [使用例](#使用例)

---

## 概要

Callback Handler は、Agent の実行中に発生するイベントを処理するためのコールバックベースのインターフェースを提供する。これにより、特定のイベントタイプに対してカスタム処理を定義できる。

Async Iterators と比較して、Callback Handler は以下の利点がある：

- 同期コードとの統合が容易
- イベントタイプごとに明確に分離された処理
- 既存のコードベースへの組み込みが簡単

## 組み込み Callback Handler

Strands は、すぐに使用できる組み込みの Callback Handler を提供している。

### PrintingCallbackHandler

最もシンプルなハンドラーで、テキストを標準出力に出力する：

```python
from strands import Agent
from strands.handlers import PrintingCallbackHandler

agent = Agent(callback_handler=PrintingCallbackHandler())
agent("物語を書いてください")
```

### NullCallbackHandler

何も出力しないハンドラー。ストリーミング出力を抑制したい場合に使用：

```python
from strands import Agent
from strands.handlers import NullCallbackHandler

agent = Agent(callback_handler=NullCallbackHandler())
result = agent("静かに計算してください")
# 中間出力なし、結果のみ取得
print(result)
```

## カスタム Callback Handler

独自の Callback Handler を作成して、特定のニーズに合わせた処理を実装できる。

### 基本構造

```python
from strands.handlers import CallbackHandler

class MyCallbackHandler(CallbackHandler):
    def on_stream_start(self):
        """ストリーミング開始時に呼び出される"""
        print("ストリーミング開始")
    
    def on_stream_end(self):
        """ストリーミング終了時に呼び出される"""
        print("ストリーミング終了")
    
    def on_llm_chunk(self, text: str, **kwargs):
        """テキストチャンク受信時に呼び出される"""
        print(text, end="", flush=True)
    
    def on_tool_start(self, tool_name: str, tool_input: dict, **kwargs):
        """Tool 実行開始時に呼び出される"""
        print(f"\nTool 開始: {tool_name}")
    
    def on_tool_end(self, tool_name: str, tool_result: dict, **kwargs):
        """Tool 実行終了時に呼び出される"""
        print(f"Tool 終了: {tool_name}")
    
    def on_error(self, error: Exception, **kwargs):
        """エラー発生時に呼び出される"""
        print(f"エラー: {error}")
```

### カスタムハンドラーの使用

```python
from strands import Agent

agent = Agent(callback_handler=MyCallbackHandler())
agent("タスクを実行してください")
```

## コールバックメソッド

| メソッド | 説明 | パラメータ |
|---------|------|-----------|
| `on_stream_start` | ストリーミング開始時 | なし |
| `on_stream_end` | ストリーミング終了時 | なし |
| `on_llm_chunk` | テキストチャンク受信時 | `text: str` |
| `on_tool_start` | Tool 実行開始時 | `tool_name: str`, `tool_input: dict` |
| `on_tool_end` | Tool 実行終了時 | `tool_name: str`, `tool_result: dict` |
| `on_tool_stream` | Tool ストリーミング更新時 | `tool_name: str`, `data: Any` |
| `on_message_complete` | メッセージ完了時 | `message: dict` |
| `on_error` | エラー発生時 | `error: Exception` |

## 使用例

### ログ収集ハンドラー

```python
from strands import Agent
from strands.handlers import CallbackHandler
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LoggingCallbackHandler(CallbackHandler):
    def __init__(self):
        self.chunks = []
        self.tools_used = []
    
    def on_llm_chunk(self, text: str, **kwargs):
        self.chunks.append(text)
        logger.debug(f"チャンク: {text}")
    
    def on_tool_start(self, tool_name: str, tool_input: dict, **kwargs):
        self.tools_used.append(tool_name)
        logger.info(f"Tool 開始: {tool_name} with input: {tool_input}")
    
    def on_tool_end(self, tool_name: str, tool_result: dict, **kwargs):
        logger.info(f"Tool 終了: {tool_name}")
    
    def on_stream_end(self):
        logger.info(f"合計チャンク数: {len(self.chunks)}")
        logger.info(f"使用された Tool: {self.tools_used}")
    
    def get_full_response(self):
        return "".join(self.chunks)

# 使用
handler = LoggingCallbackHandler()
agent = Agent(callback_handler=handler)
agent("複雑なタスクを実行してください")

print(f"完全な応答: {handler.get_full_response()}")
print(f"使用された Tool: {handler.tools_used}")
```

### UI 更新ハンドラー

```python
from strands import Agent
from strands.handlers import CallbackHandler

class UICallbackHandler(CallbackHandler):
    def __init__(self, ui_callback):
        self.ui_callback = ui_callback
    
    def on_llm_chunk(self, text: str, **kwargs):
        self.ui_callback("text", text)
    
    def on_tool_start(self, tool_name: str, tool_input: dict, **kwargs):
        self.ui_callback("tool_start", {"name": tool_name, "input": tool_input})
    
    def on_tool_end(self, tool_name: str, tool_result: dict, **kwargs):
        self.ui_callback("tool_end", {"name": tool_name, "result": tool_result})
    
    def on_error(self, error: Exception, **kwargs):
        self.ui_callback("error", str(error))

# UI フレームワークとの統合例
def update_ui(event_type, data):
    if event_type == "text":
        # UI にテキストを追加
        pass
    elif event_type == "tool_start":
        # Tool 実行中の表示
        pass
    elif event_type == "tool_end":
        # Tool 完了の表示
        pass
    elif event_type == "error":
        # エラー表示
        pass

agent = Agent(callback_handler=UICallbackHandler(update_ui))
```

### メトリクス収集ハンドラー

```python
from strands import Agent
from strands.handlers import CallbackHandler
import time

class MetricsCallbackHandler(CallbackHandler):
    def __init__(self):
        self.start_time = None
        self.end_time = None
        self.chunk_count = 0
        self.total_characters = 0
        self.tool_durations = {}
        self._tool_start_times = {}
    
    def on_stream_start(self):
        self.start_time = time.time()
    
    def on_stream_end(self):
        self.end_time = time.time()
    
    def on_llm_chunk(self, text: str, **kwargs):
        self.chunk_count += 1
        self.total_characters += len(text)
    
    def on_tool_start(self, tool_name: str, tool_input: dict, **kwargs):
        self._tool_start_times[tool_name] = time.time()
    
    def on_tool_end(self, tool_name: str, tool_result: dict, **kwargs):
        if tool_name in self._tool_start_times:
            duration = time.time() - self._tool_start_times[tool_name]
            self.tool_durations[tool_name] = duration
    
    def get_metrics(self):
        return {
            "total_duration": self.end_time - self.start_time if self.end_time else None,
            "chunk_count": self.chunk_count,
            "total_characters": self.total_characters,
            "tool_durations": self.tool_durations
        }

# 使用
handler = MetricsCallbackHandler()
agent = Agent(callback_handler=handler)
agent("分析を実行してください")

metrics = handler.get_metrics()
print(f"合計時間: {metrics['total_duration']:.2f}秒")
print(f"チャンク数: {metrics['chunk_count']}")
print(f"文字数: {metrics['total_characters']}")
print(f"Tool 実行時間: {metrics['tool_durations']}")
```
