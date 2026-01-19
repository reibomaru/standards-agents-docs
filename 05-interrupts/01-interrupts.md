# Interrupts

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/interrupts/

## 目次

- [概要](#概要)
- [Interrupt の種類](#interrupt-の種類)
- [Human-in-the-Loop](#human-in-the-loop)
- [Tool Interrupt](#tool-interrupt)
- [使用例](#使用例)
- [ベストプラクティス](#ベストプラクティス)

---

## 概要

Interrupt は、Agent の実行を一時停止し、外部からの入力や確認を待つためのメカニズム。これにより、Human-in-the-Loop（人間の介入）パターンや、重要なアクションの前に確認を求めるワークフローを実装できる。

Interrupt の主な用途：

- **承認フロー**: 重要なアクションの実行前に人間の承認を取得
- **情報収集**: Agent が追加情報を必要とする場合に入力を要求
- **エラー回復**: エラー発生時に人間の介入を求める
- **安全チェック**: 危険な操作の前に確認を要求

## Interrupt の種類

### 1. User Interrupt

ユーザーからの入力を待つための Interrupt：

```python
from strands import Agent, interrupt

@interrupt
async def request_user_input(question: str) -> str:
    """ユーザーからの入力を要求"""
    # この関数は Agent の実行を一時停止し、
    # 外部からの応答を待つ
    pass

agent = Agent(tools=[request_user_input])
```

### 2. Approval Interrupt

承認を要求するための Interrupt：

```python
from strands import Agent, interrupt

@interrupt
async def request_approval(action: str, details: dict) -> bool:
    """アクションの承認を要求"""
    # 承認または拒否の結果を返す
    pass
```

### 3. Confirmation Interrupt

確認を求めるための Interrupt：

```python
from strands import Agent, interrupt

@interrupt
async def confirm_action(message: str) -> bool:
    """アクションの確認を要求"""
    # True または False を返す
    pass
```

## Human-in-the-Loop

Human-in-the-Loop パターンでは、Agent の実行中に人間が介入できる：

### 基本的な実装

```python
from strands import Agent
from strands.types import Interrupt, InterruptResult

# 承認を必要とするアクション
def sensitive_action(data: dict) -> dict:
    """承認が必要なアクション"""
    # 承認を要求
    raise Interrupt(
        type="approval",
        message="このアクションを実行してもよいですか？",
        data=data
    )

agent = Agent(
    tools=[sensitive_action],
    system_prompt="重要なアクションを実行する前に確認を求めてください。"
)

# Agent を実行
try:
    result = agent.run("データを削除してください")
except Interrupt as interrupt:
    # 人間に承認を求める
    user_approval = get_user_approval(interrupt.message)
    
    if user_approval:
        # 承認された場合、処理を続行
        result = agent.resume(InterruptResult(approved=True))
    else:
        # 拒否された場合
        result = agent.resume(InterruptResult(approved=False))
```

### 非同期での実装

```python
import asyncio
from strands import Agent
from strands.types import Interrupt, InterruptResult

async def run_with_approval():
    agent = Agent(tools=[sensitive_action])
    
    async for event in agent.stream_async("データを処理してください"):
        if isinstance(event, Interrupt):
            # 人間の介入を待つ
            approval = await get_async_approval(event.message)
            await agent.resume_async(InterruptResult(approved=approval))
        elif "data" in event:
            print(event["data"], end="")

asyncio.run(run_with_approval())
```

## Tool Interrupt

Tool 内から Interrupt を発生させる：

### Tool での Interrupt 使用

```python
from strands import Agent, tool
from strands.types import Interrupt

@tool
def delete_files(paths: list[str]) -> str:
    """ファイルを削除する（確認が必要）"""
    # 確認を要求
    raise Interrupt(
        type="confirmation",
        message=f"以下のファイルを削除しますか？\n{paths}",
        data={"paths": paths}
    )

@tool
def transfer_money(amount: float, to_account: str) -> str:
    """送金する（承認が必要）"""
    if amount > 1000:
        raise Interrupt(
            type="approval",
            message=f"{amount}円を{to_account}に送金します。承認しますか？",
            data={"amount": amount, "to_account": to_account}
        )
    
    # 少額の場合は承認なしで実行
    return f"{amount}円を{to_account}に送金しました"
```

### Interrupt のハンドリング

```python
from strands import Agent
from strands.types import Interrupt, InterruptResult

agent = Agent(tools=[delete_files, transfer_money])

def handle_agent_with_interrupts(prompt: str):
    state = agent.start(prompt)
    
    while not state.is_complete:
        if state.interrupt:
            # Interrupt を処理
            interrupt = state.interrupt
            
            if interrupt.type == "confirmation":
                # 確認を求める
                confirmed = input(f"{interrupt.message} (y/n): ").lower() == 'y'
                state = agent.resume(
                    InterruptResult(confirmed=confirmed),
                    state
                )
            
            elif interrupt.type == "approval":
                # 承認を求める
                approved = input(f"{interrupt.message} (y/n): ").lower() == 'y'
                state = agent.resume(
                    InterruptResult(approved=approved),
                    state
                )
        else:
            # 通常の処理を続行
            state = agent.continue_execution(state)
    
    return state.result

result = handle_agent_with_interrupts("大きなファイルを削除してください")
```

## 使用例

### 承認ワークフロー

```python
from strands import Agent, tool
from strands.types import Interrupt

@tool
def create_order(items: list, total: float) -> str:
    """注文を作成する"""
    if total > 10000:
        raise Interrupt(
            type="approval",
            message=f"高額注文の承認が必要です。合計: {total}円",
            data={"items": items, "total": total},
            required_role="manager"  # マネージャーの承認が必要
        )
    
    return f"注文を作成しました: {items}"

@tool
def process_refund(order_id: str, amount: float) -> str:
    """返金処理を行う"""
    raise Interrupt(
        type="approval",
        message=f"注文 {order_id} の返金（{amount}円）を承認しますか？",
        data={"order_id": order_id, "amount": amount}
    )

agent = Agent(
    tools=[create_order, process_refund],
    system_prompt="注文と返金の処理を行います。"
)
```

### 情報収集フロー

```python
from strands import Agent, tool
from strands.types import Interrupt

@tool
def book_flight(destination: str = None, date: str = None) -> str:
    """フライトを予約する"""
    if not destination:
        raise Interrupt(
            type="input",
            message="目的地を教えてください",
            field="destination"
        )
    
    if not date:
        raise Interrupt(
            type="input",
            message="出発日を教えてください（YYYY-MM-DD）",
            field="date"
        )
    
    return f"{date}に{destination}行きのフライトを予約しました"

agent = Agent(tools=[book_flight])
```

### セキュリティチェック

```python
from strands import Agent, tool
from strands.types import Interrupt

@tool
def execute_command(command: str) -> str:
    """システムコマンドを実行する"""
    dangerous_commands = ["rm", "delete", "format", "drop"]
    
    if any(cmd in command.lower() for cmd in dangerous_commands):
        raise Interrupt(
            type="security_check",
            message=f"危険なコマンドが検出されました: {command}\n実行を許可しますか？",
            data={"command": command},
            severity="high"
        )
    
    # コマンドを実行
    return f"コマンドを実行しました: {command}"
```

## ベストプラクティス

### 1. 明確なメッセージ

```python
# 良い例
raise Interrupt(
    type="approval",
    message="顧客データベースから100件のレコードを削除します。この操作は元に戻せません。続行しますか？",
    data={"count": 100, "table": "customers"}
)

# 悪い例
raise Interrupt(
    type="approval",
    message="削除しますか？"
)
```

### 2. 適切な粒度

すべてのアクションに Interrupt を設定するのではなく、重要なものだけに：

```python
# 重要なアクションのみに Interrupt
if amount > threshold:
    raise Interrupt(...)

# 小さなアクションは承認なしで実行
return execute_action()
```

### 3. タイムアウト設定

```python
interrupt = Interrupt(
    type="approval",
    message="承認してください",
    timeout=3600,  # 1時間
    on_timeout="reject"  # タイムアウト時は拒否
)
```

### 4. ロールベースの承認

```python
interrupt = Interrupt(
    type="approval",
    message="高額取引の承認",
    required_role="finance_manager",
    escalation_path=["team_lead", "director"]
)
```
