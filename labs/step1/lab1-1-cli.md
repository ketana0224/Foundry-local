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
