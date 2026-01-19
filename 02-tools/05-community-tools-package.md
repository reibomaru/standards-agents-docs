# Community Tools Package

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/community-tools-package/

## 目次

- [概要](#概要)
- [インストール](#インストール)
- [利用可能な Tool](#利用可能な-tool)
- [使用方法](#使用方法)
- [貢献方法](#貢献方法)

---

## 概要

Strands Agents Tools は、Agent がすぐに使用できる強力な Tool セットを提供するコミュニティ主導のプロジェクト。このパッケージには、開発用の事前構築 Tool が含まれており、Agent の機能を素早く拡張できる。

GitHub リポジトリ: [strands-agents/tools](https://github.com/strands-agents/tools)

## インストール

pip を使用して strands-tools パッケージをインストール：

```bash
pip install strands-tools
```

## 利用可能な Tool

Community Tools Package には、以下のようなカテゴリの Tool が含まれている：

### ファイル操作 Tool

| Tool 名 | 説明 |
|--------|------|
| `file_read` | ファイルの内容を読み取る |
| `file_write` | ファイルに内容を書き込む |
| `file_delete` | ファイルを削除する |

### シェル・システム Tool

| Tool 名 | 説明 |
|--------|------|
| `shell` | シェルコマンドを実行する |
| `environment` | 環境変数を取得・設定する |

### ユーティリティ Tool

| Tool 名 | 説明 |
|--------|------|
| `calculator` | 数学的な計算を実行する |
| `json_parser` | JSON データを解析・操作する |

### Web・ネットワーク Tool

| Tool 名 | 説明 |
|--------|------|
| `http_request` | HTTP リクエストを送信する |
| `web_search` | Web 検索を実行する |

### データベース Tool

| Tool 名 | 説明 |
|--------|------|
| `sql_query` | SQL クエリを実行する |

## 使用方法

### 基本的な使用法

```python
from strands import Agent
from strands_tools import calculator, file_read, shell

# 複数の Tool を Agent に追加
agent = Agent(
    tools=[calculator, file_read, shell]
)

# 計算を実行
agent("42 の 9 乗は何ですか？")

# ファイルを読み取る
agent("config.json ファイルの内容を表示してください")

# シェルコマンドを実行
agent("現在のディレクトリのファイル一覧を表示してください")
```

### 特定の Tool のみをインポート

必要な Tool のみをインポートして使用することもできる：

```python
from strands import Agent
from strands_tools import calculator

agent = Agent(tools=[calculator])
agent("100 + 200 の結果を計算してください")
```

### すべての Tool をインポート

すべての利用可能な Tool をインポートすることもできる（ただし、必要な Tool のみをインポートすることを推奨）：

```python
from strands import Agent
import strands_tools

# すべての Tool を確認
print(dir(strands_tools))
```

## Tool の詳細

各 Tool は適切に文書化されており、モデルが自動的に使用方法を理解できる。Tool の詳細は以下の方法で確認できる：

```python
from strands_tools import calculator

# Tool の仕様を確認
print(calculator.TOOL_SPEC)
```

## 貢献方法

Community Tools Package はコミュニティ主導のプロジェクトであり、新しい Tool の追加や既存 Tool の改善に貢献できる。

### 新しい Tool を追加するには

1. [strands-agents/tools](https://github.com/strands-agents/tools) リポジトリをフォーク
2. 新しい Tool を作成（[Creating Custom Tools](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/custom-tools/) を参照）
3. テストを追加
4. ドキュメントを更新
5. プルリクエストを提出

### Tool 設計のガイドライン

新しい Tool を作成する際は、以下のガイドラインに従うこと：

- **明確な説明**: Tool の目的と使用方法を明確に説明する
- **適切なパラメータ**: 必要なパラメータのみを定義し、適切なデフォルト値を設定する
- **エラーハンドリング**: 適切なエラーメッセージを返す
- **セキュリティ**: 潜在的なセキュリティリスクを考慮する
- **テスト**: 包括的なテストを作成する
