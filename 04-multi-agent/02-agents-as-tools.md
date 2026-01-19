# Agents as Tools

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/agents-as-tools/

## 目次

- [概要](#概要)
- [基本的な使用法](#基本的な使用法)
- [Agent を Tool に変換](#agent-を-tool-に変換)
- [使用例](#使用例)
- [ベストプラクティス](#ベストプラクティス)

---

## 概要

Agents as Tools は、Agent を別の Agent の Tool として使用できるパターン。これにより、Agent 間の階層的な関係を構築し、複雑なタスクを専門的な sub-agent に委任できる。

このパターンの主な利点：

- **専門化**: 各 Agent が特定のタスクに特化できる
- **再利用性**: 既存の Agent を新しいコンテキストで再利用できる
- **モジュール性**: Agent を独立して開発・テストできる
- **シンプルさ**: 複雑なオーケストレーションなしで Agent を組み合わせられる

## 基本的な使用法

Agent を Tool として使用する最もシンプルな方法：

```python
from strands import Agent

# 専門家 Agent を作成
researcher = Agent(
    name="researcher",
    system_prompt="あなたはリサーチの専門家です。情報を調査して詳細なレポートを提供します。"
)

writer = Agent(
    name="writer",
    system_prompt="あなたは文章作成の専門家です。情報を基に読みやすい文章を作成します。"
)

# メイン Agent に専門家を Tool として追加
main_agent = Agent(
    tools=[researcher, writer],
    system_prompt="""
    あなたはプロジェクトマネージャーです。
    タスクに応じて適切な専門家に仕事を委任してください。
    - 調査が必要な場合: researcher を使用
    - 文章作成が必要な場合: writer を使用
    """
)

# 実行
response = main_agent("AIの最新トレンドについて調査し、ブログ記事を作成してください")
```

## Agent を Tool に変換

### 自動変換

Agent を `tools` リストに直接追加すると、自動的に Tool として扱われる：

```python
from strands import Agent

specialist = Agent(
    name="math_specialist",
    system_prompt="数学の問題を解決する専門家です。"
)

# specialist が自動的に Tool に変換される
coordinator = Agent(
    tools=[specialist]
)
```

### 明示的な変換

より細かい制御が必要な場合、`agent_tool` デコレーターを使用：

```python
from strands import Agent, agent_tool

# Agent をカスタム設定で Tool に変換
@agent_tool(
    name="data_analyzer",
    description="データを分析して洞察を提供します"
)
def create_analyzer():
    return Agent(
        system_prompt="データ分析の専門家です。統計分析と可視化を行います。"
    )

analyzer = create_analyzer()

main_agent = Agent(tools=[analyzer])
```

## 使用例

### 階層的な Agent 構造

```python
from strands import Agent

# レベル 2: 専門家 Agent
code_reviewer = Agent(
    name="code_reviewer",
    system_prompt="コードレビューを行い、改善点を提案します。"
)

security_expert = Agent(
    name="security_expert",
    system_prompt="セキュリティの観点からコードを分析します。"
)

performance_expert = Agent(
    name="performance_expert",
    system_prompt="パフォーマンスの観点からコードを分析します。"
)

# レベル 1: 中間コーディネーター
code_quality_coordinator = Agent(
    name="code_quality_coordinator",
    tools=[code_reviewer, security_expert, performance_expert],
    system_prompt="""
    コード品質を評価するコーディネーターです。
    必要に応じて専門家に相談してください。
    """
)

# レベル 0: メイン Agent
project_lead = Agent(
    tools=[code_quality_coordinator],
    system_prompt="プロジェクトリーダーとして、コード品質を管理します。"
)

response = project_lead("このプルリクエストをレビューしてください")
```

### 専門家チーム

```python
from strands import Agent

# 各分野の専門家
frontend_dev = Agent(
    name="frontend_developer",
    system_prompt="フロントエンド開発の専門家です。React、Vue、Angular に精通しています。"
)

backend_dev = Agent(
    name="backend_developer",
    system_prompt="バックエンド開発の専門家です。Python、Node.js、Go に精通しています。"
)

devops_engineer = Agent(
    name="devops_engineer",
    system_prompt="DevOps エンジニアです。CI/CD、Docker、Kubernetes に精通しています。"
)

designer = Agent(
    name="ui_designer",
    system_prompt="UI/UX デザイナーです。ユーザー体験の設計を行います。"
)

# テックリード
tech_lead = Agent(
    tools=[frontend_dev, backend_dev, devops_engineer, designer],
    system_prompt="""
    あなたはテックリードです。
    プロジェクトの技術的な側面を管理し、適切なチームメンバーにタスクを割り当てます。
    
    各メンバーの専門分野：
    - frontend_developer: フロントエンド開発
    - backend_developer: バックエンド開発
    - devops_engineer: インフラとデプロイメント
    - ui_designer: デザインと UX
    """
)

response = tech_lead("新しい機能のフルスタック実装計画を作成してください")
```

### 動的な Agent 選択

```python
from strands import Agent

def create_specialist_agent(domain: str) -> Agent:
    """ドメインに応じた専門家 Agent を動的に作成"""
    prompts = {
        "finance": "財務分析の専門家です。",
        "marketing": "マーケティング戦略の専門家です。",
        "operations": "オペレーション最適化の専門家です。",
    }
    return Agent(
        name=f"{domain}_specialist",
        system_prompt=prompts.get(domain, "一般的なビジネスアドバイザーです。")
    )

# 必要に応じて専門家を作成
specialists = [
    create_specialist_agent("finance"),
    create_specialist_agent("marketing"),
    create_specialist_agent("operations")
]

business_consultant = Agent(
    tools=specialists,
    system_prompt="ビジネスコンサルタントとして、適切な専門家に相談します。"
)
```

## ベストプラクティス

### 1. 明確な役割定義

各 Agent に明確で具体的な役割を定義する：

```python
# 良い例
agent = Agent(
    name="contract_reviewer",
    system_prompt="契約書のレビューを専門とします。法的リスクを特定し、改善提案を行います。"
)

# 悪い例
agent = Agent(
    name="helper",
    system_prompt="いろいろなことを手伝います。"
)
```

### 2. 適切な粒度

Agent の責任範囲を適切に設定する：

```python
# 適切な粒度
data_cleaner = Agent(name="data_cleaner", system_prompt="データクレンジングを行います。")
data_analyzer = Agent(name="data_analyzer", system_prompt="データ分析を行います。")
report_generator = Agent(name="report_generator", system_prompt="レポートを生成します。")

# 粒度が粗すぎる
do_everything = Agent(name="do_everything", system_prompt="すべてを行います。")
```

### 3. エラーハンドリング

sub-agent のエラーを適切に処理する：

```python
from strands import Agent

main_agent = Agent(
    tools=[specialist_agent],
    system_prompt="""
    専門家に相談する際、エラーが発生した場合は：
    1. エラーの内容を報告する
    2. 可能であれば代替手段を提案する
    3. 必要に応じてユーザーに確認を求める
    """
)
```

### 4. コンテキストの伝達

必要な情報を sub-agent に適切に伝達する：

```python
main_agent = Agent(
    tools=[sub_agent],
    system_prompt="""
    sub-agent にタスクを委任する際は、以下の情報を含めてください：
    - タスクの目的と背景
    - 必要な入力データ
    - 期待される出力形式
    - 制約条件や注意点
    """
)
```
