# Lab 4.1: Microsoft Foundry (クラウド) との比較

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 45 分 |
| 前提 | [Lab 2.1](../step2/lab2-1-python.md) 完了、Azure CLI 2.60+、サブスクリプション権限 (Cognitive Services Contributor) |
| Azure | Microsoft Foundry リソース 1 つ（東日本、最小構成） |
| 想定コスト | 数十円〜数百円程度（gpt-4o-mini / phi-4-mini を数回呼ぶ程度。**lab 終了時に必ず削除**） |

クラウド側 Microsoft Foundry リソースを最小構成で作成し、ローカル（Foundry Local）と **同じ OpenAI クライアントコード** で同一プロンプトを実行 → 応答品質と所要時間を比較します。

## 4.1.1 サブスクリプション / ログイン

```powershell
az login --tenant 05891013-f760-4c23-b235-5479cc057a9d
az account set --subscription 9353f1a1-94a4-4e4b-ae82-c27ea3d07160
az account show -o table
```

## 4.1.2 リソース作成（東日本・最小構成）

```powershell
$RG       = "rg-foundry-local-hol"
$LOCATION = "japaneast"
$ACCOUNT  = "fnd-loc-hol-$((Get-Random -Maximum 9999))"
$DEPLOY   = "gpt-4o-mini"
$MODEL    = "gpt-4o-mini"
$MODEL_VER = "2024-07-18"  # 利用可能な最新版は az cognitiveservices model list で確認可
$TAGS     = "purpose=foundry-local-hol owner=$env:USERNAME"

# 1. リソースグループ
az group create -n $RG -l $LOCATION --tags $TAGS.Split() | Out-Null

# 2. Microsoft Foundry (AIServices) アカウント (kind=AIServices)
az cognitiveservices account create `
  --name $ACCOUNT --resource-group $RG --location $LOCATION `
  --kind AIServices --sku S0 `
  --custom-domain $ACCOUNT `
  --tags $TAGS.Split() | Out-Null

# 3. モデルデプロイ（最小 SKU = GlobalStandard 10K TPM 程度）
az cognitiveservices account deployment create `
  --resource-group $RG --name $ACCOUNT `
  --deployment-name $DEPLOY `
  --model-name $MODEL --model-version $MODEL_VER --model-format OpenAI `
  --sku-name GlobalStandard --sku-capacity 10 | Out-Null
```

> モデル / SKU は地域や時期で変化します。失敗時は `az cognitiveservices model list -l japaneast --kind AIServices -o table` で利用可能な組み合わせを確認してください。

## 4.1.3 endpoint / キー取得

```powershell
$CLOUD_ENDPOINT = (az cognitiveservices account show -n $ACCOUNT -g $RG --query "properties.endpoints.\"OpenAI Language Model Instance API\"" -o tsv)
if (-not $CLOUD_ENDPOINT) {
  # 古い API バージョン用フォールバック
  $CLOUD_ENDPOINT = (az cognitiveservices account show -n $ACCOUNT -g $RG --query "properties.endpoint" -o tsv) + "openai/v1"
}
$CLOUD_KEY = (az cognitiveservices account keys list -n $ACCOUNT -g $RG --query "key1" -o tsv)

"FOUNDRY_CLOUD_ENDPOINT = $CLOUD_ENDPOINT"
"FOUNDRY_CLOUD_MODEL    = $DEPLOY"
"FOUNDRY_CLOUD_KEY      = (取得済み)"
```

## 4.1.4 比較スクリプト

`compare.py`:

```python
import os
import time

from foundry_local import FoundryLocalManager
from openai import OpenAI

ALIAS = "phi-4-mini"

# ローカル
local_mgr = FoundryLocalManager(ALIAS)
local = OpenAI(base_url=local_mgr.endpoint, api_key=local_mgr.api_key or "x")
local_model = local_mgr.get_model_info(ALIAS).id

# クラウド (Microsoft Foundry / OpenAI v1 互換 endpoint)
cloud = OpenAI(
    base_url=os.environ["FOUNDRY_CLOUD_ENDPOINT"],
    api_key=os.environ["FOUNDRY_CLOUD_KEY"],
)
cloud_model = os.environ["FOUNDRY_CLOUD_MODEL"]


PROMPTS = [
    "Foundry Local を 1 文で説明してください。",
    "四国の県を順に説明してください。",
    "Python で 1〜100 までの素数を返す関数を書いてください。",
]


def run(label, client, model):
    print(f"\n=== {label} ({model}) ===")
    for p in PROMPTS:
        t0 = time.perf_counter()
        r = client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": p}],
            temperature=0.0, max_tokens=400,
        )
        dt = time.perf_counter() - t0
        usage = r.usage
        print(f"\n[Q] {p}\n[A({dt:.1f}s, in={usage.prompt_tokens} out={usage.completion_tokens})]")
        print(r.choices[0].message.content)


run("local",  local,  local_model)
run("cloud",  cloud,  cloud_model)
```

実行:

```powershell
$env:FOUNDRY_CLOUD_ENDPOINT = $CLOUD_ENDPOINT
$env:FOUNDRY_CLOUD_KEY      = $CLOUD_KEY
$env:FOUNDRY_CLOUD_MODEL    = $DEPLOY

python compare.py
```

## 4.1.5 比較観点

実行結果を以下の観点で観察します。

| 観点 | ローカル (phi-4-mini) | クラウド (gpt-4o-mini) |
| --- | --- | --- |
| レイテンシ（初回 / 2 回目） | 初回はモデルロード分が加算、定常はネットワーク 0 で速いケースあり | 一定、ネットワーク往復が支配的 |
| 出力品質（推論・コード） | 軽量モデル相応 | より長く、構造化された回答 |
| プライバシ | 完全ローカル、外部に出ない | クラウド送信 |
| コスト | 0 円 / 電気代のみ | 従量課金（このサンプル数なら 1 円〜数十円） |
| オフライン可否 | ◯ | ✕ |

## 4.1.6 後処理

リソースは Lab 4.2 でもそのまま使うため、**Lab 4.2 終了後に削除** します。今回はそのまま残しておきます。  
今すぐ削除したい場合は Lab 4.2 末尾の削除手順を実施。

## チェックリスト

- [ ] クラウド側 Foundry リソースとモデルデプロイが作成された
- [ ] `compare.py` でローカル・クラウド両方の応答とトークン使用量が出力された
- [ ] レイテンシ・品質の差を体感し、比較表に書き込めた

次へ → [Lab 4.2: ローカル優先 + クラウドフォールバック](lab4-2-failover.md)
