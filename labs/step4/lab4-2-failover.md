# Lab 4.2: ローカル優先 + クラウドフォールバック

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 50 分 |
| 前提 | [Lab 4.1](lab4-1-foundry-cloud.md) 完了（Azure 側リソース有効） |
| Azure | Lab 4.1 のリソースを流用 |

クライアント側で **ローカル（Foundry Local） → クラウド（Microsoft Foundry）** の順にフェイルオーバする多層構成を実装します。  
オフライン時・ローカル PC のリソース不足時・モデル未対応時にクラウドへ自動切替するパターンを学びます。

## 4.2.1 設計

```
┌─────────────────────────────┐
│      Application            │
│                             │
│   ChatService.complete()    │
│        │                    │
│        ▼                    │
│  ┌──────────┐   fail / 5xx  │
│  │ local    │──────────────►┐
│  │ Foundry  │               │
│  │ Local    │               │
│  └──────────┘               │
│        │ ok                 │
│        ▼                    │
│   return                    │
│                             │
│        ▲                    │
│        │ ok                 │
│  ┌──────────┐               │
│  │ cloud    │◄──────────────┘
│  │ Microsoft│
│  │ Foundry  │
│  └──────────┘
└─────────────────────────────┘
```

判定ロジック:

| 状況 | 動作 |
| --- | --- |
| ローカルが応答（HTTP 200, タイムアウト内） | ローカルを採用 |
| ローカルが接続エラー / 5xx / タイムアウト | クラウドへフォールバック |
| いずれも失敗 | エラーを伝搬 |

## 4.2.2 環境変数

```powershell
# 既に Lab 4.1 で設定済みなら不要
$env:FOUNDRY_CLOUD_ENDPOINT = "<取得した v1 互換 endpoint>"
$env:FOUNDRY_CLOUD_KEY      = "<key1>"
$env:FOUNDRY_CLOUD_MODEL    = "gpt-4o-mini"
```

## 4.2.3 実装

`failover.py`:

```python
import os
import time
import logging
from dataclasses import dataclass
from typing import Iterable, List

import httpx
from foundry_local import FoundryLocalManager
from openai import OpenAI, APIConnectionError, APIStatusError, APITimeoutError

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
log = logging.getLogger("failover")


@dataclass
class Backend:
    name: str
    client: OpenAI
    model: str


def build_local() -> Backend:
    mgr = FoundryLocalManager("phi-4-mini")
    client = OpenAI(
        base_url=mgr.endpoint,
        api_key=mgr.api_key or "x",
        timeout=httpx.Timeout(connect=2.0, read=30.0, write=30.0, pool=2.0),
        max_retries=0,
    )
    return Backend("local", client, mgr.get_model_info("phi-4-mini").id)


def build_cloud() -> Backend:
    client = OpenAI(
        base_url=os.environ["FOUNDRY_CLOUD_ENDPOINT"],
        api_key=os.environ["FOUNDRY_CLOUD_KEY"],
        timeout=httpx.Timeout(connect=5.0, read=60.0, write=30.0, pool=5.0),
        max_retries=1,
    )
    return Backend("cloud", client, os.environ["FOUNDRY_CLOUD_MODEL"])


def chat_with_failover(backends: Iterable[Backend], messages: List[dict]) -> str:
    last_err: Exception | None = None
    for b in backends:
        t0 = time.perf_counter()
        try:
            r = b.client.chat.completions.create(
                model=b.model, messages=messages,
                temperature=0.0, max_tokens=400,
            )
            log.info("[%s] OK in %.1fs", b.name, time.perf_counter() - t0)
            return r.choices[0].message.content or ""
        except (APIConnectionError, APITimeoutError) as e:
            log.warning("[%s] connection/timeout: %s", b.name, e)
            last_err = e
        except APIStatusError as e:
            log.warning("[%s] http %s: %s", b.name, e.status_code, e.message)
            # 4xx (リクエスト不正) はフォールバックしても無駄
            if 400 <= (e.status_code or 0) < 500:
                raise
            last_err = e
    raise RuntimeError("all backends failed") from last_err


if __name__ == "__main__":
    local = build_local()
    cloud = build_cloud()

    messages = [
        {"role": "system", "content": "あなたは日本語アシスタントです。"},
        {"role": "user",   "content": "Foundry Local を 1 文で説明してください。"},
    ]

    print("\n--- 正常系 (local 優先) ---")
    print(chat_with_failover([local, cloud], messages))
```

実行:

```powershell
python failover.py
```

## 4.2.4 フォールバック発火テスト

ローカル側を **わざと止めて** クラウドへフォールバックすることを確認:

```powershell
foundry service stop          # ローカル サービス停止
python failover.py            # → [local] connection/timeout の後 [cloud] OK が出るはず
foundry service start         # 後始末
```

ログ例:

```
WARNING [local] connection/timeout: Connection error.
INFO    [cloud] OK in 1.6s
Foundry Local とは...
```

## 4.2.5 補足: SDK レベルの「不要な再試行」を避ける

- `OpenAI(max_retries=0, timeout=...)` をローカル側に設定し、長く待たずに即フォールバックさせる。
- クラウド側は `max_retries=1` 程度残して、瞬断耐性を持たせる。
- リクエスト不正（400 系）はクラウドで再試行しても通らないので、即エラーを上げる。

## 4.2.6 後始末（Azure リソース削除）

Lab 4 が完了したら、Lab 4.1 で作成したリソースをまとめて削除します。

```powershell
$RG = "rg-foundry-local-hol"

az group delete -n $RG --yes --no-wait
```

完全削除後、念のため:

```powershell
az group exists -n $RG    # false が返れば OK
```

> Foundry/Cognitive Services は「論理削除」されることがあります。同名で再作成したい場合は `az cognitiveservices account purge` も検討してください。

## チェックリスト

- [ ] 正常系でローカル応答が返り、ログに `[local] OK` が出る
- [ ] ローカル停止時にクラウドへフォールバックして応答が返る
- [ ] 4xx エラー時はフォールバックされず即エラーとなることを確認
- [ ] Azure 側 Resource Group を削除した

次へ → [Lab 5.1: Execution Provider 切替と性能計測](../step5/lab5-1-ep.md)
