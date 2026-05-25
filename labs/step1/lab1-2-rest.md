# Lab 1.2: OpenAI 互換 REST エンドポイント

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 40 分 |
| 前提 | [Lab 1.1](lab1-1-cli.md) 完了 |
| Azure | 使用しない |

Foundry Local が提供する OpenAI 互換 REST API を `curl` / PowerShell / REST Client から直接呼び出します。

## 1.2.1 エンドポイントとモデル名の確認

```powershell
foundry service status
```

出力例:

```
Foundry Local service is running.
Endpoint: http://localhost:5273
```

> ポート番号はサービス起動ごとに変わるため、必ず実行時に確認します。以後 `$ENDPOINT` と `$MODEL` を変数として使います。

```powershell
$ENDPOINT = "http://localhost:5273"   # 自分の出力に置き換える
```

モデルが起動していなければ、別ターミナルで:

```powershell
foundry model run qwen2.5-0.5b      # Lab 0 で DL 済み。品質重視なら phi-4-mini 等に差し替え
```

起動中モデルの **Model ID**（リクエストに渡す `model` の値）を取得:

```powershell
(Invoke-RestMethod "$ENDPOINT/v1/models").data | Format-Table id, owned_by, maxInputTokens
```

複数モデルが同時にロードされている場合があるので、**使いたいベース名で 1 件に絞ります**。Lab 0 で DL 済みの軽量モデルを使う例:

```powershell
$MODEL = ((Invoke-RestMethod "$ENDPOINT/v1/models").data |
          Where-Object { $_.id -like "qwen2.5-0.5b*" }).id |
         Select-Object -First 1
"Using model: $MODEL"
```

品質を上げたい場合は `phi-4-mini` に切り替え:

```powershell
$MODEL = ((Invoke-RestMethod "$ENDPOINT/v1/models").data |
          Where-Object { $_.id -like "Phi-4-mini*" }).id |
         Select-Object -First 1
```

出力例（マシンによって EP 部分が変わる）:

```
Using model: qwen2.5-0.5b-instruct-trtrtx-gpu:2
```

> ⚠️ **REST 直叩きでは Model ID が必須**。`foundry` CLI / SDK は `phi-4-mini` のような **エイリアス** を内部で Model ID に解決しますが、`/v1/chat/completions` にエイリアスをそのまま渡すと `400 Bad Request` になります。必ず `/v1/models` で返った `id`（例: `Phi-4-mini-instruct-cuda-gpu:5`）を使ってください。Model ID の読み方は [Lab 1.1 § 1.1.2](lab1-1-cli.md#112-モデルカタログの一覧) 参照。

## 1.2.2 chat.completions（非ストリーミング）

```powershell
$body = @{
  model    = $MODEL
  messages = @(
    @{ role = "system"; content = "あなたは簡潔に答える日本語アシスタントです。" },
    @{ role = "user";   content = "Microsoft Foundry Local とは何ですか？1 文で説明してください。" }
  )
  temperature = 0.2
  max_tokens  = 200
} | ConvertTo-Json -Depth 5

Invoke-RestMethod -Uri "$ENDPOINT/v1/chat/completions" `
  -Method Post -ContentType "application/json" -Body $body |
  ConvertTo-Json -Depth 5
```

応答に `choices[0].message.content` が含まれることを確認します。

## 1.2.3 chat.completions（ストリーミング SSE）

ストリーミングは PowerShell では扱いにくいので `curl` を使います。

```powershell
$json = @"
{
  "model": "$MODEL",
  "stream": true,
  "messages": [
    {"role":"user","content":"四国の県を順に説明してください。"}
  ]
}
"@

$json | Out-File -FilePath body.json -Encoding utf8 -NoNewline

curl -N -X POST "$ENDPOINT/v1/chat/completions" `
  -H "Content-Type: application/json" `
  --data-binary "@body.json"
```

`data: {...}` の連続行と最後に `data: [DONE]` が出れば成功。

> 💡 SSE のペイロード内では日本語が `\u3067\u3059` のような **JSON Unicode エスケープ** で返ります（OpenAI 互換 API の仕様通りで、文字化けではありません）。ターミナルで自然な日本語として読みたい場合は、`curl` 出力を PowerShell でパースしながら表示します:

```powershell
# 文字化け対策：コンソール出力を UTF-8 にしておく（このセッション中だけ有効）
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

curl -N -s -X POST "$ENDPOINT/v1/chat/completions" `
  -H "Content-Type: application/json" `
  --data-binary "@body.json" |
  ForEach-Object {
    if ($_ -match '^data:\s*(\{.*\})\s*$') {
      $chunk = $matches[1] | ConvertFrom-Json
      Write-Host -NoNewline $chunk.choices[0].delta.content
    }
  }
Write-Host ""   # 末尾改行
```



## 1.2.4 VS Code REST Client 用ファイル（任意）

`samples/rest/foundry-local.http` を作成:

```http
@endpoint = http://localhost:5273
# /v1/models で確認した Model ID をそのまま貼り付ける（エイリアス不可）。例：
@model = qwen2.5-0.5b-instruct-trtrtx-gpu:2
# @model = Phi-4-mini-instruct-cuda-gpu:5

### List models
GET {{endpoint}}/v1/models

### Chat completion
POST {{endpoint}}/v1/chat/completions
Content-Type: application/json

{
  "model": "{{model}}",
  "messages": [
    {"role": "system", "content": "あなたは簡潔に答える日本語アシスタントです。"},
    {"role": "user",   "content": "Microsoft Foundry Local とは何ですか？"}
  ],
  "temperature": 0.2,
  "max_tokens": 200
}

### Streaming
POST {{endpoint}}/v1/chat/completions
Content-Type: application/json

{
  "model": "{{model}}",
  "stream": true,
  "messages": [
    {"role": "user", "content": "四国の県を順に説明してください。"}
  ]
}
```

VS Code 上で `Send Request` をクリックして動作確認。

## 1.2.5 補足: Open WebUI からの利用（任意）

ブラウザ UI で試したい場合は [Open WebUI](https://github.com/open-webui/open-webui) を起動し、Direct Connections に `http://localhost:<PORT>/v1`（Auth: None）を登録します。詳細は [Foundry Local CLI reference](https://learn.microsoft.com/azure/foundry-local/reference/reference-cli#use-open-webui-with-the-local-server) 参照。

## チェックリスト

- [ ] `GET /v1/models` で起動中モデルが返る
- [ ] 非ストリーミング chat.completions が JSON で応答する
- [ ] ストリーミング chat.completions が SSE で逐次出力される

次へ → [Lab 2.1: Python SDK でチャット & ストリーミング](../step2/lab2-1-python.md)
