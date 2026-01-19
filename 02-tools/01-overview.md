# Tools Overview

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/

## 目次

- [Agent への Tool の追加](#agent-への-tool-の追加)
- [ファイルからの Tool の読み込み](#ファイルからの-tool-の読み込み)
- [Tool の使用](#tool-の使用)
- [Tool Executor](#tool-executor)
- [Tool の構築と読み込み](#tool-の構築と読み込み)
- [Tool 設計のベストプラクティス](#tool-設計のベストプラクティス)

---

Tool は Agent の機能を拡張するための主要なメカニズムであり、単純なテキスト生成を超えたアクションを実行できるようにする。Tool により、Agent は外部システムとやり取りし、データにアクセスし、環境を操作できる。

Strands Agents Tools は、Agent が使用できる強力な Tool セットを提供するコミュニティ主導のプロジェクト。詳細については [Strands Agents Tools](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/community-tools-package/) を参照。

## Agent への Tool の追加

Tool は初期化時または実行時に Agent に渡され、Agent のライフサイクル全体で利用可能になる。読み込まれると、Agent はユーザーのリクエストに応じてこれらの Tool を使用できる：

```python
from strands import Agent
from strands_tools import calculator, file_read, shell

# Agent に Tool を追加
agent = Agent(
    tools=[calculator, file_read, shell]
)

# Agent は自動的に calculator ツールを使用するタイミングを判断
agent("42 ^ 9 は何ですか")
print("\n\n")  # 改行を出力

# Agent は必要に応じて shell と file reader ツールを使用
agent("このディレクトリの 1 つのファイルの内容を表示してください")
```

Agent に読み込まれている Tool を確認できる：

```python
print(agent.tool_names)
print(agent.tool_registry.get_all_tools_config())
```

## ファイルからの Tool の読み込み

Tool は初期化時にファイルパスを渡すことで Agent に読み込むこともできる：

```python
agent = Agent(tools=["/path/to/my_tool.py"])
```

### Tool の自動読み込みとリロード

現在の作業ディレクトリの `./tools/` に配置された Tool は、Agent の初期化時に自動的に読み込み、変更時に自動的にリロードできる。これは Tool の開発とデバッグ時に非常に便利：Tool コードを変更するだけで、その Tool を使用している Agent は最新の変更を使用するためにリロードする！

`./tools/` ディレクトリからの自動読み込みとリロードはデフォルトで無効。この動作を有効にするには、`Agent` の初期化時に `load_tools_from_directory=True` を設定：

```python
from strands import Agent

agent = Agent(load_tools_from_directory=True)
```

> **Warning**: 自動 Tool 読み込みを有効にすると、`./tools/` ディレクトリに配置された Python ファイルは Agent によって実行される。共有責任モデルの下では、Tool 読み込みディレクトリには安全で信頼できるコードのみが書き込まれるようにするのはあなたの責任であり、Agent はそこに見つかった Tool を自動的に取得して実行する。

## Tool の使用

Tool は 2 つの主要な方法で呼び出せる。

Agent は会話履歴の一部として Tool 呼び出しとその結果のコンテキストを持つ。詳細については [Using State in Tools](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/state/#using-state-in-tools) を参照。

### 自然言語による呼び出し

Agent が Tool を使用する最も一般的な方法は、自然言語リクエストを通じて。Agent はユーザーの入力に基づいて、いつ、どのように Tool を呼び出すかを判断する：

```python
# Agent はリクエストに基づいて Tool を使用するタイミングを判断
agent("/path/to/file.txt のファイルを読んでください")
```

### 直接メソッド呼び出し

自然言語呼び出しに加えて、Tool をプログラムで呼び出すこともできる。Agent に追加されたすべての Tool は、Agent オブジェクトで直接アクセスできるメソッドになる：

```python
# Tool をメソッドとして直接呼び出す
result = agent.tool.file_read(path="/path/to/file.txt", mode="view")
```

Tool をメソッドとして直接呼び出す場合、常にキーワード引数を使用する - 位置引数はサポートされて**いない**：

```python
# これは動作しない - 位置引数はサポートされていない
result = agent.tool.file_read("/path/to/file.txt", "view")  # ❌ これはしないこと
```

Tool 名にハイフンが含まれている場合、代わりにアンダースコアを使用して Tool を呼び出せる：

```python
# "read-all" という名前の Tool を直接呼び出す
result = agent.tool.read_all(path="/path/to/file.txt")
```

## Tool Executor

モデルが複数の Tool リクエストを返した場合、それらを同時に実行するか順次実行するかを制御できる。

Agent はデフォルトで同時実行を使用するが、順序が重要な場合は順次実行を指定できる：

```python
from strands import Agent
from strands.tools.executors import SequentialToolExecutor

# 同時実行（デフォルト）
agent = Agent(tools=[weather_tool, time_tool])
agent("ニューヨークの天気と時間は何ですか？")

# 順次実行
agent = Agent(
    tool_executor=SequentialToolExecutor(),
    tools=[screenshot_tool, email_tool]
)
agent("スクリーンショットを撮って友達にメールしてください")
```

詳細については [Tool Executors](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/executors/) を参照。

## Tool の構築と読み込み

### 1. カスタム Tool

Strands SDK の Tool インターフェースを使用して独自の Tool を構築する。

#### 関数ベースの Tool

`@tool` デコレーターを使用して、任意の Python 関数を Tool として定義する。デコレートされた Tool はコードベースのどこにでも配置でき、Agent の Tool リストにインポートできる。

```python
import asyncio
from strands import Agent, tool

@tool
def get_user_location() -> str:
    """ユーザーの位置を取得。"""
    # ユーザー位置検索ロジックをここに実装
    return "Seattle, USA"

@tool
def weather(location: str) -> str:
    """場所の天気情報を取得。
    
    Args:
        location: 都市または場所名
    """
    # 天気検索ロジックをここに実装
    return f"{location} の天気: 晴れ、72°F"

@tool
async def call_api() -> str:
    """API を非同期で呼び出す。
    Strands はすべての async Tool を同時に呼び出す。
    """
    await asyncio.sleep(5)  # シミュレートされた API 呼び出し
    return "API 結果"

def basic_example():
    agent = Agent(tools=[get_user_location, weather])
    agent("私の位置の天気はどうですか？")

async def async_example():
    agent = Agent(tools=[call_api])
    await agent.invoke_async("API を呼び出せますか？")

def main():
    basic_example()
    asyncio.run(async_example())
```

#### モジュールベースの Tool

Tool モジュールは、デコレーターパターンを使用しない単一の Tool も提供できる。代わりに `TOOL_SPEC` 変数と Tool 名に一致する関数を定義する。

**weather.py:**

```python
from typing import Any
from strands.types.tools import ToolResult, ToolUse

TOOL_SPEC = {
    "name": "weather",
    "description": "場所の天気情報を取得",
    "inputSchema": {
        "json": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "都市または場所名"
                }
            },
            "required": ["location"]
        }
    }
}

# 関数名は Tool 名と一致する必要がある
def weather(tool: ToolUse, **kwargs: Any) -> ToolResult:
    tool_use_id = tool["toolUseId"]
    location = tool["input"]["location"]
    
    # 天気検索ロジックをここに実装
    weather_info = f"{location} の天気: 晴れ、72°F"
    
    return {
        "toolUseId": tool_use_id,
        "status": "success",
        "content": [{"text": weather_info}]
    }
```

**agent.py:**

```python
from strands import Agent
import get_user_location
import weather

# Python モジュールインポートを通じて Agent に Tool を追加
agent = Agent(tools=[get_user_location, weather])

# カスタム Tool で Agent を使用
agent("私の位置の天気はどうですか？")
```

Tool モジュールはファイルパスを指定して読み込むこともできる：

```python
from strands import Agent

# ファイルパス文字列を通じて Agent に Tool を追加
agent = Agent(tools=["./get_user_location.py", "./weather.py"])
agent("私の位置の天気はどうですか？")
```

カスタム Python Tool の構築の詳細については [Creating Custom Tools](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/custom-tools/) を参照。

### 2. ベンダー提供の Tool

すぐに始められるように、Python と TypeScript の両方で事前構築された Tool が利用可能。

**コミュニティ Tools パッケージ**

Python では、Strands は開発用の事前構築 Tool を備えた[コミュニティサポートの Tools パッケージ](https://github.com/strands-agents/tools/blob/main)を提供：

```python
from strands import Agent
from strands_tools import calculator, file_read, shell

agent = Agent(tools=[calculator, file_read, shell])
```

利用可能な Tool の完全なリストについては [Community Tools Package](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/community-tools-package/) を参照。

### 3. Model Context Protocol (MCP) Tool

[Model Context Protocol (MCP)](https://modelcontextprotocol.io) は、異なるシステム間で Tool を公開および使用するための標準化された方法を提供する。このアプローチは、複数の Agent またはアプリケーション間で共有できる再利用可能な Tool コレクションを作成するのに理想的。

```python
from mcp.client.sse import sse_client
from strands import Agent
from strands.tools.mcp import MCPClient

# SSE トランスポートを使用して MCP サーバーに接続
sse_mcp_client = MCPClient(lambda: sse_client("http://localhost:8000/sse"))

# MCP Tool で Agent を作成
with sse_mcp_client:
    # MCP サーバーから Tool を取得
    tools = sse_mcp_client.list_tools_sync()
    
    # MCP サーバーの Tool で Agent を作成
    agent = Agent(tools=tools)
    
    # MCP Tool で Agent を使用
    agent("144 の平方根を計算してください")
```

MCP Tool の使用の詳細については [MCP Tools](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/mcp-tools/) を参照。

## Tool 設計のベストプラクティス

### 効果的な Tool 説明

言語モデルは、Tool をいつ、どのように使用するかを判断するために、Tool の説明に大きく依存する。適切に作成された説明は、Tool の使用精度を大幅に向上させる。

**良い Tool 説明には以下を含める：**

- Tool の目的と機能の明確な説明
- Tool をいつ使用すべきかの指定
- 受け付けるパラメータとそのフォーマットの詳細
- 期待される出力フォーマットの説明
- 制限や制約の注記

**適切に説明された Tool の例：**

```python
@tool
def search_database(query: str, max_results: int = 10) -> list:
    """
    クエリ文字列に一致するアイテムを製品データベースで検索。
    
    キーワード、製品名、またはカテゴリに基づいて詳細な製品情報を
    見つける必要がある場合にこの Tool を使用してください。
    検索は大文字小文字を区別せず、タイプミスや検索語のバリエーションを
    処理するためのファジーマッチングをサポートしています。
    
    この Tool は企業の製品カタログデータベースに接続し、すべての
    製品フィールドにわたってセマンティック検索を実行し、利用可能な
    すべての製品メタデータを含む包括的な結果を提供します。
    
    レスポンス例:
    [
        {
            "id": "P12345",
            "name": "Ultra Comfort Running Shoes",
            "description": "軽量ランニングシューズ...",
            "price": 89.99,
            "category": ["Footwear", "Athletic", "Running"]
        },
        ...
    ]
    
    注記:
    - この Tool は製品カタログのみを検索し、在庫や可用性情報は提供しない
    - 結果はパフォーマンス向上のため 15 分間キャッシュされる
    - 検索インデックスは 6 時間ごとに更新されるため、非常に最近の製品は表示されない可能性がある
    - リアルタイムの在庫状況には、別の在庫確認 Tool を使用
    
    Args:
        query: 検索文字列（製品名、カテゴリ、またはキーワード）
               例: "red running shoes" または "smartphone charger"
        max_results: 返す結果の最大数（デフォルト: 10、範囲: 1-100）
                    完全一致が期待される場合は、より高速な応答のために低い値を使用
    
    Returns:
        一致する製品レコードのリスト、各レコードには以下を含む:
        - id: 一意の製品識別子（文字列）
        - name: 製品名（文字列）
        - description: 詳細な製品説明（文字列）
        - price: 現在の価格（USD、浮動小数点）
        - category: 製品カテゴリ階層（リスト）
    """
    # 実装
    pass
```
