# Lab 3.1: Semantic Kernel / LangChain 連携

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 50 分 |
| 前提 | [Lab 2.1](../step2/lab2-1-python.md) 完了 |
| Azure | 使用しない |

オーケストレーション フレームワーク（**Semantic Kernel** と **LangChain**）の OpenAI コネクタを Foundry Local の endpoint に向け、**Tool / Function Calling** までローカルで動くことを確認します。

> 時間が足りない場合は前半（3.1.A）の Semantic Kernel のみでも OK。

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

## 3.1.A Semantic Kernel（Python）

### インストール

```powershell
mkdir labs-work\step3.1\sk
cd labs-work\step3.1\sk

python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install foundry-local-sdk-winml openai "semantic-kernel>=1.20"
```

### Function Calling サンプル

`sk_app.py`:

```python
import asyncio
import os
from datetime import datetime
from typing import Annotated

from semantic_kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import (
    OpenAIChatCompletion,
    OpenAIChatPromptExecutionSettings,
)
from semantic_kernel.connectors.ai.function_choice_behavior import FunctionChoiceBehavior
from semantic_kernel.contents import ChatHistory
from semantic_kernel.functions import kernel_function


class TimePlugin:
    @kernel_function(description="現在の日時を ISO8601 で返す。")
    def now(self) -> Annotated[str, "ISO8601 datetime"]:
        return datetime.now().isoformat(timespec="seconds")


async def main():
    kernel = Kernel()
    kernel.add_service(
        OpenAIChatCompletion(
            ai_model_id=os.environ["FOUNDRY_LOCAL_MODEL"],
            api_key=os.environ["FOUNDRY_LOCAL_API_KEY"],
            base_url=os.environ["FOUNDRY_LOCAL_ENDPOINT"],
            service_id="foundry-local",
        )
    )
    kernel.add_plugin(TimePlugin(), plugin_name="time")

    settings = OpenAIChatPromptExecutionSettings(
        service_id="foundry-local",
        function_choice_behavior=FunctionChoiceBehavior.Auto(),
        max_tokens=400,
        temperature=0.0,
    )

    history = ChatHistory()
    history.add_system_message("あなたは日本語で答えるアシスタントです。必要に応じてツールを呼びます。")
    history.add_user_message("今は何時ですか？必要ならツールを使ってください。")

    chat = kernel.get_service("foundry-local")
    result = await chat.get_chat_message_content(
        chat_history=history, settings=settings, kernel=kernel,
    )
    print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

実行:

```powershell
python sk_app.py
```

> モデルが Function Calling 未対応 / 弱い場合は `time-now` を直接呼ばずに自然文回答することがあります。その場合は `phi-4-mini` 等の Tool Calling 対応モデルを利用してください。

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
- 本番では Lab 4.2 同様、`base_url` を環境変数で切替可能にしておくとローカル / クラウドを差し替えやすい。

## チェックリスト

- [ ] Semantic Kernel か LangChain のどちらかで応答が返った
- [ ] Tool / Function を呼び出して結果が応答に反映された（モデル依存）
- [ ] endpoint・モデル ID を環境変数で差し替えやすくしてある

次へ → [Lab 3.2: ローカル RAG（Embedding + ベクトル検索）](lab3-2-rag.md)
