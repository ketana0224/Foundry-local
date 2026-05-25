# Lab 2.1: Python SDK でチャット & ストリーミング

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 45 分 |
| 前提 | [Lab 1.2](../step1/lab1-2-rest.md) 完了、Python 3.11 以降 |
| Azure | 使用しない |

`foundry-local-sdk(-winml)` と `openai` Python パッケージを組み合わせ、Foundry Local をプログラムから利用します。SDK が裏でローカルサービスの起動とモデル取得を担い、推論呼び出しは OpenAI SDK 経由で行うのが推奨パターンです。

## 2.1.1 仮想環境とパッケージ

```powershell
mkdir labs-work\step2.1
cd labs-work\step2.1

python -m venv .venv
.\.venv\Scripts\Activate.ps1

# Windows（推奨：ハードウェア アクセラレーション付き）
pip install foundry-local-sdk-winml openai

# Windows 以外、または HW アクセラレーション不要の場合
# pip install foundry-local-sdk openai
```

> ⚠ PyPI の `foundry-local`（`-sdk` なし）は **無関係なサードパーティ パッケージ** です。必ず `foundry-local-sdk` か `foundry-local-sdk-winml` を入れてください。
> ⚠ 上記 2 つは `onnxruntime-core` の依存が競合するため **どちらか一方だけ** をインストールします。

## 2.1.2 サンプルコード（チャット + ストリーミング）

`app.py`:

```python
import os
from foundry_local import FoundryLocalManager
from openai import OpenAI

ALIAS = "phi-4-mini"  # ハードウェアに応じて自動で最適 variant を選択

# Foundry Local サービスの起動 + モデル取得を SDK に任せる
manager = FoundryLocalManager(ALIAS)

# OpenAI 互換クライアント
client = OpenAI(
    base_url=manager.endpoint,
    api_key=manager.api_key or "not-needed",
)

# 実際に推論で使うモデル ID (SDK が解決した variant 名)
model_id = manager.get_model_info(ALIAS).id

print(f"endpoint = {manager.endpoint}")
print(f"model_id = {model_id}\n")

# 1. 非ストリーミング
resp = client.chat.completions.create(
    model=model_id,
    messages=[
        {"role": "system", "content": "あなたは簡潔に答える日本語アシスタントです。"},
        {"role": "user",   "content": "Foundry Local とは何ですか？1文で。"},
    ],
    temperature=0.2,
    max_tokens=200,
)
print("[non-streaming]")
print(resp.choices[0].message.content, "\n")

# 2. ストリーミング
print("[streaming]")
stream = client.chat.completions.create(
    model=model_id,
    stream=True,
    messages=[
        {"role": "user", "content": "四国の県を順に説明してください。"},
    ],
)
for chunk in stream:
    delta = chunk.choices[0].delta.content if chunk.choices else None
    if delta:
        print(delta, end="", flush=True)
print()
```

実行:

```powershell
python app.py
```

初回はモデル DL があるため数分かかります。2 回目以降はキャッシュからすぐ起動。

## 2.1.3 環境差吸収パターン（参考）

クラウド側 Foundry へ切り替えやすくするための **明示的な分岐**:

```python
import os
from openai import OpenAI

provider = os.getenv("LLM_PROVIDER", "local")  # "local" or "cloud"

if provider == "local":
    from foundry_local import FoundryLocalManager
    manager = FoundryLocalManager("phi-4-mini")
    client = OpenAI(base_url=manager.endpoint, api_key=manager.api_key or "x")
    model = manager.get_model_info("phi-4-mini").id
else:
    # 例: Lab 4.1 で作成する Azure Foundry の endpoint / key を利用
    client = OpenAI(
        base_url=os.environ["FOUNDRY_CLOUD_ENDPOINT"],
        api_key=os.environ["FOUNDRY_CLOUD_KEY"],
    )
    model = os.environ["FOUNDRY_CLOUD_MODEL"]
```

このパターンは Lab 4.2（フェイルオーバ）で発展させます。

## 2.1.4 トラブルシュート

| 現象 | 原因 / 対処 |
| --- | --- |
| `ModuleNotFoundError: foundry_local` | パッケージ名のタイポ。`foundry-local-sdk(-winml)` を再インストール |
| `Request to local service failed` | `foundry service restart` |
| 応答が極端に遅い | AI Toolkit for VS Code の推論セッションを停止する／量子化が強いモデルへ切替 |
| `model not found` | `manager.get_model_info(ALIAS).id` を出力して実際のモデル ID をリクエストに使う |

## チェックリスト

- [ ] `pip list` に `foundry-local-sdk(-winml)` と `openai` がある
- [ ] 非ストリーミングで日本語応答が返る
- [ ] ストリーミングでトークンが逐次出力される

次へ → [Lab 2.2: .NET / JavaScript SDK 実装](lab2-2-dotnet-js.md)
