# Multi-agent Patterns

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/multi-agent-patterns/

## 目次

- [概要](#概要)
- [パターン比較](#パターン比較)
- [選択ガイド](#選択ガイド)
- [組み合わせパターン](#組み合わせパターン)
- [ベストプラクティス](#ベストプラクティス)

---

## 概要

Strands は複数のマルチエージェントパターンを提供し、それぞれ異なるユースケースに適している。この文書では、各パターンの特徴と選択基準について説明する。

利用可能なパターン：

1. **Agents as Tools** - Agent を別の Agent の Tool として使用
2. **Swarm** - Agent 間の動的なハンドオフ
3. **Graph** - 有向グラフベースのワークフロー
4. **Workflow** - シーケンシャルなステップ実行
5. **Agent2Agent (A2A)** - 分散 Agent 間の通信

## パターン比較

### 構造の比較

| パターン | 構造 | 複雑さ | 状態共有 |
|---------|------|--------|----------|
| Agents as Tools | 階層的 | 低 | 親から子へ |
| Swarm | フラット | 中 | 完全共有 |
| Graph | 有向グラフ | 高 | ノード間で渡す |
| Workflow | 線形 | 低 | 順次渡す |
| A2A | 分散 | 高 | メッセージング |

### 制御フローの比較

| パターン | 制御の流れ | 分岐 | ループ |
|---------|-----------|------|--------|
| Agents as Tools | 親 Agent が制御 | なし | なし |
| Swarm | 動的ハンドオフ | 動的 | 可能 |
| Graph | 明示的なエッジ | 条件付き | 可能 |
| Workflow | 順次実行 | 限定的 | なし |
| A2A | 非同期/分散 | 動的 | 可能 |

## 選択ガイド

### Agents as Tools を選ぶ場合

- シンプルな階層構造が必要
- 親 Agent が完全に制御を維持
- 専門家への単純な委任

```python
# ユースケース: 専門家への質問
from strands import Agent

expert = Agent(name="expert", system_prompt="専門家です")
coordinator = Agent(tools=[expert], system_prompt="必要に応じて専門家に相談")
```

### Swarm を選ぶ場合

- 動的なルーティングが必要
- 会話の流れに応じて Agent を切り替え
- カスタマーサポートなどの対話型システム

```python
# ユースケース: カスタマーサポート
from strands import Agent
from strands.multiagent import Swarm

sales = Agent(name="sales", system_prompt="販売担当")
support = Agent(name="support", system_prompt="サポート担当")
swarm = Swarm(agents=[sales, support], default_agent="sales")
```

### Graph を選ぶ場合

- 複雑な分岐ロジックが必要
- 条件付きのパス選択
- ループを含むワークフロー

```python
# ユースケース: 承認ワークフロー
from strands.multiagent import Graph

graph = Graph()
graph.add_node("submit", submitter)
graph.add_node("review", reviewer)
graph.add_node("approve", approver)
graph.add_conditional_edge("review", {...})
```

### Workflow を選ぶ場合

- シンプルな順次処理
- 明確なステップの連続
- パイプライン処理

```python
# ユースケース: データ処理パイプライン
from strands.multiagent import Workflow

workflow = (
    Workflow()
    .add_step("extract", extractor)
    .add_step("transform", transformer)
    .add_step("load", loader)
)
```

### A2A を選ぶ場合

- 分散システム
- マイクロサービスアーキテクチャ
- 異なるサービス間での Agent 連携

```python
# ユースケース: 分散サービス
from strands.multiagent.a2a import A2AExecutor

remote_agent = A2AExecutor(url="http://remote-service:8080")
local_agent = Agent(tools=[remote_agent])
```

## 組み合わせパターン

### Swarm + Graph

動的なハンドオフとグラフベースのワークフローを組み合わせ：

```python
from strands import Agent
from strands.multiagent import Swarm, Graph

# 各部門は内部で Graph を使用
support_graph = Graph()
support_graph.add_node("diagnose", diagnose_agent)
support_graph.add_node("solve", solve_agent)
support_graph.add_edge("diagnose", "solve")

# Swarm で部門間をハンドオフ
support_swarm_agent = Agent(
    name="support",
    graph=support_graph,
    system_prompt="技術サポート部門"
)

main_swarm = Swarm(
    agents=[sales_agent, support_swarm_agent, billing_agent]
)
```

### Workflow + Agents as Tools

ワークフローの各ステップで専門家を活用：

```python
from strands import Agent
from strands.multiagent import Workflow

# 専門家 Agent
researcher = Agent(name="researcher", system_prompt="調査担当")
analyst = Agent(name="analyst", system_prompt="分析担当")

# ワークフローの各ステップで専門家を使用
step1 = Agent(tools=[researcher], system_prompt="調査を依頼")
step2 = Agent(tools=[analyst], system_prompt="分析を依頼")

workflow = (
    Workflow()
    .add_step("research", step1)
    .add_step("analyze", step2)
)
```

### A2A + Local Agents

リモートとローカルの Agent を組み合わせ：

```python
from strands import Agent
from strands.multiagent.a2a import A2AExecutor

# リモート Agent
cloud_ml = A2AExecutor(url="http://ml-service:8080", name="ml_expert")

# ローカル Agent
local_processor = Agent(name="processor", system_prompt="データ前処理")

# 組み合わせ
orchestrator = Agent(
    tools=[cloud_ml, local_processor],
    system_prompt="""
    データ処理はローカルで行い、
    ML 推論はリモートサービスを使用します。
    """
)
```

## ベストプラクティス

### 1. 適切なパターンの選択

```
シンプルな委任 → Agents as Tools
対話型ルーティング → Swarm
複雑なワークフロー → Graph
順次処理 → Workflow
分散システム → A2A
```

### 2. 責任の明確化

各 Agent の責任を明確に定義：

```python
# 良い例
agent = Agent(
    name="invoice_processor",
    system_prompt="請求書の処理のみを担当。他のタスクは適切な Agent に委任。"
)

# 悪い例
agent = Agent(
    name="helper",
    system_prompt="何でもやります"
)
```

### 3. エラーハンドリング

すべてのパターンでエラーを適切に処理：

```python
try:
    result = multi_agent_system.run(task)
except AgentError as e:
    # Agent レベルのエラー
    handle_agent_error(e)
except CommunicationError as e:
    # 通信エラー（A2A の場合）
    handle_communication_error(e)
except TimeoutError as e:
    # タイムアウト
    handle_timeout(e)
```

### 4. 監視とログ

```python
import logging

logging.basicConfig(level=logging.INFO)

# 各 Agent の動作をログに記録
agent = Agent(
    name="monitored_agent",
    system_prompt="...",
    callback_handler=LoggingCallbackHandler()
)
```

### 5. テスト戦略

```python
# 各パターンのテスト方針
def test_agents_as_tools():
    # 親 Agent と子 Agent を個別にテスト
    pass

def test_swarm():
    # 各ハンドオフパスをテスト
    pass

def test_graph():
    # 各ノードと各パスをテスト
    pass

def test_workflow():
    # 各ステップを個別にテスト
    pass

def test_a2a():
    # ローカルで A2A サーバーをモック
    pass
```

### 6. スケーラビリティ

大規模システムでの考慮事項：

- **Agents as Tools**: Agent の階層が深くなりすぎないように
- **Swarm**: Agent 数が増えるとハンドオフが複雑に
- **Graph**: ノード数に応じて実行時間が増加
- **Workflow**: ステップ数は妥当な範囲に
- **A2A**: ネットワーク遅延と負荷分散を考慮
