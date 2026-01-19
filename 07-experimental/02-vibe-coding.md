# Vibe Coding

> 原文: https://strandsagents.com/latest/documentation/docs/user-guide/concepts/experimental/vibe-coding/

## 目次

- [概要](#概要)
- [Vibe Coding とは](#vibe-coding-とは)
- [セットアップ](#セットアップ)
- [基本的な使い方](#基本的な使い方)
- [高度な機能](#高度な機能)
- [ベストプラクティス](#ベストプラクティス)
- [制限事項](#制限事項)

---

## 概要

Vibe Coding は、自然言語でコードを生成、修正、リファクタリングする実験的機能。「雰囲気（vibe）」を伝えるだけで、意図したコードを生成できる。

> ⚠️ **注意**: これは実験的機能です。API は予告なく変更される可能性があります。

## Vibe Coding とは

### 従来のコーディング vs Vibe Coding

**従来のコーディング**：
```python
# 明示的にコードを書く
def authenticate_user(username: str, password: str) -> bool:
    user = database.get_user(username)
    if user is None:
        return False
    return verify_password(password, user.password_hash)
```

**Vibe Coding**：
```python
from strands.experimental import VibeCoder

coder = VibeCoder()
code = coder.generate(
    "ユーザー認証関数を作って。"
    "データベースからユーザーを取得して、"
    "パスワードを検証する。"
)
```

### 主な特徴

1. **自然言語入力**: 日本語や英語で意図を伝える
2. **コンテキスト理解**: 既存コードの文脈を理解
3. **スタイル適応**: プロジェクトのコーディングスタイルに適応
4. **インタラクティブ修正**: 生成結果を対話的に改善

## セットアップ

### インストール

```bash
pip install strands[experimental]
```

### 有効化

```python
from strands.experimental import enable_experimental, VibeCoder

# 実験的機能を有効化
enable_experimental("vibe_coding")

# VibeCoder を初期化
coder = VibeCoder(
    language="python",
    style_guide="pep8"
)
```

### 設定オプション

```python
coder = VibeCoder(
    # プログラミング言語
    language="python",
    
    # コーディングスタイル
    style_guide="pep8",  # pep8, google, custom
    
    # 型ヒントの使用
    use_type_hints=True,
    
    # ドキュメント生成
    generate_docs=True,
    
    # テスト生成
    generate_tests=True,
    
    # モデル設定
    model="claude-3-opus",
    temperature=0.2
)
```

## 基本的な使い方

### コード生成

```python
from strands.experimental import VibeCoder

coder = VibeCoder(language="python")

# シンプルな関数を生成
code = coder.generate(
    "素数判定関数を作成して"
)
print(code)
# def is_prime(n: int) -> bool:
#     """Check if a number is prime."""
#     if n < 2:
#         return False
#     for i in range(2, int(n ** 0.5) + 1):
#         if n % i == 0:
#             return False
#     return True
```

### クラス生成

```python
code = coder.generate("""
ユーザー管理クラスを作成して。
- ユーザーの作成、取得、更新、削除ができる
- データはメモリに保存
- ユーザーIDは自動生成
""")
```

### API エンドポイント生成

```python
code = coder.generate("""
FastAPI で REST API を作成して。
- /users エンドポイント
- CRUD 操作をサポート
- Pydantic モデルを使用
- 適切なエラーハンドリング
""")
```

## 高度な機能

### コード修正

```python
existing_code = """
def calculate_total(items):
    total = 0
    for item in items:
        total += item.price
    return total
"""

modified = coder.modify(
    code=existing_code,
    instruction="税金（10%）と送料（500円以上で無料、それ以下は300円）を追加して"
)
```

### リファクタリング

```python
legacy_code = """
def process(d):
    r = []
    for i in d:
        if i['a'] > 10:
            r.append(i['b'] * 2)
    return r
"""

refactored = coder.refactor(
    code=legacy_code,
    goals=[
        "変数名を分かりやすく",
        "リスト内包表記を使用",
        "型ヒントを追加",
        "ドキュメントを追加"
    ]
)
```

### コンテキスト認識

```python
# 既存のコードベースを読み込み
coder.load_context(
    files=["models.py", "utils.py"],
    imports=["from myapp.models import User, Product"]
)

# コンテキストを考慮したコード生成
code = coder.generate(
    "User モデルを使ってユーザー検索機能を実装して"
)
```

### インタラクティブセッション

```python
# インタラクティブセッションを開始
session = coder.start_session()

# 初期コードを生成
code = session.generate("ToDo アプリのバックエンドを作成")

# 修正を依頼
code = session.refine("認証機能を追加して")

# さらに修正
code = session.refine("テストも書いて")

# セッションを終了
session.end()
```

### テスト生成

```python
function_code = """
def validate_email(email: str) -> bool:
    import re
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))
"""

tests = coder.generate_tests(
    code=function_code,
    framework="pytest",
    coverage=["happy_path", "edge_cases", "error_cases"]
)
```

## ベストプラクティス

### 1. 明確な指示

```python
# 良い例: 具体的で明確
code = coder.generate("""
ユーザー登録 API エンドポイントを作成:
- POST /api/users
- 入力: email, password, name
- パスワードは bcrypt でハッシュ化
- 重複メールはエラー
- 成功時は 201 と user_id を返す
""")

# 悪い例: 曖昧
code = coder.generate("ユーザー登録を作って")
```

### 2. 段階的な生成

```python
# 複雑な機能は段階的に
# Step 1: 基本構造
base = coder.generate("ショッピングカートクラスの基本構造")

# Step 2: メソッドを追加
with_add = coder.modify(base, "商品追加メソッドを実装")

# Step 3: さらに機能を追加
complete = coder.modify(with_add, "割引適用機能を追加")
```

### 3. スタイルガイドの活用

```python
coder = VibeCoder(
    language="python",
    style_guide="custom",
    custom_rules={
        "max_line_length": 88,
        "use_trailing_commas": True,
        "docstring_style": "google",
        "naming_convention": "snake_case"
    }
)
```

### 4. レビューと検証

```python
code = coder.generate("データ処理パイプライン")

# 生成されたコードをレビュー
review = coder.review(
    code=code,
    checks=[
        "security",      # セキュリティ問題
        "performance",   # パフォーマンス問題
        "best_practices" # ベストプラクティス
    ]
)

if review.issues:
    # 問題を修正
    code = coder.fix(code, review.issues)
```

## 制限事項

### 現在の制限

1. **言語サポート**: 現在は Python、JavaScript、TypeScript のみ
2. **コード長**: 一度に生成できるコードは約 1000 行まで
3. **複雑な依存関係**: 複雑な外部ライブラリの使用は限定的
4. **パフォーマンス最適化**: 高度な最適化は手動で行う必要がある

### 非推奨のユースケース

```python
# 以下のケースでは注意が必要:

# 1. セキュリティクリティカルなコード
# → 必ず専門家によるレビューを行う

# 2. パフォーマンスクリティカルなコード
# → ベンチマークと最適化が必要

# 3. 複雑なアルゴリズム
# → 正確性の検証が必要
```

### 推奨されるワークフロー

```python
# 1. プロトタイピング
prototype = coder.generate("機能のプロトタイプ")

# 2. レビュー
review = coder.review(prototype)

# 3. 手動で改善
# エディタで編集

# 4. テスト
tests = coder.generate_tests(prototype)

# 5. 本番コードとして採用
```
