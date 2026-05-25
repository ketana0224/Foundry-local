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

**恒久対応**: 上記は現在のセッション限定です。新しいターミナルでも有効にするには、ユーザー PATH に追加します（`setx` は既存 PATH を 1024 文字で切り詰めるリスクがあるため `[Environment]::SetEnvironmentVariable` を使用）:

```powershell
$add = "$env:LOCALAPPDATA\Microsoft\WindowsApps"
$old = [Environment]::GetEnvironmentVariable('Path', 'User')
if (-not (($old -split ';') -contains $add)) {
    [Environment]::SetEnvironmentVariable('Path', "$add;$old", 'User')
    Write-Host "Added to User PATH." -ForegroundColor Yellow
} else {
    Write-Host "Already present."
}
```

設定後は **Windows Terminal / VS Code を完全に終了して開き直す** こと（タブを閉じるだけでは親プロセスが古い PATH をキャッシュしたままなので反映されません）。それでも反映されない場合は一度サインアウト → サインインしてください。

## 0.3 サービスの起動

```powershell
foundry service start
```

**新規起動時** の出力（例）:

```
🟢 Service is Started on http://127.0.0.1:51296/, PID 38200!
```

**既に起動済み** の場合の出力（例）:

```
🟢 Service is already running on http://127.0.0.1:59207/.
```

どちらが表示されても OK です。**表示されたエンドポイント URL のポート番号を控えておきます**（後続 lab で `http://127.0.0.1:<port>/v1` を OpenAI 互換 base URL として使用します）。ポート番号は新規起動の度に変わるため、毎セッション確認してください。

サービスの状態確認:

```powershell
foundry service status
```

出力（例）:

```
🟢 Model management service is running on http://127.0.0.1:51296/openai/status
EP autoregistration status: Successfully downloaded and registered the following EPs: NvTensorRTRTXExecutionProvider, OpenVINOExecutionProvider, CUDAExecutionProvider.
Valid EPs: CPUExecutionProvider, WebGpuExecutionProvider, NvTensorRTRTXExecutionProvider, OpenVINOExecutionProvider, CUDAExecutionProvider
```

`Valid EPs:` 行に列挙された実行プロバイダー（CPU / WebGPU / CUDA / OpenVINO / NvTensorRT 等）が、このマシンでモデル推論に使える EP の一覧です（Step 5 で活用）。各 EP の特徴・使い分けは [../docs/execution-providers.md](../docs/execution-providers.md) を参照してください。

確実にリセットしたい場合（ポートを変えたい / プロセスを掃除したい）:

```powershell
foundry service stop
foundry service start
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

初回はモデル本体（数百 MB）のダウンロードが入ります。プロンプトが出たら `Hello!` を送って応答が返れば OK。`/exit` で終了。

キャッシュ管理コマンド:

```powershell
foundry cache list                # ダウンロード済みモデル一覧
```

```powershell
foundry cache location            # 現在のキャッシュディレクトリパス表示
```

```powershell
foundry cache remove <model>      # モデル削除
```

```powershell
foundry cache cd <path>           # キャッシュディレクトリの場所変更
```

サービス停止（モデルもアンロード）:

```powershell
foundry service stop
```

> 💡 `foundry model run` / `foundry cache list` 実行時に  
> `[ERR] Failed to process model #0 on page N.`  
> という赤文字エラーが複数行出ることがありますが **無視して構いません**。これは Foundry のリモートカタログ API をページ送りで取得する際、CLI 側パーサがまだ知らない新しいスキーマのモデルエントリをスキップしているだけのノイズ警告で、機能には影響しません（Fresh Install でも発生）。気になる場合は `winget upgrade --id Microsoft.FoundryLocal` で最新クライアントに更新してください。

## チェックリスト

- [ ] `foundry --version` がバージョンを返す
- [ ] `foundry service start` で 🟢 表示（`Service is Started ...` または `Service is already running ...`）
- [ ] エンドポイント URL（`http://127.0.0.1:<port>/`）を 1 回控えた
- [ ] Python / Node.js / .NET / Git が必要バージョンを満たす
- [ ] `qwen2.5-0.5b` で最低 1 ターン会話できた

次へ → [Lab 1.1: モデルカタログと CLI 推論](step1/lab1-1-cli.md)
