# Lab 5.2: モデル追加 / 容量管理

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 40 分 |
| 前提 | [Lab 1.1](../step1/lab1-1-cli.md) 完了 |
| Azure | 使用しない |

複数モデルを試すと容易にディスクが圧迫されます。本 lab では **追加 / 切替 / 削除 / 容量モニタ** の運用フローと、**カスタム ONNX モデル** の取り込み手順を整理します。

## 5.2.1 キャッシュの場所とサイズ

```powershell
foundry cache location
$cache = (foundry cache location) -replace '^Cache location:\s*',''
"$cache"

# 全体サイズ
"{0:N2} GB" -f ((Get-ChildItem $cache -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum / 1GB)

# トップディレクトリ別 (上位 10 件)
Get-ChildItem $cache -Directory | ForEach-Object {
    $bytes = (Get-ChildItem $_.FullName -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    [PSCustomObject]@{ Path = $_.Name; SizeGB = [math]::Round($bytes / 1GB, 2) }
} | Sort-Object SizeGB -Descending | Select-Object -First 10 | Format-Table
```

## 5.2.2 ロード済みモデルとカタログ済みモデル

```powershell
foundry model ls       # ローカルキャッシュにあるモデル
foundry model running  # 現在ロード（起動）中のモデル
```

> 起動中モデルはメモリを占有するため、不要なものは `foundry model stop <id>` で停止します。

## 5.2.3 モデルの追加（カタログから）

```powershell
foundry model download qwen2.5-7b-instruct
foundry model run qwen2.5-7b-instruct
```

ダウンロードのみ済ませて後で起動したい場合は `download` のみで OK。

## 5.2.4 モデル/キャッシュの削除

特定モデルの削除:

```powershell
foundry cache remove qwen2.5-7b-instruct
```

特定 variant のみ削除:

```powershell
foundry cache remove phi-4-mini-instruct-gpu-int4-rtn-block-32-acc-level-4
```

未使用の Execution Provider なども含めて掃除（破壊的なので注意）:

```powershell
foundry cache clear   # キャッシュをまとめて消去
```

> `foundry cache clear` は次回利用時に再 DL になります。検証 PC 以外では慎重に。

## 5.2.5 カスタム / 自前 ONNX モデルの取り込み

Foundry Local は HuggingFace のモデルを **ONNX に変換 + 量子化** することで取り込めます。

> 詳細手順は [Compile Hugging Face models to run on Foundry Local](https://learn.microsoft.com/azure/foundry-local/how-to/how-to-compile-hugging-face-models) を参照（コマンド変更が頻繁なため、本 lab では概念のみ）。

代表的な流れ:

1. HuggingFace から target モデルを取得
2. Olive または `onnxruntime-genai` ツールで ONNX 形式へ変換
3. `int4` 等の量子化を適用
4. Foundry Local のローカルカタログにモデルマニフェスト（`genai_config.json` 等を含むディレクトリ）を登録
5. `foundry model run <local-alias>` で実行

簡易確認: 既存のローカルディレクトリを直接実行することも可能 (バージョンによる)。

```powershell
foundry model run --model-path C:\models\my-onnx-model
```

成功すると、カタログモデルと同様にチャットできます。

## 5.2.6 運用ベストプラクティス

| 項目 | 推奨 |
| --- | --- |
| 同時起動モデル数 | 1〜2 個に絞る（メモリ/VRAM 圧迫防止） |
| 量子化 | 個人 PC は `int4` を第一候補。品質不足なら `fp16` |
| バックグラウンド占有 | 使い終わったら `foundry model stop` |
| キャッシュ場所 | デフォルトはユーザプロファイル配下。大容量ドライブに移したい場合は環境変数 `FOUNDRY_CACHE_DIR` を設定（リファレンス参照） |
| 監視 | `foundry cache ls` をスクリプト化して定期実行、閾値超えで通知 |

## 5.2.7 後始末（任意）

ハンズオン用に DL したモデルが不要なら削除しておく:

```powershell
foundry cache remove qwen2.5-7b-instruct
foundry cache remove phi-4-mini
# 必要に応じて Execution Provider もまとめて
# foundry cache clear
```

## チェックリスト

- [ ] キャッシュ場所と現在のサイズを確認できた
- [ ] モデル追加 → 起動 → 停止 → 削除の一連を実行できた
- [ ] カスタム ONNX モデル取り込みの全体像を把握できた

これで全 Lab 完了です。お疲れさまでした。
