# Swarm

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/multi-agent/swarm/

## 目次

- [概要](#概要)
- [Swarm の仕組み](#swarm-の仕組み)
- [基本的な使用法](#基本的な使用法)
- [ハンドオフ](#ハンドオフ)
- [使用例](#使用例)
- [ベストプラクティス](#ベストプラクティス)

---

## 概要

Swarm は、複数の Agent 間でタスクを動的に委任するためのパターン。Agent は会話中に他の Agent に制御を「ハンドオフ」でき、それぞれの Agent が特定の役割を担当する。

Swarm パターンの主な特徴：

- **動的なハンドオフ**: Agent が自律的に他の Agent に制御を移す
- **状態の共有**: 会話履歴と状態が Agent 間で共有される
- **柔軟な構造**: Agent のネットワークを自由に構成できる

## Swarm の仕組み

Swarm では、各 Agent が以下の能力を持つ：

1. **タスク処理**: 自身の専門領域のタスクを処理
2. **ハンドオフ判断**: 他の Agent にハンドオフすべきかを判断
3. **制御の移譲**: 適切な Agent に制御を移譲

```
ユーザー → Agent A → [処理] → ハンドオフ → Agent B → [処理] → ハンドオフ → Agent C → 応答
```

## 基本的な使用法

### Swarm の作成

```python
from strands import Agent
from strands.multiagent import Swarm

# 専門家 Agent を定義
sales_agent = Agent(
    name="sales",
    system_prompt="""
    あなたは販売担当です。製品に関する質問に答え、購入を支援します。
    技術的な質問は support にハンドオフしてください。
    請求に関する質問は billing にハンドオフしてください。
    """
)

support_agent = Agent(
    name="support",
    system_prompt="""
    あなたは技術サポート担当です。技術的な問題を解決します。
    製品購入に関する質問は sales にハンドオフしてください。
    請求に関する質問は billing にハンドオフしてください。
    """
)

billing_agent = Agent(
    name="billing",
    system_prompt="""
    あなたは請求担当です。請求、支払い、返金に関する質問に対応します。
    製品に関する質問は sales にハンドオフしてください。
    技術的な質問は support にハンドオフしてください。
    """
)

# Swarm を作成
swarm = Swarm(
    agents=[sales_agent, support_agent, billing_agent],
    default_agent="sales"  # 最初に応答する Agent
)

# 実行
response = swarm.run("製品の技術仕様について教えてください")
```

## ハンドオフ

### 自動ハンドオフ

Swarm は Agent の応答を分析し、適切なハンドオフを自動的に検出する：

```python
from strands import Agent
from strands.multiagent import Swarm

# Agent はシステムプロンプトでハンドオフのルールを認識
receptionist = Agent(
    name="receptionist",
    system_prompt="""
    あなたは受付担当です。
    - 予約に関する質問 → reservations にハンドオフ
    - 苦情 → complaints にハンドオフ
    - 一般的な質問 → 自分で対応
    """
)

swarm = Swarm(agents=[receptionist, reservations, complaints])
```

### 明示的なハンドオフ

ハンドオフを明示的に制御することもできる：

```python
from strands import Agent, tool
from strands.multiagent import Swarm, handoff

# ハンドオフ関数を定義
@tool
def transfer_to_support() -> str:
    """技術サポートに転送します"""
    return handoff("support")

@tool
def transfer_to_billing() -> str:
    """請求担当に転送します"""
    return handoff("billing")

sales_agent = Agent(
    name="sales",
    tools=[transfer_to_support, transfer_to_billing],
    system_prompt="販売担当です。必要に応じて他の部門に転送します。"
)
```

## 使用例

### カスタマーサポート Swarm

```python
from strands import Agent
from strands.multiagent import Swarm

# 一次対応 Agent
triage_agent = Agent(
    name="triage",
    system_prompt="""
    あなたは一次対応担当です。
    顧客の問い合わせを分類し、適切な担当にルーティングします。
    
    ルーティングルール：
    - 製品の使い方 → how_to
    - 不具合報告 → bug_support  
    - 機能要望 → feature_request
    - アカウント問題 → account_support
    """
)

# 専門担当 Agent
how_to_agent = Agent(
    name="how_to",
    system_prompt="製品の使い方を説明する担当です。"
)

bug_support_agent = Agent(
    name="bug_support",
    system_prompt="不具合の調査と解決を行う担当です。"
)

feature_request_agent = Agent(
    name="feature_request",
    system_prompt="機能要望を受け付け、記録する担当です。"
)

account_support_agent = Agent(
    name="account_support",
    system_prompt="アカウントに関する問題を解決する担当です。"
)

# Swarm を構成
support_swarm = Swarm(
    agents=[
        triage_agent,
        how_to_agent,
        bug_support_agent,
        feature_request_agent,
        account_support_agent
    ],
    default_agent="triage"
)

# 実行
response = support_swarm.run("ログインできなくなりました")
```

### 営業チーム Swarm

```python
from strands import Agent
from strands.multiagent import Swarm

# リードクォリフィケーション
lead_qualifier = Agent(
    name="lead_qualifier",
    system_prompt="""
    見込み客を評価し、適格性を判断します。
    - エンタープライズ顧客 → enterprise_sales
    - 中小企業顧客 → smb_sales
    - 個人顧客 → consumer_sales
    """
)

enterprise_sales = Agent(
    name="enterprise_sales",
    system_prompt="エンタープライズ顧客向けの営業を担当します。"
)

smb_sales = Agent(
    name="smb_sales",
    system_prompt="中小企業向けの営業を担当します。"
)

consumer_sales = Agent(
    name="consumer_sales",
    system_prompt="個人顧客向けの営業を担当します。"
)

sales_swarm = Swarm(
    agents=[lead_qualifier, enterprise_sales, smb_sales, consumer_sales],
    default_agent="lead_qualifier"
)
```

### 状態を共有する Swarm

```python
from strands import Agent
from strands.multiagent import Swarm

# 共有状態を使用
def create_swarm_with_state():
    researcher = Agent(
        name="researcher",
        system_prompt="調査を行い、情報を収集します。結果は共有状態に保存してください。"
    )
    
    analyzer = Agent(
        name="analyzer",
        system_prompt="共有状態の情報を分析し、洞察を提供します。"
    )
    
    reporter = Agent(
        name="reporter",
        system_prompt="分析結果を基にレポートを作成します。"
    )
    
    swarm = Swarm(
        agents=[researcher, analyzer, reporter],
        default_agent="researcher"
    )
    
    return swarm

swarm = create_swarm_with_state()
response = swarm.run("市場動向を調査してレポートを作成してください")
```

## ベストプラクティス

### 1. 明確なハンドオフ条件

各 Agent のハンドオフ条件を明確に定義する：

```python
agent = Agent(
    name="support",
    system_prompt="""
    技術サポート担当です。
    
    ハンドオフ条件：
    - 「請求」「支払い」「返金」に関する質問 → billing へハンドオフ
    - 「購入」「価格」「見積もり」に関する質問 → sales へハンドオフ
    - 技術的な問題 → 自分で対応
    """
)
```

### 2. 無限ループの防止

Agent 間のハンドオフが無限ループにならないように注意：

```python
# 各 Agent にループ防止のルールを追加
agent = Agent(
    system_prompt="""
    ...
    注意: 同じ Agent から戻ってきた場合は、自分で解決を試みてください。
    3回以上のハンドオフは避けてください。
    """
)
```

### 3. 適切な粒度

Agent の数と責任範囲を適切に設定する：

- 少なすぎる Agent: 各 Agent の責任が大きすぎる
- 多すぎる Agent: ハンドオフが複雑になりすぎる

### 4. フォールバック処理

どの Agent も対応できない場合のフォールバックを用意：

```python
fallback_agent = Agent(
    name="fallback",
    system_prompt="""
    他の Agent が対応できない場合の最終対応を行います。
    - ユーザーに状況を説明
    - 人間のオペレーターへのエスカレーションを提案
    """
)
```
