# Experimental 機能 概要

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/experimental/

## 目次

- [概要](#概要)
- [実験的機能とは](#実験的機能とは)
- [利用可能な実験的機能](#利用可能な実験的機能)
- [使用上の注意](#使用上の注意)
- [フィードバック](#フィードバック)

---

## 概要

Experimental（実験的）機能は、開発中または評価段階にある新しい機能。これらの機能は将来的に変更または削除される可能性があるが、次世代の Agent 開発を先取りして体験できる。

## 実験的機能とは

### 特徴

実験的機能には以下の特徴がある：

1. **プレビュー段階**: 本番環境での使用は推奨されない
2. **API の変更可能性**: インターフェースが変更される可能性がある
3. **限定的なサポート**: ドキュメントやサポートが限定的
4. **フィードバック歓迎**: ユーザーからの意見を積極的に募集

### 有効化方法

実験的機能は明示的に有効化する必要がある：

```python
from strands import Agent
from strands.experimental import enable_experimental

# 実験的機能を有効化
enable_experimental("feature_name")

# または Agent 作成時に指定
agent = Agent(
    system_prompt="You are a helpful assistant.",
    experimental_features=["feature_name"]
)
```

### 環境変数での有効化

```bash
export STRANDS_EXPERIMENTAL_FEATURES="feature1,feature2"
```

```python
import os
from strands import Agent

# 環境変数から自動的に読み込まれる
agent = Agent(system_prompt="...")
```

## 利用可能な実験的機能

### 1. Vibe Coding

自然言語でコードを生成・修正する機能：

```python
from strands.experimental import VibeCoder

coder = VibeCoder(
    language="python",
    style="functional"
)

code = await coder.generate(
    "ユーザー認証を行う関数を作成して。"
    "JWT トークンを使用し、有効期限は24時間。"
)
```

詳細: [Vibe Coding](./02-vibe-coding.md)

### 2. Auto-Tool Generation

Agent が必要に応じて自動的にツールを生成：

```python
from strands import Agent
from strands.experimental import AutoToolGenerator

generator = AutoToolGenerator()

agent = Agent(
    system_prompt="You are a helpful assistant.",
    experimental_features=["auto_tool_generation"],
    tool_generator=generator
)

# Agent が「天気を調べる」と言われたとき、
# 必要なツールを自動生成
result = agent.run("今日の東京の天気を調べて")
```

### 3. Memory System

長期記憶と短期記憶を管理するシステム：

```python
from strands import Agent
from strands.experimental import MemorySystem

memory = MemorySystem(
    short_term_capacity=100,
    long_term_storage="redis://localhost:6379"
)

agent = Agent(
    system_prompt="You are a helpful assistant.",
    experimental_features=["memory"],
    memory=memory
)

# 会話を通じて情報を記憶
agent.run("私の名前は田中です")
# 後のセッションでも記憶を保持
agent.run("私の名前を覚えていますか？")  # "田中さんですね"
```

### 4. Reasoning Chain

思考プロセスを可視化し、推論を改善：

```python
from strands import Agent
from strands.experimental import ReasoningChain

agent = Agent(
    system_prompt="You are a helpful assistant.",
    experimental_features=["reasoning_chain"]
)

result = agent.run(
    "この数学の問題を解いて: 2x + 5 = 15",
    show_reasoning=True
)

# 推論プロセスを表示
for step in result.reasoning_steps:
    print(f"Step {step.number}: {step.description}")
    print(f"  Result: {step.result}")
```

### 5. Multi-Modal Input

画像、音声などの複数のモダリティを処理：

```python
from strands import Agent
from strands.experimental import MultiModalProcessor

processor = MultiModalProcessor()

agent = Agent(
    system_prompt="You are a helpful assistant.",
    experimental_features=["multi_modal"],
    processor=processor
)

# 画像を含む入力
result = agent.run(
    "この画像に何が写っていますか？",
    attachments=[
        {"type": "image", "path": "photo.jpg"}
    ]
)
```

## 使用上の注意

### 本番環境での使用

実験的機能を本番環境で使用する場合は以下に注意：

```python
import warnings
from strands.experimental import enable_experimental

# 警告を表示
warnings.filterwarnings("always", category=ExperimentalWarning)

# 明示的に本番使用を許可
enable_experimental(
    "feature_name",
    allow_production=True,
    acknowledge_risks=True
)
```

### バージョン固定

実験的機能を使用する場合は、バージョンを固定することを推奨：

```
# requirements.txt
strands==0.5.0  # 実験的機能のバージョンを固定
```

### フォールバック実装

実験的機能が失敗した場合のフォールバックを用意：

```python
from strands import Agent
from strands.experimental import ExperimentalFeatureError

try:
    agent = Agent(
        experimental_features=["new_feature"]
    )
    result = agent.run(prompt)
except ExperimentalFeatureError:
    # フォールバック: 通常の Agent を使用
    fallback_agent = Agent()
    result = fallback_agent.run(prompt)
```

## フィードバック

### バグレポート

実験的機能でバグを発見した場合：

```python
from strands.experimental import report_issue

report_issue(
    feature="feature_name",
    description="バグの説明",
    reproduction_steps="再現手順",
    expected_behavior="期待される動作",
    actual_behavior="実際の動作"
)
```

### 機能リクエスト

新しい機能のリクエスト：

```python
from strands.experimental import feature_request

feature_request(
    title="機能のタイトル",
    description="機能の詳細な説明",
    use_case="ユースケース"
)
```

### コミュニティ

- GitHub Discussions: 議論や質問
- Discord: リアルタイムでの交流
- Issue Tracker: バグ報告と機能リクエスト
