# Lab 1.1: モデルカタログと CLI 推論

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 30 分 |
| 前提 | [Lab 0](../lab0-prep.md) 完了 |
| Azure | 使用しない |

CLI から Foundry Local のモデルカタログを確認し、ローカルでチャット推論を実行します。

## 1.1.1 サービス確認

```powershell
foundry service status
```

`running` でなければ:

```powershell
foundry service start
```

## 1.1.2 モデルカタログの一覧

```powershell
foundry model list
```

初回はハードウェア向けの Execution Provider がダウンロードされます（数百 MB）。

フィルタ例:

```powershell
foundry model list --filter device=GPU
foundry model list --filter task=chat-completion
```

### Model ID の読み方

**Model ID** は、Foundry カタログ上で 1 つのバリアントを一意に識別する完全名です。例えば `Phi-4-mini-instruct-cuda-gpu:5` は次のように分解できます。

| 部分 | 例 | 意味 |
| --- | --- | --- |
| ベースモデル名 | `Phi-4-mini-instruct` | モデルファミリ + 規模 + チューニング種別 (instruct / reasoning など) |
| EP / HW ターゲット | `cuda-gpu` | 対応 Execution Provider とハードウェア（`cuda-gpu` / `trtrtx-gpu` / `openvino-gpu` / `qnn-npu` / `generic-cpu` 等） |
| バージョン | `:5` | このバリアントのカタログ上のリビジョン番号 |

同じ `phi-4-mini` エイリアスでも、マシン環境によって実際にロードされる Model ID が変わります。

| マシン環境 | 解決される Model ID 例 |
| --- | --- |
| NVIDIA GPU + CUDA EP | `Phi-4-mini-instruct-cuda-gpu:N` |
| NVIDIA RTX + TensorRT-RTX EP | `Phi-4-mini-instruct-trtrtx-gpu:N` |
| Intel/AMD + OpenVINO | `Phi-4-mini-instruct-openvino-gpu:N` |
| Snapdragon NPU | `Phi-4-mini-instruct-qnn-npu:N` |
| CPU フォールバック | `Phi-4-mini-instruct-generic-cpu:N` |

使い分け:

- **エイリアス指定**（`foundry model run phi-4-mini`） → Valid EPs と突き合わせて最適バリアントを自動選択。通常はこちらでよい。
- **Model ID 直指定**（`foundry model run Phi-4-mini-instruct-cuda-gpu:5`） → バリアントを固定したい場合（再現性確保、特定 EP の性能ベンチ等）。

## 1.1.3 モデル情報の確認

```powershell
foundry model info phi-4-mini
```

ライセンス確認:

```powershell
foundry model info phi-4-mini --license
```

> Tip: モデル指定は **エイリアス**（`phi-4-mini` など）が推奨。Foundry Local がハードウェアに最適な variant（CPU / CUDA / NPU など）を自動選択します。

## 1.1.4 モデルを起動してチャット

```powershell
foundry model run phi-4-mini
```

初回はモデル本体（数 GB）のダウンロードあり。プロンプトに以下を順に入力:

```
日本の首都はどこですか？
四国の県を列挙してください。
```

`/exit` で終了。

軽量モデル（最初の動作確認用）:

```powershell
foundry model run qwen2.5-0.5b
```

## 1.1.5 ロード済みモデルとリソースの確認

別ターミナルで:

```powershell
foundry model ls           # ローカルにキャッシュ済みのモデル
foundry cache ls           # キャッシュ全体（モデル + EP）
foundry cache location     # キャッシュフォルダのパス
```

ディスク使用量の概算:

```powershell
$cache = (foundry cache location) -replace 'Cache location: ',''
"{0:N2} GB" -f ((Get-ChildItem $cache -Recurse | Measure-Object Length -Sum).Sum / 1GB)
```

## 1.1.6 モデルの停止・削除

実行中モデルの停止:

```powershell
foundry model stop phi-4-mini
```

ローカルキャッシュからモデルを削除（容量を空けたいとき）:

```powershell
foundry cache remove phi-4-mini
```

> Lab 5.2 で容量管理を詳しく扱います。ここでは「削除できる」ことだけ確認すれば十分。

## チェックリスト

- [ ] `foundry model list` がカタログを表示する
- [ ] `phi-4-mini` または `qwen2.5-0.5b` でチャット応答が返った
- [ ] `foundry model ls` でローカルキャッシュにモデルが登録されている

次へ → [Lab 1.2: OpenAI 互換 REST エンドポイント](lab1-2-rest.md)
