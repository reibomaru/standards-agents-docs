# Workflow

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/workflow/

## 目次

- [概要](#概要)
- [Workflow と Graph の違い](#workflow-と-graph-の違い)
- [基本的な使用法](#基本的な使用法)
- [ステップの定義](#ステップの定義)
- [使用例](#使用例)

---

## 概要

Workflow は、一連のステップを順序立てて実行するためのシンプルなパターン。Graph よりも単純な構造で、線形または準線形のフローに適している。

Workflow パターンの主な特徴：

- **シンプルさ**: 直線的なフローを簡単に定義
- **可読性**: ワークフローの流れが明確
- **メンテナンス性**: ステップの追加・削除が容易

## Workflow と Graph の違い

| 特徴 | Workflow | Graph |
|------|----------|-------|
| 構造 | 線形/シーケンシャル | 任意の有向グラフ |
| 複雑さ | シンプル | 複雑なフローに対応 |
| 分岐 | 限定的 | 豊富な条件分岐 |
| ループ | 非対応 | 対応 |
| ユースケース | 単純なパイプライン | 複雑なワークフロー |

## 基本的な使用法

### Workflow の作成

```python
from strands import Agent
from strands.multiagent import Workflow

# ステップとなる Agent を定義
step1 = Agent(name="planner", system_prompt="タスクの計画を立てます。")
step2 = Agent(name="executor", system_prompt="計画を実行します。")
step3 = Agent(name="reviewer", system_prompt="結果をレビューします。")

# Workflow を作成
workflow = Workflow()
workflow.add_step("plan", step1)
workflow.add_step("execute", step2)
workflow.add_step("review", step3)

# 実行
result = workflow.run("プロジェクトを完了させてください")
```

### チェーン形式での定義

```python
from strands import Agent
from strands.multiagent import Workflow

workflow = (
    Workflow()
    .add_step("step1", Agent(system_prompt="ステップ1"))
    .add_step("step2", Agent(system_prompt="ステップ2"))
    .add_step("step3", Agent(system_prompt="ステップ3"))
)

result = workflow.run("タスクを実行")
```

## ステップの定義

### Agent ステップ

```python
from strands import Agent
from strands.multiagent import Workflow

workflow = Workflow()

# Agent をステップとして追加
agent = Agent(name="processor", system_prompt="データを処理します。")
workflow.add_step("process", agent)
```

### 関数ステップ

```python
from strands.multiagent import Workflow

workflow = Workflow()

# 関数をステップとして追加
def validate_data(state):
    data = state.get("data")
    is_valid = len(data) > 0
    return {**state, "valid": is_valid}

workflow.add_step("validate", validate_data)
```

### 条件付きステップ

```python
from strands import Agent
from strands.multiagent import Workflow

workflow = Workflow()

# 条件付きでステップをスキップ
workflow.add_step(
    "optional_step",
    Agent(system_prompt="オプションの処理"),
    condition=lambda state: state.get("needs_processing", False)
)
```

## 使用例

### ドキュメント作成ワークフロー

```python
from strands import Agent
from strands.multiagent import Workflow

# 各ステップの Agent
outliner = Agent(
    name="outliner",
    system_prompt="ドキュメントのアウトラインを作成します。"
)

writer = Agent(
    name="writer",
    system_prompt="アウトラインに基づいて本文を執筆します。"
)

editor = Agent(
    name="editor",
    system_prompt="文章を編集し、品質を向上させます。"
)

formatter = Agent(
    name="formatter",
    system_prompt="ドキュメントをフォーマットします。"
)

# Workflow の構築
doc_workflow = (
    Workflow()
    .add_step("outline", outliner)
    .add_step("write", writer)
    .add_step("edit", editor)
    .add_step("format", formatter)
)

result = doc_workflow.run("AI についての技術レポートを作成してください")
```

### データ分析ワークフロー

```python
from strands import Agent
from strands.multiagent import Workflow

# 分析ステップ
data_collector = Agent(
    name="collector",
    system_prompt="データを収集します。"
)

data_cleaner = Agent(
    name="cleaner",
    system_prompt="データをクレンジングします。"
)

data_analyzer = Agent(
    name="analyzer",
    system_prompt="データを分析して洞察を抽出します。"
)

report_generator = Agent(
    name="reporter",
    system_prompt="分析結果をレポートにまとめます。"
)

# Workflow
analysis_workflow = (
    Workflow()
    .add_step("collect", data_collector)
    .add_step("clean", data_cleaner)
    .add_step("analyze", data_analyzer)
    .add_step("report", report_generator)
)

result = analysis_workflow.run("売上データを分析してレポートを作成")
```

### コードレビューワークフロー

```python
from strands import Agent
from strands.multiagent import Workflow

# レビューステップ
syntax_checker = Agent(
    name="syntax_checker",
    system_prompt="構文エラーと基本的な問題をチェックします。"
)

style_reviewer = Agent(
    name="style_reviewer",
    system_prompt="コーディングスタイルとベストプラクティスを確認します。"
)

security_reviewer = Agent(
    name="security_reviewer",
    system_prompt="セキュリティ上の問題を特定します。"
)

performance_reviewer = Agent(
    name="performance_reviewer",
    system_prompt="パフォーマンスの問題を特定します。"
)

summary_generator = Agent(
    name="summary_generator",
    system_prompt="すべてのレビュー結果をまとめます。"
)

# Workflow
review_workflow = (
    Workflow()
    .add_step("syntax", syntax_checker)
    .add_step("style", style_reviewer)
    .add_step("security", security_reviewer)
    .add_step("performance", performance_reviewer)
    .add_step("summary", summary_generator)
)

result = review_workflow.run("このプルリクエストをレビューしてください")
```

### 前処理を含むワークフロー

```python
from strands import Agent
from strands.multiagent import Workflow

workflow = Workflow()

# 前処理関数
def preprocess(state):
    text = state.get("input", "")
    return {
        **state,
        "cleaned_input": text.strip().lower(),
        "word_count": len(text.split())
    }

# メイン処理 Agent
processor = Agent(
    name="processor",
    system_prompt="前処理されたデータを処理します。"
)

# 後処理関数
def postprocess(state):
    result = state.get("result", "")
    return {
        **state,
        "final_output": result,
        "processed": True
    }

# Workflow に追加
workflow.add_step("preprocess", preprocess)
workflow.add_step("process", processor)
workflow.add_step("postprocess", postprocess)

result = workflow.run("  処理するテキスト  ")
```

## ベストプラクティス

### 1. 適切なステップ分割

```python
# 良い例: 各ステップが明確な責任を持つ
workflow = (
    Workflow()
    .add_step("validate_input", validator)
    .add_step("transform_data", transformer)
    .add_step("process_data", processor)
    .add_step("format_output", formatter)
)

# 悪い例: ステップが大きすぎる
workflow = Workflow().add_step("do_everything", giant_agent)
```

### 2. エラーハンドリング

```python
from strands.multiagent import Workflow

workflow = Workflow()

def safe_step(state):
    try:
        # 処理
        return {**state, "success": True}
    except Exception as e:
        return {**state, "success": False, "error": str(e)}

workflow.add_step("safe_processing", safe_step)
```

### 3. 状態の管理

```python
# 各ステップで状態を明確に更新
def step_function(state):
    # 入力を取得
    input_data = state.get("input")
    
    # 処理
    output = process(input_data)
    
    # 状態を更新して返す
    return {
        **state,
        "step_result": output,
        "step_completed": True
    }
```
