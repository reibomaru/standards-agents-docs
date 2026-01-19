# Model Context Protocol (MCP) Tools

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/mcp-tools/

## 目次

- [クイックスタート](#クイックスタート)
- [統合アプローチ](#統合アプローチ)
- [トランスポートオプション](#トランスポートオプション)
- [複数の MCP サーバーの使用](#複数の-mcp-サーバーの使用)
- [クライアント設定](#クライアント設定)
- [直接 Tool 呼び出し](#直接-tool-呼び出し)
- [MCP サーバーの実装](#mcp-サーバーの実装)
- [高度な使用法](#高度な使用法)
- [ベストプラクティス](#ベストプラクティス)
- [トラブルシューティング](#トラブルシューティング)

---

[Model Context Protocol (MCP)](https://modelcontextprotocol.io) は、アプリケーションが大規模言語モデルにコンテキストを提供する方法を標準化するオープンプロトコルである。Strands Agents は MCP と統合して、外部の Tool やサービスを通じて Agent の機能を拡張する。

MCP により、Agent と追加の Tool を提供する MCP サーバー間の通信が可能になる。Strands には、Python と TypeScript の両方で MCP サーバーに接続して Tool を使用するための組み込みサポートが含まれている。

## クイックスタート

```python
from mcp import stdio_client, StdioServerParameters
from strands import Agent
from strands.tools.mcp import MCPClient

# stdio トランスポートで MCP クライアントを作成
mcp_client = MCPClient(lambda: stdio_client(
    StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )
))

# ライフサイクル管理にコンテキストマネージャーを使用
with mcp_client:
    tools = mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
    agent("AWS Lambda とは何ですか？")
```

## 統合アプローチ

### 手動コンテキスト管理

Python では、MCP 接続のライフサイクルを管理するために `with` ステートメントを使用した明示的なコンテキスト管理が必要：

```python
from mcp import stdio_client, StdioServerParameters
from strands import Agent
from strands.tools.mcp import MCPClient

mcp_client = MCPClient(lambda: stdio_client(
    StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )
))

# 手動ライフサイクル管理
with mcp_client:
    tools = mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
    agent("AWS Lambda とは何ですか？")  # コンテキスト内で使用する必要がある
```

このアプローチは MCP セッションのライフサイクルを直接制御できるが、接続エラーを避けるために注意深い管理が必要。

### マネージド統合（実験的）

> **実験的機能**: マネージド統合機能は実験的であり、将来のバージョンで変更される可能性がある。本番アプリケーションでは、手動コンテキスト管理アプローチを使用すること。

`MCPClient` は実験的な `ToolProvider` インターフェースを実装しており、Agent コンストラクターで直接使用して自動ライフサイクル管理が可能：

```python
# 直接使用 - 接続ライフサイクルは自動的に管理される
agent = Agent(tools=[mcp_client])
response = agent("AWS Lambda とは何ですか？")
```

## トランスポートオプション

Python と TypeScript の両方で、MCP サーバーに接続するための複数のトランスポートメカニズムをサポート。

### Standard I/O (stdio)

MCP プロトコルを実装するコマンドラインツールやローカルプロセス用：

```python
from mcp import stdio_client, StdioServerParameters
from strands import Agent
from strands.tools.mcp import MCPClient

# macOS/Linux 用:
stdio_mcp_client = MCPClient(lambda: stdio_client(
    StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )
))

# Windows 用:
stdio_mcp_client = MCPClient(lambda: stdio_client(
    StdioServerParameters(
        command="uvx",
        args=[
            "--from", "awslabs.aws-documentation-mcp-server@latest",
            "awslabs.aws-documentation-mcp-server.exe"
        ]
    )
))

with stdio_mcp_client:
    tools = stdio_mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
    response = agent("AWS Lambda とは何ですか？")
```

### Streamable HTTP

Streamable HTTP トランスポートを使用する HTTP ベースの MCP サーバー用：

```python
from mcp.client.streamable_http import streamablehttp_client
from strands import Agent
from strands.tools.mcp import MCPClient

streamable_http_mcp_client = MCPClient(
    lambda: streamablehttp_client("http://localhost:8000/mcp")
)
```

#### AWS IAM

IAM 認証情報を使用した SigV4 認証を使用する AWS 上の MCP サーバーの場合、[mcp-proxy-for-aws](https://pypi.org/project/mcp-proxy-for-aws/) パッケージを使用して、AWS 認証情報管理とリクエスト署名を自動的に処理できる。

まず、パッケージをインストール：

```bash
pip install mcp-proxy-for-aws
```

他のトランスポートと同様に使用：

```python
from mcp_proxy_for_aws.client import aws_iam_streamablehttp_client
from strands.tools.mcp import MCPClient

mcp_client = MCPClient(lambda: aws_iam_streamablehttp_client(
    endpoint="https://your-service.us-east-1.amazonaws.com/mcp",
    aws_region="us-east-1",
    aws_service="bedrock-agentcore"
))
```

### Server-Sent Events (SSE)

Server-Sent Events トランスポートを使用する HTTP ベースの MCP サーバー用：

```python
from mcp.client.sse import sse_client
from strands import Agent
from strands.tools.mcp import MCPClient

sse_mcp_client = MCPClient(lambda: sse_client("http://localhost:8000/sse"))

with sse_mcp_client:
    tools = sse_mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
```

追加のプロパティ（認証など）を設定可能：

```python
import os
from mcp.client.streamable_http import streamablehttp_client
from strands.tools.mcp import MCPClient

github_mcp_client = MCPClient(
    lambda: streamablehttp_client(
        url="https://api.githubcopilot.com/mcp/",
        headers={"Authorization": f"Bearer {os.getenv('MCP_PAT')}"}
    )
)
```

## 複数の MCP サーバーの使用

複数の MCP サーバーからの Tool を単一の Agent で組み合わせる：

```python
from mcp import stdio_client, StdioServerParameters
from mcp.client.sse import sse_client
from strands import Agent
from strands.tools.mcp import MCPClient

# 複数のクライアントを作成
sse_mcp_client = MCPClient(lambda: sse_client("http://localhost:8000/sse"))
stdio_mcp_client = MCPClient(lambda: stdio_client(
    StdioServerParameters(command="python", args=["path/to/mcp_server.py"])
))

# 手動アプローチ - 明示的なコンテキスト管理
with sse_mcp_client, stdio_mcp_client:
    tools = sse_mcp_client.list_tools_sync() + stdio_mcp_client.list_tools_sync()
    agent = Agent(tools=tools)

# マネージドアプローチ（実験的）
agent = Agent(tools=[sse_mcp_client, stdio_mcp_client])
```

## クライアント設定

Python の `MCPClient` は、複数のサーバーからの Tool を管理するための Tool フィルタリングと名前プレフィックスをサポート。

### Tool フィルタリング

`tool_filters` パラメータを使用してロードする Tool を制御：

```python
from mcp import stdio_client, StdioServerParameters
from strands.tools.mcp import MCPClient
import re

# 文字列マッチング - 指定された Tool のみをロード
filtered_client = MCPClient(
    lambda: stdio_client(StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )),
    tool_filters={"allowed": ["search_documentation", "read_documentation"]}
)

# 正規表現パターン
regex_client = MCPClient(
    lambda: stdio_client(StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )),
    tool_filters={"allowed": [re.compile(r"^search_.*")]}
)

# 組み合わせフィルター - allowed を先に適用し、次に rejected を適用
combined_client = MCPClient(
    lambda: stdio_client(StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )),
    tool_filters={
        "allowed": [re.compile(r".*documentation$")],
        "rejected": ["read_documentation"]
    }
)
```

### Tool 名プレフィックス

複数の MCP サーバーを使用する際の名前衝突を防止：

```python
aws_docs_client = MCPClient(
    lambda: stdio_client(StdioServerParameters(
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"]
    )),
    prefix="aws_docs"
)

other_client = MCPClient(
    lambda: stdio_client(StdioServerParameters(
        command="uvx",
        args=["other-mcp-server@latest"]
    )),
    prefix="other"
)

# Tool は以下のように名前が付けられる: aws_docs_search_documentation, other_search など
agent = Agent(tools=[aws_docs_client, other_client])
```

## 直接 Tool 呼び出し

Tool は通常、ユーザーのリクエストに基づいて Agent によって呼び出されるが、MCP Tool は直接呼び出すこともできる：

```python
result = mcp_client.call_tool_sync(
    tool_use_id="tool-123",
    name="calculator",
    arguments={"x": 10, "y": 20}
)
print(f"結果: {result['content'][0]['text']}")
```

## MCP サーバーの実装

カスタム MCP サーバーを作成して Agent の機能を拡張できる：

```python
from mcp.server import FastMCP

# MCP サーバーを作成
mcp = FastMCP("Calculator Server")

# Tool を定義
@mcp.tool(description="計算を実行する電卓 Tool")
def calculator(x: int, y: int) -> int:
    return x + y

# SSE トランスポートでサーバーを実行
mcp.run(transport="sse")
```

MCP サーバーの実装に関する詳細については、[MCP ドキュメント](https://modelcontextprotocol.io) を参照。

## 高度な使用法

### Elicitation（情報引き出し）

MCP サーバーは、elicitation リクエストを送信してユーザーから追加情報を要求できる。これらのリクエストを処理するために elicitation コールバックを設定する：

**サーバー側 (server.py):**

```python
from mcp.server import FastMCP
from pydantic import BaseModel, Field

class ApprovalSchema(BaseModel):
    username: str = Field(description="誰が承認しますか？")

server = FastMCP("mytools")

@server.tool()
async def delete_files(paths: list[str]) -> str:
    result = await server.get_context().elicit(
        message=f"{paths} を削除しますか？",
        schema=ApprovalSchema,
    )
    if result.action != "accept":
        return f"ユーザー {result.data.username} が削除を拒否しました"
    # 削除を実行...
    return f"ユーザー {result.data.username} が削除を承認しました"

server.run()
```

**クライアント側 (client.py):**

```python
from mcp import stdio_client, StdioServerParameters
from mcp.types import ElicitResult
from strands import Agent
from strands.tools.mcp import MCPClient

async def elicitation_callback(context, params):
    print(f"ELICITATION: {params.message}")
    # ユーザー確認を取得...
    return ElicitResult(
        action="accept",
        content={"username": "myname"}
    )

client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(command="python", args=["/path/to/server.py"])
    ),
    elicitation_callback=elicitation_callback,
)

with client:
    agent = Agent(tools=client.list_tools_sync())
    result = agent("'a/b/c.txt' を削除して、承認者の名前を共有してください")
```

Elicitation に関する詳細については、[MCP 仕様](https://modelcontextprotocol.io/specification/draft/client/elicitation) を参照。

## ベストプラクティス

- **Tool の説明**: Agent が Tool をいつどのように使用するかを理解できるよう、Tool に明確な説明を提供する
- **エラーハンドリング**: Tool の実行に失敗した場合は、有益なエラーメッセージを返す
- **セキュリティ**: MCP 経由で Tool を公開する際は、特にネットワークアクセス可能なサーバーでセキュリティへの影響を考慮する
- **接続管理**: Python では、MCP 接続の適切なクリーンアップを確保するために、常にコンテキストマネージャー（`with` ステートメント）を使用する
- **タイムアウト**: 長時間実行される操作でハングするのを防ぐため、Tool 呼び出しに適切なタイムアウトを設定する

## トラブルシューティング

### MCPClientInitializationError (Python)

MCP 接続に依存する Tool は、コンテキストマネージャー内で使用する必要がある。Agent を `with` ステートメントブロック外で使用すると、操作が失敗する。

```python
# 正しい
with mcp_client:
    agent = Agent(tools=mcp_client.list_tools_sync())
    response = agent("プロンプト")  # 動作する

# 誤り
with mcp_client:
    agent = Agent(tools=mcp_client.list_tools_sync())
response = agent("プロンプト")  # 失敗 - コンテキスト外
```

### 接続失敗

MCP サーバーとの接続確立に問題がある場合に接続失敗が発生する。以下を確認：

- MCP サーバーが実行中でアクセス可能である
- ネットワーク接続が利用可能で、ファイアウォールが接続を許可している
- URL またはコマンドが正しく適切にフォーマットされている

### Tool 検出の問題

Tool が検出されない場合：

- MCP サーバーが `list_tools` メソッドを正しく実装していることを確認
- すべての Tool がサーバーに登録されていることを確認

### Tool 実行エラー

Tool の実行が失敗する場合：

- Tool の引数が期待されるスキーマと一致していることを確認
- 詳細なエラー情報についてサーバーログを確認
