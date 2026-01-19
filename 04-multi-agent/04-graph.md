# Graph

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/graph/

## 目次

- [概要](#概要)
- [Graph の構造](#graph-の構造)
- [基本的な使用法](#基本的な使用法)
- [ノードとエッジ](#ノードとエッジ)
- [条件付きルーティング](#条件付きルーティング)
- [使用例](#使用例)

---

## 概要

Graph は、Agent のワークフローを有向グラフとして定義するパターン。ノード（Agent）とエッジ（遷移）を明示的に定義することで、複雑なワークフローを構築できる。

Graph パターンの主な特徴：

- **明示的なフロー制御**: ワークフローの流れを明確に定義
- **条件付き分岐**: 結果に応じて異なるパスを選択
- **サイクル対応**: 必要に応じてループを含むフローを構築
- **可視化可能**: グラフ構造として可視化が容易

## Graph の構造

Graph は以下の要素で構成される：

1. **ノード (Nodes)**: 処理を行う Agent または関数
2. **エッジ (Edges)**: ノード間の遷移
3. **開始ノード**: ワークフローの開始点
4. **終了ノード**: ワークフローの終了点

```
[開始] → [Agent A] → [条件分岐] → [Agent B] → [終了]
                           ↓
                      [Agent C] → [終了]
```

## 基本的な使用法

### Graph の作成

```python
from strands import Agent
from strands.multiagent import Graph

# Agent を定義
classifier = Agent(
    name="classifier",
    system_prompt="入力を分類し、カテゴリを返します。"
)

processor_a = Agent(
    name="processor_a",
    system_prompt="カテゴリ A の処理を行います。"
)

processor_b = Agent(
    name="processor_b",
    system_prompt="カテゴリ B の処理を行います。"
)

# Graph を構築
graph = Graph()

# ノードを追加
graph.add_node("classify", classifier)
graph.add_node("process_a", processor_a)
graph.add_node("process_b", processor_b)

# エッジを追加
graph.add_edge("classify", "process_a", condition=lambda x: x["category"] == "A")
graph.add_edge("classify", "process_b", condition=lambda x: x["category"] == "B")

# 開始・終了ノードを設定
graph.set_entry_point("classify")
graph.set_finish_point("process_a")
graph.set_finish_point("process_b")

# 実行
result = graph.run("このテキストを処理してください")
```

## ノードとエッジ

### ノードの種類

```python
from strands import Agent
from strands.multiagent import Graph

graph = Graph()

# 1. Agent ノード
agent_node = Agent(name="agent", system_prompt="...")
graph.add_node("agent", agent_node)

# 2. 関数ノード
def process_data(state):
    # データ処理ロジック
    return {"processed": True, **state}

graph.add_node("processor", process_data)

# 3. 条件ノード
def check_condition(state):
    if state.get("valid"):
        return "success_path"
    else:
        return "error_path"

graph.add_conditional_node("check", check_condition, {
    "success_path": "success_handler",
    "error_path": "error_handler"
})
```

### エッジの定義

```python
from strands.multiagent import Graph

graph = Graph()

# 無条件エッジ
graph.add_edge("node_a", "node_b")

# 条件付きエッジ
graph.add_edge("node_a", "node_b", condition=lambda x: x["score"] > 0.5)
graph.add_edge("node_a", "node_c", condition=lambda x: x["score"] <= 0.5)

# 複数の出力エッジ
graph.add_edges("node_a", ["node_b", "node_c", "node_d"])
```

## 条件付きルーティング

### 条件分岐

```python
from strands import Agent
from strands.multiagent import Graph

graph = Graph()

# 分類 Agent
classifier = Agent(
    name="classifier",
    system_prompt="テキストのセンチメントを分析し、positive/negative/neutral を返します。"
)

graph.add_node("classify", classifier)

# 条件付きルーティング
def route_by_sentiment(state):
    sentiment = state.get("sentiment", "neutral")
    return f"handle_{sentiment}"

graph.add_conditional_node("router", route_by_sentiment, {
    "handle_positive": "positive_handler",
    "handle_negative": "negative_handler",
    "handle_neutral": "neutral_handler"
})

graph.add_edge("classify", "router")
```

### ループ（サイクル）

```python
from strands import Agent
from strands.multiagent import Graph

graph = Graph()

# 検証 Agent
validator = Agent(
    name="validator",
    system_prompt="結果を検証します。問題があれば修正が必要であることを示します。"
)

# 修正 Agent
fixer = Agent(
    name="fixer",
    system_prompt="問題を修正します。"
)

graph.add_node("validate", validator)
graph.add_node("fix", fixer)

# ループを含むフロー
graph.add_edge("validate", "fix", condition=lambda x: not x.get("valid", False))
graph.add_edge("fix", "validate")  # 修正後に再検証
graph.add_edge("validate", "end", condition=lambda x: x.get("valid", False))
```

## 使用例

### データ処理パイプライン

```python
from strands import Agent
from strands.multiagent import Graph

# 各ステップの Agent
extractor = Agent(name="extractor", system_prompt="データを抽出します。")
transformer = Agent(name="transformer", system_prompt="データを変換します。")
validator = Agent(name="validator", system_prompt="データを検証します。")
loader = Agent(name="loader", system_prompt="データをロードします。")

graph = Graph()

# ETL パイプライン
graph.add_node("extract", extractor)
graph.add_node("transform", transformer)
graph.add_node("validate", validator)
graph.add_node("load", loader)

graph.add_edge("extract", "transform")
graph.add_edge("transform", "validate")

# 検証結果に応じた分岐
graph.add_edge("validate", "load", condition=lambda x: x.get("valid"))
graph.add_edge("validate", "transform", condition=lambda x: not x.get("valid"))

graph.set_entry_point("extract")
graph.set_finish_point("load")

result = graph.run("データを処理してください")
```

### 承認ワークフロー

```python
from strands import Agent
from strands.multiagent import Graph

# ワークフローの各ステップ
drafter = Agent(name="drafter", system_prompt="ドキュメントを作成します。")
reviewer = Agent(name="reviewer", system_prompt="ドキュメントをレビューします。")
approver = Agent(name="approver", system_prompt="ドキュメントを承認します。")
publisher = Agent(name="publisher", system_prompt="ドキュメントを公開します。")

graph = Graph()

graph.add_node("draft", drafter)
graph.add_node("review", reviewer)
graph.add_node("approve", approver)
graph.add_node("publish", publisher)

# ワークフローの流れ
graph.add_edge("draft", "review")

# レビュー結果に応じた分岐
def review_router(state):
    status = state.get("review_status")
    if status == "approved":
        return "to_approve"
    elif status == "needs_revision":
        return "to_draft"
    else:
        return "to_review"

graph.add_conditional_node("review_check", review_router, {
    "to_approve": "approve",
    "to_draft": "draft",
    "to_review": "review"
})

graph.add_edge("review", "review_check")

# 承認結果に応じた分岐
graph.add_edge("approve", "publish", condition=lambda x: x.get("approved"))
graph.add_edge("approve", "draft", condition=lambda x: not x.get("approved"))

graph.set_entry_point("draft")
graph.set_finish_point("publish")
```

### 並列処理

```python
from strands import Agent
from strands.multiagent import Graph

# 並列で実行される Agent
analyzer1 = Agent(name="analyzer1", system_prompt="技術分析を行います。")
analyzer2 = Agent(name="analyzer2", system_prompt="市場分析を行います。")
analyzer3 = Agent(name="analyzer3", system_prompt="競合分析を行います。")
aggregator = Agent(name="aggregator", system_prompt="分析結果を統合します。")

graph = Graph()

# 並列ノード
graph.add_node("analyze_tech", analyzer1)
graph.add_node("analyze_market", analyzer2)
graph.add_node("analyze_competitor", analyzer3)
graph.add_node("aggregate", aggregator)

# 開始点から並列に分岐
graph.add_edge("start", "analyze_tech")
graph.add_edge("start", "analyze_market")
graph.add_edge("start", "analyze_competitor")

# 並列結果を統合
graph.add_edge("analyze_tech", "aggregate")
graph.add_edge("analyze_market", "aggregate")
graph.add_edge("analyze_competitor", "aggregate")

graph.set_entry_point("start")
graph.set_finish_point("aggregate")
```

## ベストプラクティス

1. **明確なノード名**: 各ノードに説明的な名前を付ける
2. **エラーハンドリング**: エラーパスを明示的に定義する
3. **タイムアウト設定**: 長時間実行を防ぐためにタイムアウトを設定
4. **ログ出力**: 各ノードの実行をログに記録する
5. **テスト**: 各パスを個別にテストする
