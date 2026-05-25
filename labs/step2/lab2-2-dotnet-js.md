# Lab 2.2: .NET / JavaScript SDK 実装

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 45 分 |
| 前提 | [Lab 2.1](lab2-1-python.md) 完了、.NET 9 SDK、Node.js 20 以降 |
| Azure | 使用しない |

Lab 2.1 と同等のチャット（非ストリーミング + ストリーミング）を **.NET** と **Node.js** の両方で実装し、言語間の差分を比較します。時間が足りない場合は片方のみでも可。

---

## 2.2.1 .NET（C#）コンソールアプリ

### プロジェクト作成

```powershell
mkdir labs-work\step2.2\dotnet
cd labs-work\step2.2\dotnet

dotnet new console -n FoundryLocalSample
cd FoundryLocalSample

dotnet add package Microsoft.AI.Foundry.Local --prerelease
dotnet add package OpenAI --prerelease
```

> `Microsoft.AI.Foundry.Local` は Foundry Local の .NET SDK。Public Preview 中のためバージョンは適宜最新を選択。

### `Program.cs`

```csharp
using System.ClientModel;
using Microsoft.AI.Foundry.Local;
using OpenAI;
using OpenAI.Chat;

const string alias = "phi-4-mini";

// 1. Foundry Local サービス起動 + モデル取得
var manager = await FoundryLocalManager.StartModelAsync(aliasOrModelId: alias);
var modelInfo = await manager.GetModelInfoAsync(aliasOrModelId: alias);

Console.WriteLine($"endpoint = {manager.Endpoint}");
Console.WriteLine($"model_id = {modelInfo.ModelId}\n");

// 2. OpenAI 互換クライアント
var chatClient = new OpenAIClient(
        new ApiKeyCredential(manager.ApiKey ?? "not-needed"),
        new OpenAIClientOptions { Endpoint = manager.Endpoint })
    .GetChatClient(modelInfo.ModelId);

// 3. 非ストリーミング
var nonStream = await chatClient.CompleteChatAsync(
    new SystemChatMessage("あなたは簡潔に答える日本語アシスタントです。"),
    new UserChatMessage("Foundry Local とは何ですか？1文で。"));
Console.WriteLine("[non-streaming]");
Console.WriteLine(nonStream.Value.Content[0].Text);

// 4. ストリーミング
Console.WriteLine("\n[streaming]");
await foreach (var update in chatClient.CompleteChatStreamingAsync(
    new UserChatMessage("四国の県を順に説明してください。")))
{
    foreach (var part in update.ContentUpdate)
    {
        Console.Write(part.Text);
    }
}
Console.WriteLine();
```

### 実行

```powershell
dotnet run
```

---

## 2.2.2 Node.js（JavaScript / ESM）

### プロジェクト作成

```powershell
mkdir labs-work\step2.2\node
cd labs-work\step2.2\node

npm init -y
npm pkg set type=module

# Windows（推奨）
npm install foundry-local-sdk-winml openai

# Windows 以外、または HW アクセラレーション不要
# npm install foundry-local-sdk openai
```

### `app.js`

```js
import { FoundryLocalManager } from "foundry-local-sdk";
import OpenAI from "openai";

const ALIAS = "phi-4-mini";

const manager = new FoundryLocalManager();
await manager.startService();
await manager.downloadModel(ALIAS);
await manager.loadModel(ALIAS);

const modelInfo = await manager.getModelInfo(ALIAS);

console.log(`endpoint = ${manager.endpoint}`);
console.log(`model_id = ${modelInfo.id}\n`);

const client = new OpenAI({
  baseURL: manager.endpoint,
  apiKey: manager.apiKey ?? "not-needed",
});

// 1. 非ストリーミング
const nonStream = await client.chat.completions.create({
  model: modelInfo.id,
  temperature: 0.2,
  max_tokens: 200,
  messages: [
    { role: "system", content: "あなたは簡潔に答える日本語アシスタントです。" },
    { role: "user",   content: "Foundry Local とは何ですか？1文で。" },
  ],
});
console.log("[non-streaming]");
console.log(nonStream.choices[0].message.content, "\n");

// 2. ストリーミング
console.log("[streaming]");
const stream = await client.chat.completions.create({
  model: modelInfo.id,
  stream: true,
  messages: [{ role: "user", content: "四国の県を順に説明してください。" }],
});
for await (const part of stream) {
  process.stdout.write(part.choices[0]?.delta?.content ?? "");
}
process.stdout.write("\n");
```

### 実行

```powershell
node app.js
```

---

## 2.2.3 言語別の比較ポイント

| 観点 | Python | .NET | JavaScript |
| --- | --- | --- | --- |
| パッケージ名 | `foundry-local-sdk(-winml)` + `openai` | `Microsoft.AI.Foundry.Local` + `OpenAI` | `foundry-local-sdk(-winml)` + `openai` |
| サービス起動 | `FoundryLocalManager(alias)` 一発 | `StartModelAsync(alias)` | `startService` + `downloadModel` + `loadModel` |
| モデル ID 解決 | `manager.get_model_info(alias).id` | `manager.GetModelInfoAsync(alias).ModelId` | `manager.getModelInfo(alias).id` |
| ストリーミング | `for chunk in stream:` | `await foreach (var update ...)` | `for await (const part of stream)` |

3 言語とも「**SDK で endpoint と modelId を取得し、OpenAI クライアントに渡す**」という同じ構造であることを確認できれば成功です。

## チェックリスト

- [ ] .NET / Node.js のいずれかで非ストリーミング応答が返る
- [ ] 同じくストリーミング応答が逐次出力される
- [ ] Python 版（Lab 2.1）と同じモデル ID が使われている

次へ → [Lab 3.1: Microsoft Agent Framework / LangChain 連携](../step3/lab3-1-sk-langchain.md)
