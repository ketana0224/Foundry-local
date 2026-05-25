# Foundry-local

Microsoft Foundry Local の検証・ハンズオンコンテンツです。

## ハンズオン構成

| Step | 概要 | リンク |
| --- | --- | --- |
| Lab 0 | 環境セットアップ（CLI / ランタイム） | [labs/lab0-prep.md](labs/lab0-prep.md) |
| Step 1 | CLI と OpenAI 互換 REST API | [labs/step1/](labs/step1/) |
| Step 2 | Python / .NET / JavaScript SDK | [labs/step2/](labs/step2/) |
| Step 3 | Semantic Kernel・LangChain・RAG | [labs/step3/](labs/step3/) |
| Step 4 | Microsoft Foundry (クラウド) との比較・フォールバック | [labs/step4/](labs/step4/) |
| Step 5 | Execution Provider 性能・モデル運用 | [labs/step5/](labs/step5/) |

各ラボは [labs/README.md](labs/README.md) を参照してください。

## Foundry Local とは

### 1. 概要（前提）

**オンデバイス推論用のローカル AI ランタイム**。ONNX Runtime ベースで、**Windows / macOS / Windows Server** 上で動作します。NPU / GPU / CPU を自動選択し、**OpenAI 互換 API** を提供。Azure サブスクリプションは不要で、モデル本体・推論計算はすべて手元のマシン内で完結します。

主な特徴:

- **ハードウェア最適化**: 同一モデルに対し EP (Execution Provider) 別の ONNX バリアント（CPU / CUDA / TensorRT-RTX / OpenVINO / WebGPU / QNN 等）を Foundry カタログから自動取得（詳細 → [docs/execution-providers.md](docs/execution-providers.md)）
- **OpenAI 互換 REST**: `http://127.0.0.1:<port>/v1/chat/completions` 等の OAI 互換エンドポイントをローカルに立てる
- **CLI / SDK 両対応**: `foundry` CLI に加え Python / .NET / JavaScript / Rust SDK を提供
- **オフライン動作**: 初回 DL 以降はネット不要。キャッシュは `foundry cache cd` で任意フォルダに切替可能

### 2. 公式カタログで提供されているモデル

[foundrylocal.ai/models](https://www.foundrylocal.ai/models) で配布される、**量子化・最適化済みの ONNX モデル**です。

| カテゴリ | モデル | 主な用途 |
| --- | --- | --- |
| Microsoft Phi 系 | Phi-4 / Phi-4 mini | チャット・データ分析（標準・軽量） |
|  | Phi-4 Reasoning / Phi-4 mini Reasoning | 数学・コード・科学推論 |
| OpenAI OSS | GPT-OSS 20B | エージェント／推論用途（NVIDIA GPU のみ） |
| Mistral | Mistral 7B Instruct | 汎用チャット |
| Qwen | Qwen2.5（0.5B 等の各サイズ） | テキスト・コード・要約 |
| DeepSeek | DeepSeek R1 | 推論・コーディング・数学 |
| 音声 | Whisper | 音声書き起こし |

すべて INT4 / INT8 等で量子化済みで、ハードウェアに応じて最適なバリアントが自動選択されます。

### 3. 自作モデル（FT モデル含む）の持ち込み — BYOM

Foundry Local は **ONNX 形式であれば任意のモデルをカタログに追加可能**。Microsoft 公式ツール **Olive (olive-ai)** で変換します。詳細手順 → [docs/bring-your-own-model.md](docs/bring-your-own-model.md)。

#### 持ち込みフロー

```
HuggingFace / PyTorch / Safetensors のモデル
        │  olive optimize（変換＋量子化＋最適化）
        ▼
ONNX モデル（CPU/GPU/NPU 向け、int4/int8）
        │  inference_model.json を配置
        ▼
Foundry Local SDK / CLI で読み込み（modelCacheDir 指定 or foundry cache cd）
```

#### 代表的なコマンド

```bash
pip install olive-ai transformers onnxruntime-genai

olive optimize \
  --model_name_or_path meta-llama/Llama-3.2-1B-Instruct \
  --output_path models/llama \
  --device cpu --provider CPUExecutionProvider \
  --precision int4
```

その後、`inference_model.json`（`Name` を指定）を出力フォルダに置けば、SDK の `getCachedModels()` / CLI の `foundry cache list` で認識されます。各モデル向けの推奨設定は [microsoft/olive-recipes](https://github.com/microsoft/olive-recipes) リポジトリにあります。

#### Azure AI Foundry で作った FT モデルとの関係

ここが分かれ目になります。

| FT のベースモデル | Foundry Local で動かせるか |
| --- | --- |
| オープンソース系（Phi / Llama / Mistral / Qwen 等）を Foundry で FT | ⭕ FT 後の重みを取り出し、Olive で ONNX 化すれば持ち込み可能 |
| Azure OpenAI クローズドモデル（GPT-4o, GPT-4.1 等）を FT | ❌ 重みが外に出せないため不可。Azure 上の Standard / Global Standard / PTU デプロイで使う |
| Foundry の Managed Compute（自前 GPU）にデプロイした OSS FT モデル | ⭕ ベースが OSS なら重みを取得 → Olive 経由でローカルへ |

つまり「Foundry で作った FT モデル」と一口に言っても、**ベースが OSS かクローズドか** で Foundry Local 行きの可否が決まります。

## License

このプロジェクトは [MIT License](LICENSE) の下で公開されています。

