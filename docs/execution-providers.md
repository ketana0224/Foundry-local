# ONNX Runtime Execution Provider (EP) リファレンス

Microsoft Foundry Local は内部で **ONNX Runtime (ORT)** を使ってモデルを推論します。`foundry service status` の `Valid EPs:` 行は、現在のマシンで利用可能な **Execution Provider (EP)** = どのハードウェア／アクセラレータ経路で推論計算を実行できるかの一覧です。

## Execution Provider とは

ONNX Runtime はモデル（計算グラフ）を受け取り、各ノード（演算）をどのハードウェアで実行するかを EP に委任します。EP は GPU / CPU / NPU / 専用アクセラレータごとに用意され、最適な経路を選ぶことでレイテンシ・スループット・メモリ効率が大きく変わります。

Foundry Local 起動時の **EP autoregistration** は、ハードウェア（GPU ベンダー、ドライバ、SDK の有無）を検出して、使える EP のバイナリを自動ダウンロード・登録するプロセスです。

## 主要 EP 一覧

| EP | 対象ハードウェア | バックエンド | 特徴 |
| --- | --- | --- | --- |
| `CPUExecutionProvider` | CPU（全環境） | ORT 標準（MLAS） | 常に使える既定。INT4 / INT8 量子化モデルで実用速度。GPU 非搭載マシンのフォールバック |
| `WebGpuExecutionProvider` | 任意 GPU（NV / AMD / Intel / Apple） | WebGPU (Dawn / wgpu) | ベンダーニュートラル GPU 経路。ドライバ固有 SDK 不要。性能は CUDA / DirectML より一段落ちるが配布が容易 |
| `CUDAExecutionProvider` | **NVIDIA GPU** | CUDA + cuDNN | 定番。FP16 / INT8 量子化対応、安定 |
| `NvTensorRTRTXExecutionProvider` | **NVIDIA RTX GPU** (Ada / Blackwell 等) | TensorRT for RTX | RTX 専用の最適化版 TensorRT。`CUDA` より一般に高速（カーネル融合、FP8 / INT4 サポート、グラフ最適化）。RTX 40 / 50 系で本領を発揮 |
| `OpenVINOExecutionProvider` | **Intel CPU / iGPU / NPU**（Core Ultra など） | OpenVINO Runtime | Intel 系専用最適化。Core Ultra の **NPU**（AI Boost）にもオフロード可能。低消費電力・ノート PC で強い |
| `DmlExecutionProvider`（参考） | 全 GPU（DirectX 12 対応） | DirectML | Windows 標準。NVIDIA 以外の GPU でも使える汎用 GPU 経路 |
| `QNNExecutionProvider`（参考） | Qualcomm Snapdragon X NPU | QNN SDK | Snapdragon 搭載 Copilot+ PC 向け |

> ⚠ 利用可能な EP はハードウェア／OS／ドライバの状態で変動します。`Valid EPs:` の一覧が実機の正解です。

## 推論時の自動選択ロジック

Foundry Local のモデルカタログには、各モデルごとに **どの EP で動く ONNX バリアントが用意されているか** が登録されています。例えば `phi-3.5-mini` には `cpu-int4` / `cuda-fp16` / `tensorrt-rtx-fp8` のような派生があり、利用可能な EP と組み合わせて最適なバリアントが自動選択されます。

優先順位はおおむね（高速 → 互換重視）:

1. **NvTensorRTRTXExecutionProvider**（RTX があるとき）
2. **CUDAExecutionProvider**（NVIDIA GPU）
3. **OpenVINOExecutionProvider**（Intel NPU / iGPU、低消費電力で長文向き）
4. **DmlExecutionProvider**（汎用 GPU、ある場合）
5. **WebGpuExecutionProvider**
6. **CPUExecutionProvider**（最後の砦）

明示指定する CLI 例:

```powershell
# モデル一覧で利用可能なバリアントを確認
foundry model info phi-3.5-mini

# 特定 EP で動かす（バリアント指定）
foundry model run phi-3.5-mini --variant cuda-fp16
foundry model run phi-3.5-mini --variant cpu-int4
```

## ユースケース別の選び方

| ケース | 推奨 EP | 理由 |
| --- | --- | --- |
| 開発中の高速チャット（RTX 環境） | `NvTensorRTRTXExecutionProvider` | 最高速 |
| バッチ評価（並列度高） | `CUDAExecutionProvider` | TensorRT のセッションオーバーヘッド回避 |
| ノート PC で長時間稼働 | `OpenVINOExecutionProvider`（NPU） | 低消費電力 |
| GPU を別作業で使用中 | `CPUExecutionProvider`（INT4） | リソース競合回避 |
| ブラウザ拡張 / Electron | `WebGpuExecutionProvider` | ドライバ非依存 |
| NVIDIA 以外の GPU を使いたい | `DmlExecutionProvider` | AMD / Intel GPU でも GPU 推論可 |

## Step 5 でのベンチマーク観点

[../labs/step5/lab5-1-ep.md](../labs/step5/lab5-1-ep.md) では、同一モデルを別 EP で順番に走らせて以下を比較します:

- **First-token latency**（プロンプト処理開始 → 最初のトークン）
- **Throughput**（tokens / sec、ストリーミング応答の平均）
- **メモリ使用量**（VRAM / RAM、`nvidia-smi` / タスクマネージャ）
- **電力**（NPU の優位性確認、HWiNFO / LibreHardwareMonitor 等）

## 参考リンク

- ONNX Runtime Execution Providers: <https://onnxruntime.ai/docs/execution-providers/>
- TensorRT for RTX 概要: <https://developer.nvidia.com/tensorrt-rtx>
- OpenVINO + NPU ドキュメント: <https://docs.openvino.ai/>
- DirectML 概要: <https://learn.microsoft.com/windows/ai/directml/dml>
