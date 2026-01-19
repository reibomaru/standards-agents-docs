# Creating Custom Tools

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/custom-tools/

## 目次

- [Tool 作成の方法](#tool-作成の方法)
- [Tool 作成の例](#tool-作成の例)
- [Tool の使用とカスタマイズ](#tool-の使用とカスタマイズ)
- [クラスベースの Tool](#クラスベースの-tool)
- [Tool レスポンス形式](#tool-レスポンス形式)
- [モジュールベースの Tool（Python のみ）](#モジュールベースの-toolpython-のみ)

---

Strands でカスタム Tool を定義する方法には複数のアプローチがあり、Python と TypeScript の実装では違いがある。

## Tool 作成の方法

Python では Tool を定義する 3 つのアプローチをサポートしている：

1. **`@tool` デコレーターを使用した Python 関数**: 単純なデコレーターを追加することで、通常の Python 関数を Tool に変換。このアプローチは Python の docstring と型ヒントを活用して、Tool 仕様を自動的に生成する。

2. **`@tool` デコレーターを使用したクラスベースの Tool**: クラス内で Tool を作成して、状態を維持し、オブジェクト指向プログラミングパターンを活用。

3. **特定のフォーマットに従った Python モジュール**: Tool 仕様と一致する関数を含む Python モジュールを作成して Tool を定義。このアプローチは Tool の定義をより細かく制御でき、依存関係のない Tool の実装に便利。

## Tool 作成の例

### 基本的な例

Tool としてデコレートされた関数のシンプルな例：

```python
from strands import tool

@tool
def weather_forecast(city: str, days: int = 3) -> str:
    """都市の天気予報を取得。
    
    Args:
        city: 都市名
        days: 予報の日数
    """
    return f"{city} の {days} 日間の天気予報..."
```

デコレーターは関数の docstring から情報を抽出して Tool 仕様を作成する。最初の段落が Tool の説明になり、「Args」セクションがパラメータの説明を提供する。これらは関数の型ヒントと組み合わせて、完全な Tool 仕様を作成する。

### Tool 名、説明、スキーマのオーバーライド

デコレーターの引数として提供することで、Tool 名、説明、入力スキーマをオーバーライドできる：

```python
@tool(name="get_weather", description="指定された場所の天気予報を取得")
def weather_forecast(city: str, days: int = 3) -> str:
    """天気予報の実装関数。
    
    Args:
        city: 都市名
        days: 予報の日数
    """
    return f"{city} の {days} 日間の天気予報..."
```

### 入力スキーマのオーバーライド

自動生成されたスキーマをオーバーライドするカスタム JSON スキーマを提供できる：

```python
@tool(
    inputSchema={
        "json": {
            "type": "object",
            "properties": {
                "shape": {
                    "type": "string",
                    "enum": ["circle", "rectangle"],
                    "description": "形状のタイプ"
                },
                "radius": {"type": "number", "description": "円の半径"},
                "width": {"type": "number", "description": "長方形の幅"},
                "height": {"type": "number", "description": "長方形の高さ"}
            },
            "required": ["shape"]
        }
    }
)
def calculate_area(shape: str, radius: float = None, width: float = None, height: float = None) -> float:
    """形状の面積を計算。"""
    if shape == "circle":
        return 3.14159 * radius ** 2
    elif shape == "rectangle":
        return width * height
    return 0.0
```

## Tool の使用とカスタマイズ

### 関数ベースの Tool の読み込み

関数ベースの Tool を使用するには、Agent に渡すだけ：

```python
agent = Agent(
    tools=[weather_forecast]
)
```

### カスタム戻り値型

デフォルトでは、関数の戻り値は自動的にテキストレスポンスとしてフォーマットされる。ただし、レスポンスのフォーマットをより細かく制御する必要がある場合は、特定の構造を持つ辞書を返すことができる：

```python
@tool
def fetch_data(source_id: str) -> dict:
    """指定されたソースからデータを取得。
    
    Args:
        source_id: データソースの識別子
    """
    try:
        data = some_other_function(source_id)
        return {
            "status": "success",
            "content": [
                {
                    "json": data,
                }]
        }
    except Exception as e:
        return {
            "status": "error",
            "content": [
                {"text": f"エラー:{e}"}
            ]
        }
```

### 非同期呼び出し

関数 Tool は async としても定義できる。Strands はすべての async Tool を同時に呼び出す。

```python
import asyncio
from strands import Agent, tool

@tool
async def call_api() -> str:
    """非同期で API を呼び出す。"""
    await asyncio.sleep(5)  # シミュレートされた API 呼び出し
    return "API 結果"

async def async_example():
    agent = Agent(tools=[call_api])
    await agent.invoke_async("私の API を呼び出せますか？")

asyncio.run(async_example())
```

### ToolContext

Tool は実行コンテキストにアクセスして、呼び出し元の Agent、現在の Tool 使用データ、呼び出し状態とやり取りできる。`ToolContext` がこのアクセスを提供する：

```python
from strands import tool, Agent, ToolContext

@tool(context=True)
def get_self_name(tool_context: ToolContext) -> str:
    return f"Agent 名は {tool_context.agent.name}"

@tool(context=True)
def get_tool_use_id(tool_context: ToolContext) -> str:
    return f"Tool 使用 ID は {tool_context.tool_use['toolUseId']}"

@tool(context=True)
def get_invocation_state(tool_context: ToolContext) -> str:
    return f"呼び出し状態: {tool_context.invocation_state['custom_data']}"

agent = Agent(tools=[get_self_name, get_tool_use_id, get_invocation_state], name="Best agent")
agent("あなたの名前は何ですか？")
agent("Tool 使用 ID は何ですか？")
agent("呼び出し状態は何ですか？", custom_data="あなたは最高の Agent です ;)")
```

### カスタム ToolContext パラメータ名

ToolContext に別のパラメータ名を使用するには、`@tool.context` 引数の値として希望する名前を指定する：

```python
from strands import tool, Agent, ToolContext

@tool(context="context")
def get_self_name(context: ToolContext) -> str:
    return f"Agent 名は {context.agent.name}"

agent = Agent(tools=[get_self_name], name="Best agent")
agent("あなたの名前は何ですか？")
```

### Tool での State へのアクセス

`ToolContext` の `invocation_state` 属性は、Agent 呼び出しを通じて渡されたデータへのアクセスを提供する。これは特に以下の場合に便利：

- **リクエストコンテキスト**: セッション ID、ユーザー情報、またはリクエスト固有のデータにアクセス
- **マルチエージェント共有状態**: Graph や Swarm パターンでは、すべての Agent 間で共有される状態にアクセス
- **呼び出しごとのオーバーライド**: 特定のリクエストの動作や設定をオーバーライド

```python
from strands import tool, Agent, ToolContext
import requests

@tool(context=True)
def api_call(query: str, tool_context: ToolContext) -> dict:
    """ユーザーコンテキストで API 呼び出しを行う。
    
    Args:
        query: API に送信する検索クエリ
        tool_context: ユーザー情報を含むコンテキスト
    """
    user_id = tool_context.invocation_state.get("user_id")
    response = requests.get(
        "https://api.example.com/search",
        headers={"X-User-ID": user_id},
        params={"q": query}
    )
    return response.json()

agent = Agent(tools=[api_call])
result = agent("プロフィールデータを取得", user_id="user123")
```

**呼び出し状態と他のアプローチの比較**

呼び出し状態が Tool の実行に影響を与える他のアプローチとどのように比較されるかを理解することが重要：

- **Tool パラメータ**: LLM が推論し、ユーザーのリクエストに基づいて提供すべきデータに使用。例: 検索クエリ、ファイルパス、計算入力、または Agent がコンテキストから判断する必要があるデータ。

- **呼び出し状態**: プロンプトに表示されるべきではないが、Tool の動作に影響を与えるコンテキストと設定に使用。Agent の呼び出し間で変更できるパラメータに最適。例: パーソナライゼーション用のユーザー ID、セッション ID、またはユーザーフラグ。

- **クラスベースの Tool**: リクエスト間で変更されず、初期化が必要な設定に使用。例: API キー、データベース接続文字列、サービスエンドポイント、またはセットアップが必要な共有リソース。

### Tool ストリーミング

Async Tool は中間結果を yield してリアルタイムの進捗更新を提供できる。各 yield された値はストリーミングイベントになり、最終値が Tool の戻り結果として機能する：

```python
from datetime import datetime
import asyncio
from strands import tool

@tool
async def process_dataset(records: int) -> str:
    """進捗更新付きでレコードを処理。"""
    start = datetime.now()
    for i in range(records):
        await asyncio.sleep(0.1)
        if i % 10 == 0:
            elapsed = datetime.now() - start
            yield f"{i}/{records} レコードを {elapsed.total_seconds():.1f}秒 で処理"
    
    yield f"{records} レコードを {(datetime.now() - start).total_seconds():.1f}秒 で完了"
```

ストリームイベントには `tool_stream_event` 辞書が含まれ、`tool_use`（呼び出し情報）と `data`（yield された値）フィールドがある：

```python
async def tool_stream_example():
    agent = Agent(tools=[process_dataset])
    async for event in agent.stream_async("50 レコードを処理"):
        if tool_stream := event.get("tool_stream_event"):
            if update := tool_stream.get("data"):
                print(f"進捗: {update}")

asyncio.run(tool_stream_example())
```

## クラスベースの Tool

クラスベースの Tool を使用すると、状態を維持し、オブジェクト指向プログラミングパターンを活用する Tool を作成できる。このアプローチは、Tool がリソースを共有する必要がある場合、呼び出し間でコンテキストを維持する場合、オブジェクト指向設計原則に従う場合、Agent に渡す前に Tool をカスタマイズする場合、または異なる Agent に対して異なる Tool 設定を作成する場合に便利。

### クラス内の複数の Tool の例

同じクラス内に複数の Tool を定義して、関連する機能の一貫したセットを作成できる：

```python
from strands import Agent, tool

class DatabaseTools:
    def __init__(self, connection_string):
        self.connection = self._establish_connection(connection_string)
    
    def _establish_connection(self, connection_string):
        # データベース接続をセットアップ
        return {"connected": True, "db": "example_db"}
    
    @tool
    def query_database(self, sql: str) -> dict:
        """データベースに対して SQL クエリを実行。
        
        Args:
            sql: 実行する SQL クエリ
        """
        # 共有接続を使用
        return {"results": f"クエリ結果: {sql}", "connection": self.connection}
    
    @tool
    def insert_record(self, table: str, data: dict) -> str:
        """データベースに新しいレコードを挿入。
        
        Args:
            table: テーブル名
            data: 挿入するデータ（辞書形式）
        """
        # 共有接続を使用
        return f"{table} にデータを挿入: {data}"

# 使用法
db_tools = DatabaseTools("example_connection_string")
agent = Agent(
    tools=[db_tools.query_database, db_tools.insert_record]
)
```

クラスメソッドに `@tool` デコレーターを使用すると、インスタンス化時にメソッドがクラスインスタンスにバインドされる。つまり、Tool 関数はインスタンスの属性にアクセスでき、呼び出し間で状態を維持できる。

## Tool レスポンス形式

Tool は `ToolResult` 構造を使用してさまざまなフォーマットでレスポンスを返すことができる。この構造は、一貫したインターフェースを維持しながら、異なるタイプのコンテンツを返す柔軟性を提供する。

### ToolResult 構造

`ToolResult` 辞書は以下の構造を持つ：

```python
{
    "toolUseId": str,     # Tool 使用リクエストの ID（受信リクエストと一致すべき）。オプション
    "status": str,         # "success" または "error"
    "content": List[dict]  # 異なるフォーマットを持つコンテンツアイテムのリスト
}
```

### コンテンツタイプ

`content` フィールドはコンテンツブロックのリストで、各ブロックには以下を含めることができる：

- `text`: テキスト出力を含む文字列
- `json`: 任意の JSON シリアライズ可能なデータ構造

### レスポンスの例

**成功レスポンス:**

```python
{
    "toolUseId": "tool-123",
    "status": "success",
    "content": [
        {"text": "操作が正常に完了しました"},
        {"json": {"results": [1, 2, 3], "total": 3}}
    ]
}
```

**エラーレスポンス:**

```python
{
    "toolUseId": "tool-123",
    "status": "error",
    "content": [
        {"text": "エラー: 無効なパラメータのためリクエストを処理できません"}
    ]
}
```

### Tool 結果の処理

`@tool` デコレーターを使用する場合、関数の戻り値は自動的に適切な `ToolResult` に変換される：

- 文字列または他の単純な値を返す場合、`{"text": str(result)}` としてラップされる
- 適切な `ToolResult` 構造を持つ辞書を返す場合、そのまま使用される
- 例外が発生した場合、エラーレスポンスに変換される

## モジュールベースの Tool（Python のみ）

代替アプローチとして、特定の構造を持つ Python モジュールとして Tool を定義できる。これにより、SDK に直接依存しない Tool を作成できる。

Python モジュール Tool には 2 つの主要なコンポーネントが必要：

- Tool の名前、説明、入力スキーマを定義する `TOOL_SPEC` 変数
- Tool 仕様で指定された名前と同じ名前を持つ、Tool の機能を実装する関数

### 基本的な例

天気予報 Tool をモジュールとして実装する方法：

```python
# weather_forecast.py

# 1. Tool 仕様
TOOL_SPEC = {
    "name": "weather_forecast",
    "description": "都市の天気予報を取得。",
    "inputSchema": {
        "json": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "都市名"
                },
                "days": {
                    "type": "integer",
                    "description": "予報の日数",
                    "default": 3
                }
            },
            "required": ["city"]
        }
    }
}

# 2. Tool 関数
def weather_forecast(tool, **kwargs: Any):
    # Tool パラメータを抽出
    tool_use_id = tool["toolUseId"]
    tool_input = tool["input"]
    
    # パラメータ値を取得
    city = tool_input.get("city", "")
    days = tool_input.get("days", 3)
    
    # Tool の実装
    result = f"{city} の {days} 日間の天気予報..."
    
    # 構造化されたレスポンスを返す
    return {
        "toolUseId": tool_use_id,
        "status": "success",
        "content": [{"text": result}]
    }
```

### モジュール Tool の読み込み

モジュールベースの Tool を使用するには、モジュールをインポートして Agent に渡す：

```python
from strands import Agent
import weather_forecast

agent = Agent(
    tools=[weather_forecast]
)
```

または、パスを渡して Tool を読み込むこともできる：

```python
from strands import Agent

agent = Agent(
    tools=["./weather_forecast.py"]
)
```

### 非同期呼び出し

デコレートされた Tool と同様に、モジュール Tool も async として定義できる。

```python
TOOL_SPEC = {
    "name": "call_api",
    "description": "非同期で API を呼び出す。",
    "inputSchema": {
        "json": {
            "type": "object",
            "properties": {},
            "required": []
        }
    }
}

async def call_api(tool, **kwargs):
    await asyncio.sleep(5)  # シミュレートされた API 呼び出し
    result = "API 結果"
    
    return {
        "toolUseId": tool["toolUseId"],
        "status": "success",
        "content": [{"text": result}],
    }
```
