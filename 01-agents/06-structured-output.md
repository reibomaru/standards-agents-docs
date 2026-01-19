# Structured Output

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/structured-output/

## 目次

- [はじめに](#はじめに)
- [基本的な使用法](#基本的な使用法)
- [詳細情報](#詳細情報)
- [Cookbook](#cookbook)

---

> **Note**: Structured Output は現在 TypeScript SDK ではサポートされていない。

## はじめに

Structured Output を使用すると、Pydantic モデルを使用して、言語モデルから型安全で検証済みの応答を取得できる。パースが必要な生のテキストを受け取る代わりに、スキーマに一致する検証済みの Python オブジェクトを定義して受け取ることができる。これにより、非構造化の LLM 出力が、アプリケーションの型システムと検証ルールにシームレスに統合される信頼性の高いプログラムフレンドリーなデータ構造に変換される。

**主な利点:**

- **型安全性**: 生の文字列ではなく、型付き Python オブジェクトを取得
- **自動検証**: Pydantic がスキーマに対して応答を検証
- **明確なドキュメント**: スキーマが期待される出力のドキュメントとして機能
- **IDE サポート**: LLM が生成した応答に対して IDE の型ヒントが使える
- **エラー防止**: 不正な応答を早期にキャッチ

## 基本的な使用法

Pydantic モデルを使用して出力構造を定義する。次に、Agent を呼び出すときに `structured_output_model` パラメータにモデルを割り当てる。その後、`AgentResult` から Structured Output にアクセスできる。

```python
from pydantic import BaseModel, Field
from strands import Agent

# 1) Pydantic モデルを定義
class PersonInfo(BaseModel):
    """人物に関する情報を含むモデル"""
    name: str = Field(description="人物の名前")
    age: int = Field(description="人物の年齢")
    occupation: str = Field(description="人物の職業")

# 2) モデルを Agent に渡す
agent = Agent()
result = agent(
    "John Smith は 30 歳のソフトウェアエンジニアです",
    structured_output_model=PersonInfo
)

# 3) 結果から `structured_output` にアクセス
person_info: PersonInfo = result.structured_output
print(f"名前: {person_info.name}")      # "John Smith"
print(f"年齢: {person_info.age}")       # 30
print(f"職業: {person_info.occupation}") # "software engineer"
```

### 非同期サポート

Structured Output は `invoke_async` メソッドを介した非同期処理もサポートしている：

```python
import asyncio

agent = Agent()
result = asyncio.run(
    agent.invoke_async(
        "John Smith は 30 歳のソフトウェアエンジニアです",
        structured_output_model=PersonInfo
    )
)
```

## 詳細情報

### 仕組み

Structured Output システムは、Pydantic モデルを Tool 仕様に変換し、言語モデルが正しくフォーマットされた応答を生成するようにガイドする。Strands でサポートされているすべてのモデルプロバイダーは Structured Output で動作する。

Strands は Agent 呼び出しで `structured_output_model` パラメータを受け取り、変換、検証、応答処理を自動的に管理する。検証済みの結果は `AgentResult.structured_output` フィールドで利用できる。

### エラーハンドリング

Structured Output のパースに問題がある場合、Strands はカスタム `StructuredOutputException` をスローする。これをキャッチして適切に処理できる：

```python
from pydantic import ValidationError
from strands.types.exceptions import StructuredOutputException

try:
    result = agent(prompt, structured_output_model=MyModel)
except StructuredOutputException as e:
    print(f"Structured output に失敗: {e}")
```

### レガシー API からの移行

> **Deprecated**: `Agent.structured_output()` および `Agent.structured_output_async()` メソッドは非推奨。代わりに新しい `structured_output_model` パラメータを使用する。

#### 移行前（非推奨）

```python
# 古いアプローチ - 非推奨
result = agent.structured_output(PersonInfo, "John は 30 歳です")
print(result.name)  # モデルフィールドへの直接アクセス
```

#### 移行後（推奨）

```python
# 新しいアプローチ - 推奨
result = agent("John は 30 歳です", structured_output_model=PersonInfo)
print(result.structured_output.name)  # structured_output フィールド経由でアクセス
```

### ベストプラクティス

- **モデルを焦点化**: 明確な目的のために特定のモデルを定義する
- **説明的なフィールド名を使用**: `Field` で有用な説明を含める
- **エラーを適切に処理**: フォールバックを持つ適切なエラーハンドリング戦略を実装する

### 関連ドキュメント

詳細は Pydantic ドキュメントを参照：

- [モデルとスキーマ定義](https://docs.pydantic.dev/latest/concepts/models/)
- [フィールドタイプと制約](https://docs.pydantic.dev/latest/concepts/fields/)
- [カスタムバリデーター](https://docs.pydantic.dev/latest/concepts/validators/)

## Cookbook

### バリデーション付き自動リトライ

フィールドバリデーターにより初期抽出が失敗した場合、自動的にバリデーションをリトライ：

```python
from strands.agent import Agent
from pydantic import BaseModel, field_validator

class Name(BaseModel):
    first_name: str
    
    @field_validator("first_name")
    @classmethod
    def validate_first_name(cls, value: str) -> str:
        if not value.endswith('abc'):
            raise ValueError("名前の末尾に 'abc' を追加する必要があります")
        return value

agent = Agent()
result = agent("Aaron の名前は何ですか？", structured_output_model=Name)
```

### ストリーミング Structured Output

型安全性と検証を維持しながら、Structured Output を段階的にストリーミング：

```python
from strands import Agent
from pydantic import BaseModel, Field

class WeatherForecast(BaseModel):
    """天気予報データ"""
    location: str
    temperature: int
    condition: str
    humidity: int
    wind_speed: int
    forecast_date: str

streaming_agent = Agent()
async for event in streaming_agent.stream_async(
    "シアトルの天気予報を生成: 68°F、曇り時々晴れ、湿度 55%、風速 8 mph、明日",
    structured_output_model=WeatherForecast
):
    if "data" in event:
        print(event["data"], end="", flush=True)
    elif "result" in event:
        print(f'今日の予報: {event["result"].structured_output}')
```

### Tool との組み合わせ

Structured Output と Tool を組み合わせて、Tool 実行結果をフォーマット：

```python
from strands import Agent
from strands_tools import calculator
from pydantic import BaseModel, Field

class MathResult(BaseModel):
    operation: str = Field(description="実行された演算")
    result: int = Field(description="演算の結果")

tool_agent = Agent(
    tools=[calculator]
)

res = tool_agent("42 + 8 は何ですか", structured_output_model=MathResult)
```

### 複数の出力タイプ

単一の Agent インスタンスを異なる Structured Output モデルで再利用し、様々な抽出タスクに対応：

```python
from strands import Agent
from pydantic import BaseModel, Field
from typing import Optional

class Person(BaseModel):
    """人物の基本情報"""
    name: str = Field(description="フルネーム")
    age: int = Field(description="年齢", ge=0, le=150)
    email: str = Field(description="メールアドレス")
    phone: Optional[str] = Field(description="電話番号", default=None)

class Task(BaseModel):
    """タスクまたは TODO アイテム"""
    title: str = Field(description="タスクのタイトル")
    description: str = Field(description="詳細な説明")
    priority: str = Field(description="優先度: low, medium, high")
    completed: bool = Field(description="タスクが完了しているか", default=False)

agent = Agent()

person_res = agent("人物を抽出: John Doe, 35 歳, john@test.com", structured_output_model=Person)
task_res = agent("タスクを作成: コードレビュー, 高優先度, 完了", structured_output_model=Task)
```

### 会話履歴の使用

質問を繰り返さずに、以前の会話コンテキストから構造化情報を抽出：

```python
from strands import Agent
from pydantic import BaseModel
from typing import Optional

agent = Agent()

# 会話コンテキストを構築
agent("フランスのパリについて何を知っていますか？")
agent("春の天気について教えてください。")

class CityInfo(BaseModel):
    city: str
    country: str
    population: Optional[int] = None
    climate: str

# 会話から構造化情報を抽出
result = agent(
    "会話からパリに関する構造化情報を抽出してください",
    structured_output_model=CityInfo
)

print(f"都市: {result.structured_output.city}")      # "Paris"
print(f"国: {result.structured_output.country}")    # "France"
```

### Agent レベルのデフォルト

すべての Agent 呼び出しに適用されるデフォルトの Structured Output モデルを設定することもできる：

```python
class PersonInfo(BaseModel):
    name: str
    age: int
    occupation: str

# すべての呼び出しにデフォルトの Structured Output モデルを設定
agent = Agent(structured_output_model=PersonInfo)

result = agent("John Smith は 30 歳のソフトウェアエンジニアです")
print(f"名前: {result.structured_output.name}")      # "John Smith"
print(f"年齢: {result.structured_output.age}")       # 30
print(f"職業: {result.structured_output.occupation}") # "software engineer"
```

> **Note**: これは Agent 初期化レベルであり、呼び出しレベルではないため、Agent は各呼び出しで Structured Output を試みることが期待される。

### Agent デフォルトのオーバーライド

Agent 初期化レベルでデフォルトの `structured_output_model` を設定しても、Agent 呼び出し時に異なる `structured_output_model` を渡すことで特定の呼び出しでオーバーライドできる：

```python
class PersonInfo(BaseModel):
    name: str
    age: int
    occupation: str

class CompanyInfo(BaseModel):
    name: str
    industry: str
    employees: int

# デフォルト PersonInfo モデルを持つ Agent
agent = Agent(structured_output_model=PersonInfo)

# この特定の呼び出しで CompanyInfo でオーバーライド
result = agent(
    "TechCorp は 500 人の従業員を持つソフトウェア会社です",
    structured_output_model=CompanyInfo
)

print(f"会社: {result.structured_output.name}")       # "TechCorp"
print(f"業界: {result.structured_output.industry}")   # "software"
print(f"規模: {result.structured_output.employees}")  # 500
```
