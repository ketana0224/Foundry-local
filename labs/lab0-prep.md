# Lab 0: 事前準備（環境セットアップ）

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 30 分 |
| 目的 | 以降のハンズオンを実行できる環境を整える |
| Azure | 使用しない |

## 0.1 ハードウェア / OS 要件

| 項目 | 要件 |
| --- | --- |
| OS | Windows 11 24H2 以降（Windows 10 build 26100 以降でも可）。macOS は補足扱い。 |
| メモリ | 16 GB 以上推奨（小型モデルなら 8 GB でも可） |
| ディスク空き | 20 GB 以上（モデル + 実行プロバイダ） |
| GPU | DirectX 12 対応 GPU 推奨（CPU only でも動作可、Lab 5.1 で比較） |
| 権限 | winget でソフトをインストールできる管理者権限 |

> 仮想マシンでは GPU パススルーがない限り WinML ハードウェア アクセラレーションは使えません（CPU フォールバックになります）。

## 0.2 Foundry Local CLI のインストール

PowerShell（管理者）で実行:

```powershell
winget install --id Microsoft.FoundryLocal -e
```

インストール後、ターミナルを **閉じて開き直してから** 確認:

```powershell
foundry --version
foundry --help
```

### `foundry` コマンドが見つからない場合

現在の PowerShell セッションに `%LOCALAPPDATA%\Microsoft\WindowsApps` を追加して再実行:

```powershell
$env:Path = "$env:LOCALAPPDATA\Microsoft\WindowsApps;$env:Path"
foundry --version   # 0.8.119 等が返れば OK
```

## 0.3 サービスの起動確認

```powershell
foundry service status
```

期待出力（例）:

```
Foundry Local service is running.
Endpoint: http://localhost:5273
```

エンドポイントのポート番号は起動ごとに変わる可能性があります（後続 lab で都度参照）。

エラー（`Request to local service failed` 等）が出た場合:

```powershell
foundry service restart
foundry service status
```

## 0.4 開発言語ランタイムの確認

各 Step で使う最低限のランタイムを揃えます。

| ランタイム | 確認コマンド | 必要バージョン |
| --- | --- | --- |
| Python | `python --version` | 3.11 以降 |
| Node.js | `node --version` | 20 以降 |
| .NET SDK | `dotnet --version` | .NET 9 SDK 以降（最低 .NET 8） |
| Git | `git --version` | 2.40 以降推奨 |

不足分のインストール例（PowerShell）:

```powershell
winget install --id Python.Python.3.12 -e
winget install --id OpenJS.NodeJS.LTS -e
winget install --id Microsoft.DotNet.SDK.9 -e
winget install --id Git.Git -e
```

## 0.5 推奨 VS Code 拡張

| 拡張 | 用途 |
| --- | --- |
| Python (Microsoft) | Step 2.1 / 3 / 4 / 5 |
| C# Dev Kit (Microsoft) | Step 2.2 |
| REST Client (humao) | Step 1.2 で `.http` ファイルを使う場合 |

## 0.6 ネットワーク

- 初回 `foundry model run <model>` 実行時にモデル本体と Execution Provider が GitHub Release / Foundry Catalog からダウンロードされます（数百 MB〜数 GB）。
- 企業プロキシ環境ではプロキシ経由ダウンロードに失敗するケースがあるため、検証は社外ネットワークで行うか、IT 部門にドメイン許可を依頼してください。

## 0.7 動作確認（スモークテスト）

最後に最小モデルで一往復チャットして環境を確認します（30 秒〜数分）。

```powershell
foundry model run qwen2.5-0.5b
```

プロンプト `Hello!` を送って応答が返れば OK。`/exit` で終了。

## チェックリスト

- [ ] `foundry --version` がバージョンを返す
- [ ] `foundry service status` が `running` を返す
- [ ] Python / Node.js / .NET / Git が必要バージョンを満たす
- [ ] `qwen2.5-0.5b` で最低 1 ターン会話できた

次へ → [Lab 1.1: モデルカタログと CLI 推論](step1/lab1-1-cli.md)
