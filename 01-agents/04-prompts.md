# Prompts

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/prompts/

## 目次

- [System Prompts](#system-prompts)
- [User Messages](#user-messages)
  - [テキストプロンプト](#テキストプロンプト)
  - [マルチモーダルプロンプト](#マルチモーダルプロンプト)
  - [直接 Tool 呼び出し](#直接-tool-呼び出し)
- [プロンプトエンジニアリング](#プロンプトエンジニアリング)

---

Strands Agents SDK では、System Prompts と User Messages が AI モデルとコミュニケーションする主要な方法である。SDK は、System Prompts と User Messages の両方を含む、プロンプトを管理するための柔軟なシステムを提供する。

## System Prompts

System Prompts は、モデルの役割、機能、制約についての高レベルな指示を提供する。会話全体を通じてモデルがどのように振る舞うべきかの基盤を設定する。Agent を初期化するときに System Prompt を指定できる：

```python
from strands import Agent

agent = Agent(
    system_prompt=(
        "あなたは退職計画に特化したファイナンシャルアドバイザーです。"
        "Tool を使用して情報を収集し、パーソナライズされたアドバイスを提供してください。"
        "常に推論を説明し、可能な場合は出典を引用してください。"
    )
)
```

System Prompt を指定しない場合、モデルはデフォルトの設定に従って動作する。

## User Messages

User Messages は、Agent へのクエリやリクエストである。SDK はプロンプトのための複数のテクニックをサポートする。

### テキストプロンプト

Agent と対話する最もシンプルな方法はテキストプロンプトである：

```python
response = agent("シアトルの現在時刻は？")
```

### マルチモーダルプロンプト

SDK はマルチモーダルプロンプトをサポートしており、メッセージに画像、ドキュメント、その他のコンテンツタイプを含めることができる：

```python
with open("path/to/image.png", "rb") as fp:
    image_bytes = fp.read()

response = agent([
    {"text": "この画像には何が見えますか？"},
    {
        "image": {
            "format": "png",
            "source": {
                "bytes": image_bytes,
            },
        },
    },
])
```

サポートされているコンテンツタイプの完全なリストについては、[API Reference](https://strandsagents.com/latest/documentation/docs/api-reference/python/types/content/#strands.types.content.ContentBlock) を参照。

### 直接 Tool 呼び出し

プロンプトは Strands の主要な機能であり、自然言語リクエストを通じて Tool を呼び出すことができる。しかし、よりプログラマティックな制御が必要な場合、Strands は Tool を直接呼び出すこともできる：

```python
result = agent.tool.current_time(timezone="US/Pacific")
```

直接 Tool 呼び出しは自然言語インターフェースをバイパスし、指定されたパラメータで Tool を実行する。これらの呼び出しはデフォルトで会話履歴に追加される。ただし、Python では `record_direct_tool_call=False` を設定することでこの動作をオプトアウトできる。

## プロンプトエンジニアリング

安全で責任あるプロンプトの書き方に関するガイダンスについては、[Safety & Security - Prompt Engineering](https://strandsagents.com/latest/documentation/docs/user-guide/safety-security/prompt-engineering/) ドキュメントを参照。

その他のリソース：

- [Prompt Engineering Guide](https://www.promptingguide.ai)
- [Amazon Bedrock - Prompt engineering concepts](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-engineering-guidelines.html)
- [Llama - Prompting](https://www.llama.com/docs/how-to-guides/prompting/)
- [Anthropic - Prompt engineering overview](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
- [OpenAI - Prompt engineering](https://platform.openai.com/docs/guides/prompt-engineering/six-strategies-for-getting-better-results)
