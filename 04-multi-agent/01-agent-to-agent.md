# Agent2Agent (A2A)

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/agent-to-agent/

## 目次

- [概要](#概要)
- [A2A プロトコル](#a2a-プロトコル)
- [A2A サーバー](#a2a-サーバー)
- [A2A クライアント](#a2a-クライアント)
- [使用例](#使用例)

---

## 概要

Agent2Agent (A2A) は、異なる Agent 間でタスクを実行するための標準化されたプロトコル。このプロトコルにより、Agent は他の Agent にタスクを委任し、結果を受け取ることができる。

A2A は以下のような場合に特に有用：

- 分散システムでの Agent 間通信
- 異なるサービスやアプリケーション間での Agent の連携
- マイクロサービスアーキテクチャでの Agent デプロイメント

## A2A プロトコル

A2A プロトコルは、Agent 間の通信を標準化する：

1. **Task Submission** - クライアント Agent がタスクをサーバー Agent に送信
2. **Task Execution** - サーバー Agent がタスクを実行
3. **Result Return** - 結果がクライアント Agent に返される

### 通信フロー

```
クライアント Agent → タスクリクエスト → A2A サーバー
                                            ↓
                                      Agent 処理
                                            ↓
クライアント Agent ← タスク結果 ← A2A サーバー
```

## A2A サーバー

A2A サーバーは、他の Agent からのリクエストを受け付けて処理する。

### サーバーの作成

```python
from strands import Agent
from strands.multiagent.a2a import A2AServer

# Agent を作成
agent = Agent(
    system_prompt="あなたは数学の専門家です。数学の問題を解決します。"
)

# A2A サーバーとしてラップ
server = A2AServer(agent=agent)

# サーバーを起動
server.run(host="0.0.0.0", port=8080)
```

### サーバー設定

```python
from strands import Agent
from strands.multiagent.a2a import A2AServer

agent = Agent(
    system_prompt="データ分析を行う専門家です。"
)

server = A2AServer(
    agent=agent,
    name="data-analyst",
    description="データ分析と統計処理を行います",
    version="1.0.0"
)

server.run(host="0.0.0.0", port=8080)
```

## A2A クライアント

A2A クライアントは、リモートの A2A サーバーにタスクを送信する。

### クライアントの使用

```python
from strands import Agent
from strands.multiagent.a2a import A2AExecutor

# A2A Executor を作成（リモート Agent への接続）
executor = A2AExecutor(url="http://localhost:8080")

# Agent で A2A Executor を Tool として使用
agent = Agent(
    tools=[executor],
    system_prompt="必要に応じて他の専門家に相談してください。"
)

# 実行
response = agent("複雑な数学の問題を解いてください")
```

### 複数の A2A サーバーへの接続

```python
from strands import Agent
from strands.multiagent.a2a import A2AExecutor

# 複数の専門家 Agent への接続
math_expert = A2AExecutor(url="http://math-expert:8080", name="math_expert")
data_analyst = A2AExecutor(url="http://data-analyst:8080", name="data_analyst")
writer = A2AExecutor(url="http://writer:8080", name="writer")

# オーケストレーター Agent
orchestrator = Agent(
    tools=[math_expert, data_analyst, writer],
    system_prompt="""
    あなたはタスクオーケストレーターです。
    ユーザーのリクエストに応じて、適切な専門家に相談してください。
    - 数学の問題: math_expert を使用
    - データ分析: data_analyst を使用
    - 文書作成: writer を使用
    """
)

response = orchestrator("売上データを分析して、レポートを作成してください")
```

## 使用例

### 専門家ネットワーク

```python
from strands import Agent
from strands.multiagent.a2a import A2AServer, A2AExecutor

# 専門家 Agent を定義
legal_agent = Agent(system_prompt="あなたは法律の専門家です。")
finance_agent = Agent(system_prompt="あなたは財務の専門家です。")
hr_agent = Agent(system_prompt="あなたは人事の専門家です。")

# 各専門家を A2A サーバーとして起動（別プロセスで）
# A2AServer(agent=legal_agent).run(port=8081)
# A2AServer(agent=finance_agent).run(port=8082)
# A2AServer(agent=hr_agent).run(port=8083)

# クライアント側
legal_executor = A2AExecutor(url="http://localhost:8081", name="legal_expert")
finance_executor = A2AExecutor(url="http://localhost:8082", name="finance_expert")
hr_executor = A2AExecutor(url="http://localhost:8083", name="hr_expert")

# メイン Agent
main_agent = Agent(
    tools=[legal_executor, finance_executor, hr_executor],
    system_prompt="""
    あなたはビジネスアドバイザーです。
    必要に応じて専門家に相談してください。
    """
)

response = main_agent("新しい従業員を雇用する際の法的・財務的な注意点を教えてください")
```

### 非同期タスク実行

```python
import asyncio
from strands import Agent
from strands.multiagent.a2a import A2AExecutor

async def run_parallel_analysis():
    # 複数の A2A Executor
    analyzer1 = A2AExecutor(url="http://analyzer1:8080")
    analyzer2 = A2AExecutor(url="http://analyzer2:8080")
    
    agent = Agent(tools=[analyzer1, analyzer2])
    
    # 非同期実行
    result = await agent.invoke_async("両方のアナライザーでデータを分析してください")
    return result

asyncio.run(run_parallel_analysis())
```

## ベストプラクティス

1. **明確な責任分離**: 各 A2A サーバーは特定のドメインに特化させる
2. **エラーハンドリング**: ネットワークエラーやタイムアウトを適切に処理する
3. **セキュリティ**: 認証・認可を実装してアクセスを制御する
4. **監視**: A2A 通信をログに記録してデバッグを容易にする
5. **スケーラビリティ**: 負荷に応じて A2A サーバーをスケールアウトできるように設計する
