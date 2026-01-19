# Conversation Management

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/conversation-management/

## 目次

- [概要](#概要)
- [組み込み Conversation Manager](#組み込み-conversation-manager)
- [カスタム ConversationManager の作成](#カスタム-conversationmanager-の作成)

---

Strands Agents SDK において、コンテキストとは Agent に理解と推論のために提供される情報を指す。これには以下が含まれる：

- ユーザーメッセージ
- Agent の応答
- Tool の使用と結果
- System Prompts

会話が成長するにつれて、このコンテキストの管理がますます重要になる。理由は以下の通り：

- **トークン制限**: 言語モデルには固定のコンテキストウィンドウ（処理可能な最大トークン数）がある
- **パフォーマンス**: 大きなコンテキストはより多くの処理時間とリソースを必要とする
- **関連性**: 古いメッセージは現在の会話との関連性が低くなる可能性がある
- **一貫性**: 論理的なフローを維持し、重要な情報を保持する

## 組み込み Conversation Manager

SDK は `ConversationManager` インターフェースを通じて、会話履歴を管理するための柔軟なシステムを提供する。これにより、異なる戦略を実装できる。Strands が提供するマネージャーを活用するか、独自のマネージャーを構築できる：

- **NullConversationManager**: 会話履歴を変更しないシンプルな実装
- **SlidingWindowConversationManager**: 固定数の最近のメッセージを維持（デフォルトマネージャー）
- **SummarizingConversationManager**: 古いメッセージをインテリジェントに要約してコンテキストを保持

### NullConversationManager

`NullConversationManager` は会話履歴を変更しないシンプルな実装。以下の場合に便利：

- コンテキスト制限を超えない短い会話
- デバッグ目的
- コンテキストを手動で管理したい場合

```python
from strands import Agent
from strands.agent.conversation_manager import NullConversationManager

agent = Agent(
    conversation_manager=NullConversationManager()
)
```

### SlidingWindowConversationManager

`SlidingWindowConversationManager` は、固定数の最近のメッセージを維持するスライディングウィンドウ戦略を実装する。これは Agent クラスが使用するデフォルトの Conversation Manager。

```python
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

# カスタムウィンドウサイズで Conversation Manager を作成
conversation_manager = SlidingWindowConversationManager(
    window_size=20,  # 保持するメッセージの最大数
    should_truncate_results=True,  # メッセージがモデルのコンテキストウィンドウに対して大きすぎる場合に Tool 結果を切り詰める
)

agent = Agent(
    conversation_manager=conversation_manager
)
```

**主な機能:**

- **ウィンドウサイズの維持**: メッセージ数が制限を超えると、ウィンドウから自動的にメッセージを削除
- **ダングリングメッセージのクリーンアップ**: 有効な会話状態を維持するために不完全なメッセージシーケンスを削除
- **オーバーフローのトリミング**: コンテキストウィンドウのオーバーフローの場合、リクエストがモデルのコンテキストウィンドウに収まるまで履歴から最も古いメッセージを削除
- **設定可能な Tool 結果の切り詰め**: メッセージがコンテキストウィンドウの制限を超えた場合の Tool 結果の切り詰めを有効/無効にする
- **ターンごとの管理**: Agent ループの実行中にプロアクティブにコンテキスト管理を適用するオプション

#### ターンごとの管理

デフォルトでは、`SlidingWindowConversationManager` は Agent ループが完了した後にのみコンテキスト管理を適用する。`per_turn` パラメータを使用すると、実行中にプロアクティブにコンテキストを管理でき、多くの Tool 呼び出しを持つ長時間実行の Agent ループに便利。

```python
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

# すべてのモデル呼び出しの前に管理を適用
conversation_manager = SlidingWindowConversationManager(
    per_turn=True,  # 各モデル呼び出しの前に管理を適用
)

# または N 回のモデル呼び出しごとに管理を適用
conversation_manager = SlidingWindowConversationManager(
    per_turn=3,  # 3 回のモデル呼び出しごとに管理を適用
)

agent = Agent(
    conversation_manager=conversation_manager
)
```

`per_turn` パラメータは以下を受け付ける：

- `False`（デフォルト）: Agent ループが完了した後にのみ管理を適用
- `True`: すべてのモデル呼び出しの前に管理を適用
- 整数 `N`（0 より大きい必要がある）: N 回のモデル呼び出しごとに管理を適用

### SummarizingConversationManager

> **Note**: TypeScript ではサポートされていない

`SummarizingConversationManager` は、古いメッセージを単純に破棄するのではなく、要約することでインテリジェントな会話コンテキスト管理を実装する。このアプローチは、コンテキスト制限内に収まりながら重要な情報を保持する。

**設定パラメータ:**

- **`summary_ratio`**（float、デフォルト: 0.3）: コンテキスト削減時に要約するメッセージの割合（0.1〜0.8 の間にクランプ）
- **`preserve_recent_messages`**（int、デフォルト: 10）: 常に保持する最近のメッセージの最小数
- **`summarization_agent`**（Agent、オプション）: 要約を生成するためのカスタム Agent。指定しない場合は、メイン Agent インスタンスを使用。`summarization_system_prompt` と一緒に使用できない
- **`summarization_system_prompt`**（str、オプション）: 要約用のカスタム System Prompt。指定しない場合は、キートピック、使用された Tool、技術情報に焦点を当てた構造化された箇条書きの要約を作成するデフォルトプロンプトを使用。`summarization_agent` と一緒に使用できない

#### 基本的な使用法

デフォルトでは、`SummarizingConversationManager` はメイン Agent と同じモデルと設定を使用して要約を実行：

```python
from strands import Agent
from strands.agent.conversation_manager import SummarizingConversationManager

agent = Agent(
    conversation_manager=SummarizingConversationManager()
)
```

要約率や保持するメッセージ数などのパラメータを調整してカスタマイズすることもできる：

```python
from strands import Agent
from strands.agent.conversation_manager import SummarizingConversationManager

# デフォルト設定で要約 Conversation Manager を作成
conversation_manager = SummarizingConversationManager(
    summary_ratio=0.3,              # コンテキスト削減が必要な場合に 30% のメッセージを要約
    preserve_recent_messages=10,    # 最新の 10 メッセージを常に保持
)

agent = Agent(
    conversation_manager=conversation_manager
)
```

#### ドメイン固有の要約用カスタム System Prompt

ドメインやユースケースに合わせた要約動作をカスタマイズするために、カスタム System Prompt を提供できる：

```python
from strands import Agent
from strands.agent.conversation_manager import SummarizingConversationManager

# 技術的な会話用のカスタム System Prompt
custom_system_prompt = """
あなたは技術的な会話を要約しています。
以下に焦点を当てた簡潔な箇条書きの要約を作成してください：
- コード変更、アーキテクチャの決定、技術的な解決策
- 特定の関数名、ファイルパス、設定の詳細を保持
- 会話的な要素を省略し、実行可能な情報に焦点を当てる
- ソフトウェア開発に適した技術用語を使用

会話的な言葉を使わず、箇条書きでフォーマットしてください。
"""

conversation_manager = SummarizingConversationManager(
    summarization_system_prompt=custom_system_prompt
)

agent = Agent(
    conversation_manager=conversation_manager
)
```

#### カスタム要約 Agent を使用した高度な設定

高度なユースケースでは、要約プロセスを処理するカスタム `summarization_agent` を提供できる。これにより、異なるモデル（より高速またはコスト効率の良いもの）の使用、要約中の Tool の組み込み、ドメインに合わせた特殊な要約ロジックの実装が可能になる：

```python
from strands import Agent
from strands.agent.conversation_manager import SummarizingConversationManager
from strands.models import AnthropicModel

# 要約タスク用の安価で高速なモデルを作成
summarization_model = AnthropicModel(
    model_id="claude-3-5-haiku-20241022",  # 要約にコスト効率の良いモデル
    max_tokens=1000,
    params={"temperature": 0.1}  # 一貫した要約のための低い温度
)

custom_summarization_agent = Agent(model=summarization_model)

conversation_manager = SummarizingConversationManager(
    summary_ratio=0.4,
    preserve_recent_messages=8,
    summarization_agent=custom_summarization_agent
)

agent = Agent(
    conversation_manager=conversation_manager
)
```

**主な機能:**

- **コンテキストウィンドウ管理**: トークン制限を超えると自動的にコンテキストを削減
- **インテリジェントな要約**: 構造化された箇条書きの要約を使用して重要な情報をキャプチャ
- **Tool ペアの保持**: 要約中に Tool の使用と結果のメッセージペアが壊れないことを保証
- **柔軟な設定**: さまざまなパラメータで要約動作をカスタマイズ
- **フォールバックの安全性**: 要約の失敗を適切に処理

## カスタム ConversationManager の作成

カスタム Conversation Manager を作成するには、`ConversationManager` インターフェースを実装する。これは 3 つの主要な要素で構成される：

- **`apply_management`**: 各イベントループサイクルが完了した後に呼び出され、会話履歴を管理する。Tool の結果と Assistant の応答で変更された可能性のあるメッセージ配列に管理戦略を適用する責任がある。Agent は各ユーザー入力を処理し、応答を生成した後にこのメソッドを自動的に実行する

- **`reduce_context`**: モデルのコンテキストウィンドウを超えた場合（通常はトークン制限による）に呼び出される。必要に応じてウィンドウサイズを削減するための特定の戦略を実装する。Agent はコンテキストウィンドウオーバーフロー例外が発生したときにこのメソッドを呼び出し、再試行する前に会話履歴を削減する機会を実装に与える

- **`removed_message_count`**: Conversation Manager によって追跡されるこの属性は、Session Management がセッションストレージからメッセージを効率的にロードするために使用される。カウントはユーザーまたは LLM によって提供され、Agent のメッセージから削除されたメッセージを表すが、要約などで Conversation Manager によって含まれたメッセージは含まない

- **`register_hooks`**（オプション）: Hook と統合するためにこのメソッドをオーバーライドする。これにより、モデル呼び出しの前にコンテキストを削減するなどのプロアクティブなコンテキスト管理パターンが可能になる。オーバーライドする場合は常に `super().register_hooks` を呼び出すこと

参照例として [SlidingWindowConversationManager](https://github.com/strands-agents/sdk-python/blob/main/src/strands/agent/conversation_manager/sliding_window_conversation_manager.py) の実装を参照。
