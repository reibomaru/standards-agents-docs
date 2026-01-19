# Tool Executors

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/executors/

## 目次

- [概要](#概要)
- [Concurrent Tool Executor](#concurrent-tool-executor)
- [Sequential Tool Executor](#sequential-tool-executor)
- [Executor の選択](#executor-の選択)

---

## 概要

モデルが複数の Tool リクエストを返した場合、それらを同時に実行するか順次実行するかを制御できる。Tool Executor は、モデルからの Tool 呼び出しリクエストを処理する方法を決定するコンポーネントである。

Strands は 2 つの組み込み Executor を提供している：

1. **ConcurrentToolExecutor** - Tool 呼び出しを同時に（並列で）実行（デフォルト）
2. **SequentialToolExecutor** - Tool 呼び出しを順番に実行

## Concurrent Tool Executor

`ConcurrentToolExecutor` は Agent のデフォルトの Executor であり、複数の Tool 呼び出しを同時に実行する。これにより、独立した操作のパフォーマンスが向上する。

```python
from strands import Agent
from strands_tools import weather, time

# 同時実行（デフォルト）
agent = Agent(tools=[weather, time])

# 複数の Tool が同時に呼び出される
agent("ニューヨークの天気と現在時刻を教えてください")
```

### 利点

- **パフォーマンス**: 独立した操作が並列で実行されるため、全体の実行時間が短縮される
- **効率性**: I/O バウンドな操作（API 呼び出し、データベースクエリなど）に特に効果的

### 使用ケース

- 複数の独立した API を呼び出す場合
- 相互に依存しない情報を取得する場合
- パフォーマンスが重要な場合

## Sequential Tool Executor

`SequentialToolExecutor` は Tool 呼び出しを順番に実行する。これは、Tool の実行順序が重要な場合に使用する。

```python
from strands import Agent
from strands.tools.executors import SequentialToolExecutor

# 順次実行
agent = Agent(
    tool_executor=SequentialToolExecutor(),
    tools=[screenshot_tool, email_tool]
)

# Tool が順番に実行される
agent("スクリーンショットを撮って友達にメールしてください")
```

### 利点

- **順序の保証**: Tool が定義された順序で実行される
- **依存関係の処理**: 前の Tool の結果に依存する Tool がある場合に適切

### 使用ケース

- 前の Tool の出力が後の Tool の入力になる場合
- 副作用のある操作を特定の順序で実行する必要がある場合
- トランザクション的な操作が必要な場合

## Executor の選択

適切な Executor を選択するためのガイドライン：

| 条件 | 推奨 Executor |
|------|--------------|
| Tool が互いに独立している | ConcurrentToolExecutor |
| Tool 間に依存関係がある | SequentialToolExecutor |
| パフォーマンスが最優先 | ConcurrentToolExecutor |
| 実行順序が重要 | SequentialToolExecutor |
| I/O バウンドな操作が多い | ConcurrentToolExecutor |
| 副作用の順序が重要 | SequentialToolExecutor |

### カスタム Executor

特定のニーズに合わせて、独自の Tool Executor を実装することも可能。`ToolExecutor` インターフェースを実装して、カスタムの実行ロジックを定義できる。

```python
from strands.tools.executors import ToolExecutor

class CustomToolExecutor(ToolExecutor):
    async def execute(self, tool_calls, tool_registry, **kwargs):
        # カスタム実行ロジック
        results = []
        for tool_call in tool_calls:
            # 独自の実行ロジックを実装
            result = await self._execute_single(tool_call, tool_registry, **kwargs)
            results.append(result)
        return results
```
