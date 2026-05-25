# Foundry Local ハンズオン

Microsoft Foundry Local（ローカル AI 推論ランタイム）の検証・ハンズオンコンテンツ。

> 全体計画は [`../spec.md`](../spec.md) を参照。

## 構成

| # | タイトル | ファイル | 想定時間 | Azure |
| --- | --- | --- | --- | --- |
| Lab 0 | 事前準備（インストール・環境確認） | [lab0-prep.md](lab0-prep.md) | 30 分 | なし |
| Lab 1.1 | モデルカタログと CLI 推論 | [step1/lab1-1-cli.md](step1/lab1-1-cli.md) | 30 分 | なし |
| Lab 1.2 | OpenAI 互換 REST エンドポイント | [step1/lab1-2-rest.md](step1/lab1-2-rest.md) | 40 分 | なし |
| Lab 2.1 | Python SDK でチャット & ストリーミング | [step2/lab2-1-python.md](step2/lab2-1-python.md) | 45 分 | なし |
| Lab 2.2 | .NET / JavaScript SDK 実装 | [step2/lab2-2-dotnet-js.md](step2/lab2-2-dotnet-js.md) | 45 分 | なし |
| Lab 3.1 | Semantic Kernel / LangChain 連携 | [step3/lab3-1-sk-langchain.md](step3/lab3-1-sk-langchain.md) | 50 分 | なし |
| Lab 3.2 | ローカル RAG（Embedding + ベクトル検索） | [step3/lab3-2-rag.md](step3/lab3-2-rag.md) | 50 分 | なし |
| Lab 4.1 | Microsoft Foundry (クラウド) との比較 | [step4/lab4-1-foundry-cloud.md](step4/lab4-1-foundry-cloud.md) | 45 分 | 最小 SKU |
| Lab 4.2 | ローカル優先 + クラウドフォールバック | [step4/lab4-2-failover.md](step4/lab4-2-failover.md) | 50 分 | Lab 4.1 流用 |
| Lab 5.1 | Execution Provider 切替と性能計測 | [step5/lab5-1-ep.md](step5/lab5-1-ep.md) | 45 分 | なし |
| Lab 5.2 | モデル追加 / 容量管理 | [step5/lab5-2-models.md](step5/lab5-2-models.md) | 40 分 | なし |

## 共通前提

- OS: Windows 11 24H2 以降（Mac は補足）
- ハードウェア: メモリ 16GB 以上、DirectX 12 対応 GPU 推奨（CPU only でも可）
- ネットワーク: 初回モデル DL のため必須
- Azure: Lab 4 のみ使用（`purpose=foundry-local-hol` タグ付け / 終了時に Resource Group 削除）

## ハンズオンの進め方

1. Lab 0 を実施して環境を整える
2. Step 1 → Step 5 を順に実施（Lab 3 以降は Step 1〜2 を前提とする）
3. 不要になった Azure リソースは Lab 4.2 末尾の手順でまとめて削除する
