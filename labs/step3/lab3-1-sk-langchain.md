# Lab 3.1: Microsoft Agent Framework / LangChain 連携

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 50 分 |
| 前提 | [Lab 2.1](../step2/lab2-1-python.md) 完了 |
| Azure | 使用しない |

オーケストレーション フレームワーク（**Microsoft Agent Framework** と **LangChain**）の OpenAI コネクタを Foundry Local の endpoint に向け、**Tool / Function Calling** までローカルで動くことを確認します。

> ⚠️ **Semantic Kernel と AutoGen は Microsoft Agent Framework に統合されました**。公式の 「[Why Agent Framework?](https://learn.microsoft.com/agent-framework/overview/)」で「**the direct successor**」と明記されており、新規開発は Agent Framework 推奨です。Semantic Kernel 代コードの移行手順は [Semantic Kernel → Agent Framework Migration Guide](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/) 参照。

> 時間が足りない場合は前半（3.1.A）の Agent Framework のみでも OK。

## 3.1.0 共通: モデルと endpoint の取得

両フレームワークとも「OpenAI クライアント互換 + endpoint 差し替え」の構図なので、最初にローカルサービスとモデルを準備します。

`prep.py`:

```python
from foundry_local import FoundryLocalManager

ALIAS = "phi-4-mini"
m = FoundryLocalManager(ALIAS)
print("FOUNDRY_LOCAL_ENDPOINT=", m.endpoint)
print("FOUNDRY_LOCAL_MODEL   =", m.get_model_info(ALIAS).id)
print("FOUNDRY_LOCAL_API_KEY =", m.api_key or "not-needed")
```

実行して環境変数に設定（PowerShell）:

```powershell
python prep.py
# 表示された値を貼り付け
$env:FOUNDRY_LOCAL_ENDPOINT = "http://localhost:5273"
$env:FOUNDRY_LOCAL_MODEL    = "phi-4-mini-instruct-cpu-int4-rtn-block-32-acc-level-4"
$env:FOUNDRY_LOCAL_API_KEY  = "not-needed"
```

---

## 3.1.A Microsoft Agent Framework（Python）

### インストール

```powershell
mkdir labs-work\step3.1\af
cd labs-work\step3.1\af

python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install foundry-local-sdk-winml agent-framework agent-framework-openai
```

> Agent Framework は **コア + プロバイダー個別 パッケージ** の構成。OpenAI 互換 endpoint（= Foundry Local）を使うので `agent-framework-openai` を入れます。Azure OpenAI / Foundry クラウド用は `agent-framework-foundry` と別パッケージです。

### Function Calling サンプル

`af_app.py`:

```python
import asyncio
import os
from datetime import datetime

from agent_framework import tool
from agent_framework.openai import OpenAIChatClient


@tool(approval_mode="never_require")
def now() -> str:
    """現在の日時を ISO8601 で返す。"""
    return datetime.now().isoformat(timespec="seconds")


async def main():
    client = OpenAIChatClient(
        base_url=os.environ["FOUNDRY_LOCAL_ENDPOINT"],  # 例: http://localhost:5273/v1
        api_key=os.environ.get("FOUNDRY_LOCAL_API_KEY", "not-needed"),
        model=os.environ["FOUNDRY_LOCAL_MODEL"],
    )

    agent = client.as_agent(
        name="TimeAgent",
        instructions="あなたは日本語で答えるアシスタントです。必要に応じてツールを呼びます。",
        tools=[now],
    )

    # 非ストリーミング
    response = await agent.run("今は何時ですか？必要ならツールを使ってください。")
    print(response)

    # ストリーミング版（任意）
    print("\n--- streaming ---")
    async for chunk in agent.run("今の時刻を一言で。", stream=True):
        if chunk.text:
            print(chunk.text, end="", flush=True)
    print()


if __name__ == "__main__":
    asyncio.run(main())
```

実行:

```powershell
python af_app.py
```

> モデルが Tool Calling 未対応 / 弱い場合は `now` を呼ばずに自然文で返答することがあります。その場合は `phi-4-mini` 等 Tool Calling 対応モデルを使用してください。

---

## 3.1.B LangChain（Python）

### インストール

```powershell
mkdir labs-work\step3.1\lc
cd labs-work\step3.1\lc

python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install foundry-local-sdk-winml openai langchain langchain-openai
```

### Tool 呼び出しサンプル

`lc_app.py`:

```python
import os
from datetime import datetime

from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI


@tool
def now() -> str:
    """現在の日時を ISO8601 で返す。"""
    return datetime.now().isoformat(timespec="seconds")


llm = ChatOpenAI(
    model=os.environ["FOUNDRY_LOCAL_MODEL"],
    api_key=os.environ["FOUNDRY_LOCAL_API_KEY"],
    base_url=os.environ["FOUNDRY_LOCAL_ENDPOINT"],
    temperature=0,
    max_tokens=400,
)

llm_with_tools = llm.bind_tools([now])

resp = llm_with_tools.invoke([
    SystemMessage("あなたは日本語で答えるアシスタント。必要ならツールを呼びます。"),
    HumanMessage("今は何時ですか？必要ならツールを使ってください。"),
])

print("[tool_calls]", resp.tool_calls)
print("[content]   ", resp.content)
```

実行:

```powershell
python lc_app.py
```

`tool_calls` に `now` が含まれていれば Tool Calling 成功。LangChain は OpenAI 形式の Tool Call を解釈するので、Foundry Local 経由でも同じパターンで動きます。

---

## 3.1.C ポイント整理

- 両フレームワークとも **OpenAI コネクタの `base_url` / `api_key` / `model` を Foundry Local のものに差し替えるだけ** で動く。
- Tool Calling の成否は **モデル側の対応状況** に依存する。動かないときはモデルを切替（`phi-4-mini`, `qwen2.5-7b-instruct` など）。
- Microsoft Agent Framework は Semantic Kernel + AutoGen の後継。既存の SK 資産は [Migration Guide](https://learn.microsoft.com/agent-framework/migration-guide/from-semantic-kernel/) に沿って段階的に移行可（`KernelFunction.as_agent_framework_tool` で互換ツール化も可能）。
- 本番では Lab 4.2 同様、`base_url` を環境変数で切替可能にしておくとローカル / クラウドを差し替えやすい。

## チェックリスト

- [ ] Microsoft Agent Framework か LangChain のどちらかで応答が返った
- [ ] Tool / Function を呼び出して結果が応答に反映された（モデル依存）
- [ ] endpoint・モデル ID を環境変数で差し替えやすくしてある

次へ → [Lab 3.2: ローカル RAG（Embedding + ベクトル検索）](lab3-2-rag.md)
