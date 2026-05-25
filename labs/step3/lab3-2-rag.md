# Lab 3.2: ローカル RAG（Embedding + ベクトル検索）

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 50 分 |
| 前提 | [Lab 3.1](lab3-1-sk-langchain.md) 完了 |
| Azure | 使用しない |

完全ローカル RAG（Retrieval-Augmented Generation）を構築します。Embedding は **`sentence-transformers`（CPU）** を、生成は Foundry Local の Chat モデルを使用、ベクトルストアは **FAISS (in-memory)** で完結します。

> Foundry Local は Chat モデル中心で、Embedding モデルのカタログ提供状況はバージョンによります。本 lab は「LLM はローカル、Embedding はオフライン HuggingFace」の組み合わせで「最小構成のローカル RAG」を体験することが目的です。

## 3.2.1 セットアップ

```powershell
mkdir labs-work\step3.2
cd labs-work\step3.2

python -m venv .venv
.\.venv\Scripts\Activate.ps1

pip install foundry-local-sdk-winml openai
pip install sentence-transformers faiss-cpu numpy
```

Embedding モデル（`intfloat/multilingual-e5-small`、約 470MB）は初回 `SentenceTransformer(...)` 呼び出し時に HuggingFace から自動ダウンロードされます。

## 3.2.2 ナレッジ用サンプル文書

`docs/foundry-local.md`:

```markdown
# Foundry Local の特徴

- Foundry Local はローカル PC 上で LLM 推論を行うランタイムで、ONNX Runtime を基盤としている。
- OpenAI 互換の REST API を提供し、既存の OpenAI SDK から base_url を差し替えるだけで利用できる。
- ハードウェアに応じて CPU / DirectML(GPU) / CUDA / NPU など最適な Execution Provider を自動選択する。
- モデルは Foundry Catalog から初回 DL 後、ローカルキャッシュに保存され、以降オフライン実行できる。
- 推奨 OS は Windows 11 24H2 以降。macOS では Apple Silicon の Metal 経由で動作する。
```

`docs/azure.md`:

```markdown
# Microsoft Foundry (クラウド) の特徴

- Microsoft Foundry はクラウド上で各種モデルをホストするサービスで、Azure AI Foundry の中核を成す。
- gpt 系のフロンティアモデル、Anthropic Claude、Mistral など多様なモデルを統一エンドポイントから利用できる。
- スケーラブルで大規模トラフィックに対応し、コンテンツ安全性フィルタや評価機能を備える。
- 利用は従量課金で、東日本・西日本リージョンに展開可能。
```

## 3.2.3 RAG パイプライン実装

`rag_app.py`:

```python
import glob
import os
from pathlib import Path

import faiss
import numpy as np
from foundry_local import FoundryLocalManager
from openai import OpenAI
from sentence_transformers import SentenceTransformer

ALIAS = "phi-4-mini"

# 1. Foundry Local 起動
manager = FoundryLocalManager(ALIAS)
client = OpenAI(base_url=manager.endpoint, api_key=manager.api_key or "x")
model_id = manager.get_model_info(ALIAS).id
print(f"chat model: {model_id}")

# 2. 文書を読み込み、簡易チャンク化（段落単位）
docs = []
for path in glob.glob("docs/*.md"):
    text = Path(path).read_text(encoding="utf-8")
    for i, chunk in enumerate(c.strip() for c in text.split("\n\n") if c.strip()):
        docs.append({"source": f"{path}#chunk{i}", "text": chunk})
print(f"chunks: {len(docs)}")

# 3. Embedding（CPU）
encoder = SentenceTransformer("intfloat/multilingual-e5-small")

def embed(texts, prefix):
    # e5 系は "query: " / "passage: " プレフィックス推奨
    return encoder.encode(
        [f"{prefix}: {t}" for t in texts],
        normalize_embeddings=True,
        convert_to_numpy=True,
    ).astype("float32")

doc_vecs = embed([d["text"] for d in docs], prefix="passage")

# 4. FAISS (内積 = 正規化済みなので cosine 相当)
index = faiss.IndexFlatIP(doc_vecs.shape[1])
index.add(doc_vecs)

def retrieve(query: str, k: int = 3):
    qv = embed([query], prefix="query")
    scores, ids = index.search(qv, k)
    return [(docs[i], float(scores[0][rank])) for rank, i in enumerate(ids[0])]

# 5. 推論
def answer(question: str) -> str:
    hits = retrieve(question, k=3)
    context = "\n\n".join(f"[{h['source']}] {h['text']}" for h, _ in hits)
    messages = [
        {"role": "system",
         "content": "次のコンテキストの内容にのみ基づいて、日本語で簡潔に答えてください。"
                    "コンテキストに無い情報は『資料には記載がありません』と答えてください。"},
        {"role": "user", "content": f"# 質問\n{question}\n\n# コンテキスト\n{context}"},
    ]
    resp = client.chat.completions.create(
        model=model_id, messages=messages, temperature=0.0, max_tokens=400,
    )
    return resp.choices[0].message.content


for q in [
    "Foundry Local のモデルはどこに保存されますか？",
    "Microsoft Foundry はどんなモデルを使えますか？",
    "Foundry Local の料金体系は？",  # 資料に無い → 不明と答えてほしい
]:
    print("\nQ:", q)
    print("A:", answer(q))
```

実行:

```powershell
python rag_app.py
```

期待する挙動:

- 1 問目は `docs/foundry-local.md` から「ローカルキャッシュ」と回答される
- 2 問目は `docs/azure.md` から「gpt / Claude / Mistral」等と回答される
- 3 問目は「資料には記載がありません」と返る

## 3.2.4 発展課題（時間が余ったとき）

- `k`（取得数）や `chunk` 粒度を変えて回答品質の変化を観察する
- 文書数を増やし、FAISS を `IndexFlatIP` から `IndexHNSWFlat` に切り替える
- 検索結果の `score` と `source` を回答末尾に列挙し、引用付き回答にする

## チェックリスト

- [ ] 質問 1・2 にコンテキストから引用した回答が返った
- [ ] 質問 3 に「資料には記載がありません」と返った
- [ ] 全工程がローカル（外部 LLM API 呼び出しなし）で完結した

次へ → [Lab 4.1: Microsoft Foundry (クラウド) との比較](../step4/lab4-1-foundry-cloud.md)
