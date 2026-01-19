# Session Management

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/session-management/

## 目次

- [概要](#概要)
- [基本的な使用法](#基本的な使用法)
- [組み込み Session Manager](#組み込み-session-manager)
- [Session Management の仕組み](#session-management-の仕組み)
- [サードパーティ Session Manager](#サードパーティ-session-manager)
- [カスタム Session Repository](#カスタム-session-repository)
- [ベストプラクティス](#ベストプラクティス)

---

> **Note**: Session Management は現在 TypeScript SDK ではサポートされていないが、近日対応予定。

Strands Agents における Session Management は、複数のインタラクションにわたって Agent の State と会話履歴を永続化するための堅牢なメカニズムを提供する。これにより、アプリケーションが再起動した場合や分散環境にデプロイされた場合でも、Agent がコンテキストと継続性を維持できる。

## 概要

Session は、Agent およびマルチエージェントシステムが機能するために必要なすべてのステートフル情報を表す：

**単一 Agent Session:**
- 会話履歴（messages）
- Agent State（キーバリューストレージ）
- その他のステートフル情報（Conversation Manager など）

**マルチエージェント Session:**
- Orchestrator の State と設定
- Orchestrator 内の個々の Agent の State と結果
- Agent 間で共有される State とコンテキスト
- 実行フローとノード遷移履歴

Strands は、この情報を自動的にキャプチャして復元する組み込みの Session 永続化機能を提供し、Agent とマルチエージェントシステムが中断したところからシームレスに会話を続けられるようにする。

> **Warning**: Session Manager を持つ単一の Agent をマルチエージェントシステムで使用することはできない。例外がスローされる。マルチエージェントシステム内の各 Agent は Session Manager なしで作成し、Orchestrator のみが Session Manager を持つべきである。また、マルチエージェント Session Manager は Graph/Swarm 実行の現在の State のみを追跡し、個々の Agent の会話履歴を永続化しない。

## 基本的な使用法

### 単一 Agent Session

Session Manager を持つ Agent を作成して使用するだけ：

```python
from strands import Agent
from strands.session.file_session_manager import FileSessionManager

# 一意の Session ID で Session Manager を作成
session_manager = FileSessionManager(session_id="test-session")

# Session Manager を持つ Agent を作成
agent = Agent(session_manager=session_manager)

# Agent を使用 - すべてのメッセージと State が自動的に永続化される
agent("Hello!")  # この会話は永続化される
```

会話と関連する State は、基盤となるファイルシステムに永続化される。

### マルチエージェント Session

マルチエージェントシステム（Graph/Swarm）も Session Management を使用して State を永続化できる：

```python
from strands.multiagent import Graph
from strands.session.file_session_manager import FileSessionManager

# Agent を作成
agent1 = Agent(name="researcher")
agent2 = Agent(name="writer")

# Graph 用の Session Manager を作成
session_manager = FileSessionManager(session_id="multi-agent-session")

# Session Management 付きで Graph を作成
graph = Graph(
    agents={"researcher": agent1, "writer": agent2},
    session_manager=session_manager
)

# Graph を使用 - すべての Orchestrator State が永続化される
result = graph("Research and write about AI")
```

## 組み込み Session Manager

Strands は Agent Session を永続化するための 2 つの組み込み Session Manager を提供する：

- **FileSessionManager**: ローカルファイルシステムに Session を保存
- **S3SessionManager**: Amazon S3 バケットに Session を保存

### FileSessionManager

`FileSessionManager` は、単一 Agent とマルチエージェント Session の両方をローカルファイルシステムに永続化するシンプルな方法を提供する：

```python
from strands import Agent
from strands.session.file_session_manager import FileSessionManager

# 一意の Session ID で Session Manager を作成
session_manager = FileSessionManager(
    session_id="user-123",
    storage_dir="/path/to/sessions"  # オプション、デフォルトは一時ディレクトリ
)

# Session Manager を持つ Agent を作成
agent = Agent(session_manager=session_manager)

# Agent を通常どおり使用 - State とメッセージは自動的に永続化される
agent("Hello, I'm a new user!")
```

#### ファイルストレージ構造

```
/<sessions_dir>/
└── session_<session_id>/
    ├── session.json                    # Session メタデータ
    ├── agents/                         # 単一 Agent ストレージ
    │   └── agent_<agent_id>/
    │       ├── agent.json              # Agent メタデータと State
    │       └── messages/
    │           ├── message_<message_id>.json
    │           └── message_<message_id>.json
    └── multi_agents/                   # マルチエージェントストレージ
        └── multi_agent_<orchestrator_id>/
            └── multi_agent.json        # Orchestrator State と設定
```

### S3SessionManager

クラウドベースの永続化、特に分散環境では `S3SessionManager` を使用：

```python
from strands import Agent
from strands.session.s3_session_manager import S3SessionManager
import boto3

# オプション: カスタム boto3 Session を作成
boto_session = boto3.Session(region_name="us-west-2")

# S3 にデータを保存する Session Manager を作成
session_manager = S3SessionManager(
    session_id="user-456",
    bucket="my-agent-sessions",
    prefix="production/",              # オプションのキープレフィックス
    boto_session=boto_session,         # オプションの boto3 Session
    region_name="us-west-2"            # オプションの AWS リージョン
)

# Session Manager を持つ Agent を作成
agent = Agent(session_manager=session_manager)
```

#### 必要な S3 権限

`S3SessionManager` を使用するには、以下の S3 権限が必要：

- `s3:PutObject` - Session データの作成と更新
- `s3:GetObject` - Session データの取得
- `s3:DeleteObject` - Session データの削除
- `s3:ListBucket` - バケット内のオブジェクト一覧

## Session Management の仕組み

### 1. Session 永続化トリガー

Session の永続化は、Agent およびマルチエージェントライフサイクルにおけるいくつかの主要なイベントによって自動的にトリガーされる：

**単一 Agent イベント:**
- **Agent 初期化**: Session Manager を持つ Agent が作成されると、Session から既存の State とメッセージが自動的に復元される
- **メッセージ追加**: 新しいメッセージが会話に追加されると、自動的に Session に永続化される
- **Agent 呼び出し**: 各 Agent 呼び出し後、Agent State が Session と同期されて更新がキャプチャされる
- **メッセージ墨消し**: 機密情報を墨消しする必要がある場合、Session Manager は会話フローを維持しながら元のメッセージを墨消しバージョンに置き換えることができる

**マルチエージェントイベント:**
- **マルチエージェント初期化**: Orchestrator が Session Manager を持って作成されると、Session から State が自動的に復元される
- **ノード実行**: 各ノード呼び出し後、ノード遷移後に Orchestrator State を同期
- **マルチエージェント呼び出し**: マルチエージェント完了後、実行後の最終 Orchestrator State をキャプチャ

> **Note**: Agent を初期化した後、`agent.messages` への直接変更は永続化されない。永続化可能な方法で Agent のコンテキストを管理するには Conversation Manager を使用する。

### 2. データモデル

Session データは以下の主要なデータモデルを使用して保存される：

**Session**
- **目的**: 複数の Agent とそのインタラクションを整理するための名前空間を提供
- **主要フィールド**:
  - `session_id`: Session の一意の識別子
  - `session_type`: Session のタイプ
  - `created_at`: Session が作成された ISO 形式のタイムスタンプ
  - `updated_at`: Session が最後に更新された ISO 形式のタイムスタンプ

**SessionAgent**
- **目的**: Session 内の特定の Agent の State と設定を維持
- **主要フィールド**:
  - `agent_id`: Session 内の Agent の一意の識別子
  - `state`: Agent の State データを含む辞書（キーバリューペア）
  - `conversation_manager_state`: Conversation Manager の State を含む辞書
  - `created_at` / `updated_at`: タイムスタンプ

**SessionMessage**
- **目的**: メッセージ墨消しをサポートして会話履歴を保存
- **主要フィールド**:
  - `message`: 元のメッセージコンテンツ（role、content blocks）
  - `redact_message`: オプションの墨消しバージョン
  - `message_id`: Agent の messages 配列内のメッセージのインデックス
  - `created_at` / `updated_at`: タイムスタンプ

## サードパーティ Session Manager

以下のサードパーティ Session Manager は、追加のストレージとメモリ機能で Strands を拡張する：

| Session Manager | プロバイダー | 説明 |
|----------------|------------|------|
| AgentCoreMemorySessionManager | Amazon | Amazon Bedrock AgentCore Memory を使用したインテリジェントな取得機能を持つ高度なメモリ。短期記憶（STM）と長期記憶（LTM）の両方をサポート |

## カスタム Session Repository

高度なユースケースでは、カスタム Session Repository を作成して独自のセッションストレージバックエンドを実装できる：

```python
from typing import Optional
from strands import Agent
from strands.session.repository_session_manager import RepositorySessionManager
from strands.session.session_repository import SessionRepository
from strands.types.session import Session, SessionAgent, SessionMessage

class CustomSessionRepository(SessionRepository):
    """カスタム Session Repository 実装"""
    
    def __init__(self):
        """カスタムストレージバックエンドを初期化"""
        self.db = YourDatabaseClient()
    
    def create_session(self, session: Session) -> Session:
        """新しい Session を作成"""
        self.db.sessions.insert(asdict(session))
        return session
    
    def read_session(self, session_id: str) -> Optional[Session]:
        """ID で Session を読み取る"""
        data = self.db.sessions.find_one({"session_id": session_id})
        if data:
            return Session.from_dict(data)
        return None
    
    # 他の必要なメソッドを実装...

# カスタム Repository を RepositorySessionManager で使用
custom_repo = CustomSessionRepository()
session_manager = RepositorySessionManager(
    session_id="user-789",
    session_repository=custom_repo
)

agent = Agent(session_manager=session_manager)
```

## ベストプラクティス

アプリケーションで Session 永続化を実装する際は、以下のベストプラクティスを考慮：

1. **一意の Session ID を使用**: データの重複を防ぐために、各ユーザーまたは会話コンテキストに一意の Session ID を生成する

2. **Session のクリーンアップ**: 古いまたは非アクティブな Session をクリーンアップする戦略を実装する。本番環境では Session に TTL（Time To Live）を追加することを検討

3. **永続化トリガーの理解**: Agent State またはメッセージへの変更は、特定のライフサイクルイベント中にのみ永続化されることを覚えておく

4. **並行アクセス**: Session Manager はスレッドセーフではない。並行アクセスには適切なロックを使用
