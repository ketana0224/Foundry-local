# 自作モデル (BYOM: Bring Your Own Model) を Foundry Local で動かす

Foundry Local は基本的に **Foundry カタログ**（クラウド側のモデルレジストリ）からハードウェア最適化済み ONNX バリアントを取得して実行しますが、**ローカルで自前コンパイルした ONNX モデルも実行できます**。本書は Windows 版 Foundry Local CLI を想定した最短手順です。

## 1. 全体像

```
[Hugging Face / 自前 PyTorch モデル]
        │  Olive で変換 & 量子化
        ▼
[ONNX + genai_config.json + tokenizer.*]
        │  inference_model.json を追加
        ▼
[Foundry Local のキャッシュフォルダ配下に配置]
        │  foundry cache cd / foundry cache list
        ▼
[foundry model run <Name>]
```

## 2. 必要なファイル

ランタイムによって必要ファイルが異なります。

### ONNX Runtime — 生成系 (Chat / Completion)

| ファイル | 必須 | 説明 |
| --- | --- | --- |
| `*.onnx` | ✅ | ONNX モデル本体（1 つ以上） |
| `genai_config.json` | ✅ | onnxruntime-genai 用設定（tokenizer / decoder / search params） |
| `tokenizer.json` | 推奨 | トークナイザー語彙 |
| `tokenizer_config.json` | 推奨 | トークナイザー設定 |
| `inference_model.json` | ✅(Foundry 認識用) | `{ "Name": "<モデル名>" }` |

### ONNX Runtime — 予測系 (分類/回帰)

| ファイル | 必須 | 説明 |
| --- | --- | --- |
| `*.onnx` | ✅ | モデル本体 |
| `inference_model.json` | ✅ | `{ "Name": "<モデル名>" }` |

> 予測系は `genai_config.json` 不要。入力は単一テンソル（rank 4 の画像か通常テンソル）のみ対応。

## 3. Olive で変換する

[Olive](https://github.com/microsoft/olive) は Hugging Face モデル → ONNX 変換 + 量子化 + EP 別最適化を一発で行うツールです。

### インストール（仮想環境推奨）

```powershell
python -m venv .olive
.\.olive\Scripts\Activate.ps1
pip install olive-ai
pip install transformers onnxruntime-genai
olive --help
```

### 変換コマンド例（Llama-3.2-1B を int4 / CPU 向けに）

```bash
olive optimize `
  --model_name_or_path meta-llama/Llama-3.2-1B-Instruct `
  --trust_remote_code `
  --output_path models/llama `
  --device cpu `
  --provider CPUExecutionProvider `
  --precision int4 `
  --log_level 1
```

主なパラメータ:

| Param | 値の例 |
| --- | --- |
| `--model_name_or_path` | HF ID / ローカルパス / Azure AI Model registry ID |
| `--device` | `cpu` / `gpu` / `npu` |
| `--provider` | `CPUExecutionProvider` / `CUDAExecutionProvider` / `NvTensorRTRTXExecutionProvider` / `OpenVINOExecutionProvider` / `WebGpuExecutionProvider` |
| `--precision` | `fp32` / `fp16` / `int8` / `int4` |
| `--output_path` | 出力フォルダ（ここに ONNX とコンフィグが揃う） |

> 💡 自前 PyTorch / Safetensors を変換する場合も、ローカルディレクトリを `--model_name_or_path` に渡せば OK。
> 💡 モデル種別 / ハードウェア毎に最適レシピが [microsoft/olive-recipes](https://github.com/microsoft/olive-recipes) にあるので、最初はレシピを流用するのが最短。

## 4. `inference_model.json` を作成

Foundry Local CLI / SDK にモデルを認識させる目印ファイルです。出力フォルダ直下に置きます。

```powershell
@'
{
  "Name": "llama-3.2:1"
}
'@ | Set-Content -Encoding UTF8 .\models\llama\inference_model.json
```

`Name` の値が `foundry model run <name>` で指定する識別子になります（任意命名、`:1` などのバージョンサフィックス可）。

## 5. キャッシュに登録 → 実行

Foundry Local は既定で `C:\Users\<user>\.foundry\cache\models` 配下を走査します。`foundry cache cd` で親フォルダ自体を切り替えるか、変換成果物を既定キャッシュ配下にコピーしてください。

```powershell
# 例: 自作モデル群を C:\my-models 配下に集約し、ここをキャッシュに指定
foundry cache cd C:\my-models

# 既定の場所に戻したい場合
foundry cache cd "$env:USERPROFILE\.foundry\cache\models"

# 認識確認
foundry cache list

# 実行（inference_model.json の Name と一致させる）
foundry model run llama-3.2:1
```

## 6. SDK から使う場合

SDK では `ModelCacheDir` をモデル親フォルダに向け、`GetCachedModelsAsync()` で列挙します。

```csharp
var config = new Configuration
{
    AppName  = "run-compiled-model",
    LogLevel = LogLevel.Information,
    ModelCacheDir = @"C:\my-models"
};
await FoundryLocalManager.CreateAsync(config, logger);
var mgr = FoundryLocalManager.Instance;
var catalog = await mgr.GetCatalogAsync();
var model   = (await catalog.GetCachedModelsAsync())
              .First(m => m.Id.Contains("llama-3.2:1"));
await model.LoadAsync();
```

(Python / JavaScript SDK でも `modelCacheDir` / `model_cache_dir` で同じことが可能。)

## 7. よくあるハマりどころ

| 症状 | 原因 | 対策 |
| --- | --- | --- |
| `foundry cache list` に出ない | `inference_model.json` が無い / 場所がキャッシュ外 | `foundry cache cd` で親フォルダを指す or ファイル設置 |
| `foundry model run` で EP 不一致エラー | Olive 変換時の `--provider` がマシン未搭載 | `foundry service status` の Valid EPs と一致する EP で再変換 |
| 生成が壊れる / 文字化け | tokenizer 不足 or `genai_config.json` 欠落 | Olive 出力フォルダから 4 ファイル全てコピー |
| 変換時 OOM | フル精度で実行 | `--precision int4` / `int8` に下げる |

## 参考

- [Compile Hugging Face models to run on Foundry Local](https://learn.microsoft.com/azure/foundry-local/how-to/how-to-compile-hugging-face-models)
- [Foundry Local architecture overview](https://learn.microsoft.com/azure/foundry-local/concepts/foundry-local-architecture)
- [Olive Recipes (microsoft/olive-recipes)](https://github.com/microsoft/olive-recipes)
- [Execution Providers (本リポジトリ)](./execution-providers.md)
