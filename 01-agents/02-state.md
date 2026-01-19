# State

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/state/

## 目次

- [概要](#概要)
- [Conversation History（会話履歴）](#conversation-history会話履歴)
- [Agent State（エージェントステート）](#agent-stateエージェントステート)
- [Request State（リクエストステート）](#request-stateリクエストステート)
- [次のステップ](#次のステップ)

---

## 概要

Strands Agents は、Agent の実行中にコンテキストを維持するための複数のステート管理メカニズムを提供する。これらの異なるステートタイプを理解することで、より効果的でステートフルな Agent アプリケーションを構築できる。

このガイドでは、以下の 3 つの主要なステート管理メカニズムについて説明する：

1. **Conversation History** - モデルのコンテキストウィンドウ内で会話のやり取りを追跡
2. **Agent State** - Agent のライフタイム全体で永続化する構造化データ
3. **Request State** - 単一のリクエスト内で一時的なコンテキストを保持

## Conversation History（会話履歴）

Conversation History は、Agent とのインタラクション中にすべてのメッセージを追跡する中心的なメカニズムである。ユーザーと Assistant の両方のメッセージ、Tool 呼び出し、および結果を含む。

### 仕組み

Conversation History は Agent の `messages` 属性に保存される：

```python
agent = Agent()
agent("Hello!")
print(agent.messages)  # 会話履歴を表示
```

### Conversation Manager

Conversation Manager は、モデルのコンテキストウィンドウ内で Conversation History を管理するための戦略を提供する。デフォルトでは、`SlidingWindowConversationManager` が使用され、コンテキストウィンドウがいっぱいになると古いメッセージを削除する。

```python
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

agent = Agent(
    conversation_manager=SlidingWindowConversationManager(
        window_size=10,  # 保持するメッセージ数
    )
)
```

詳細については [Conversation Management](./07-conversation-management.md) を参照。

## Agent State（エージェントステート）

Agent State は、会話コンテキスト外でステートフルな情報を永続化するためのキーバリューストレージを提供する。これは、Tool やアプリケーションロジックからアクセスでき、ユーザープリファレンス、認証トークン、蓄積データなどを保存するのに適している。

### 基本的な使用法

```python
from strands import Agent

agent = Agent()

# ステートの設定
agent.state["user_preference"] = "dark_mode"
agent.state["session_count"] = 1

# ステートの取得
preference = agent.state.get("user_preference")

# ステートの更新
agent.state["session_count"] += 1
```

### Tool からのアクセス

Tool は `tool_context` パラメータを通じて Agent State にアクセスできる：

```python
from strands import tool

@tool
def get_user_preference(tool_context):
    """ユーザーのプリファレンスを取得する"""
    return tool_context.state.get("user_preference", "default")

@tool
def set_user_preference(preference: str, tool_context):
    """ユーザーのプリファレンスを設定する"""
    tool_context.state["user_preference"] = preference
    return f"プリファレンスを {preference} に設定しました"
```

### ユースケース

- **ユーザープリファレンス**: 言語設定、表示オプションなど
- **認証情報**: APIトークン、セッション ID
- **蓄積データ**: 会話を通じて収集された情報
- **設定フラグ**: 機能の有効/無効状態

## Request State（リクエストステート）

Request State は、単一のリクエスト内で維持されるコンテキスト情報を提供する。イベントループのサイクル全体を通じて永続化され、リクエストスコープのデータを管理するのに役立つ。

### 基本的な使用法

```python
from strands import Agent

agent = Agent()

# Request State を使用してリクエストを実行
result = agent(
    "タスクを実行してください",
    request_state={
        "request_id": "req-12345",
        "timestamp": "2024-01-15T10:30:00Z",
        "user_id": "user-789"
    }
)
```

### Tool からのアクセス

Tool は `tool_context` を通じて Request State にアクセスできる：

```python
from strands import tool

@tool
def log_action(action: str, tool_context):
    """アクションをログに記録する"""
    request_id = tool_context.request_state.get("request_id")
    user_id = tool_context.request_state.get("user_id")
    
    # ログ記録
    print(f"[{request_id}] User {user_id}: {action}")
    return "ログ記録完了"
```

### Agent State との違い

| 特性 | Agent State | Request State |
|------|-------------|---------------|
| ライフタイム | Agent のライフタイム全体 | 単一リクエスト |
| 永続化 | Session Manager で永続化可能 | リクエスト終了で破棄 |
| 用途 | 長期的なデータ保存 | リクエストスコープのコンテキスト |
| 例 | ユーザープリファレンス | リクエスト ID、トレース ID |

## 次のステップ

- [Session Management](./03-session-management.md) - ステートの永続化について学ぶ
- [Conversation Management](./07-conversation-management.md) - 会話履歴の管理戦略
- [Tools](../02-tools/01-tools-overview.md) - Tool でのステートアクセス
