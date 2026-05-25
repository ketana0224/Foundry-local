# Lab 5.1: Execution Provider 切替と性能計測

| 項目 | 内容 |
| --- | --- |
| 想定時間 | 45 分 |
| 前提 | [Lab 2.1](../step2/lab2-1-python.md) 完了 |
| Azure | 使用しない |

Foundry Local は同一モデルに対して、ハードウェアに合わせた複数の **Execution Provider (EP)** バリアント（CPU / DirectML(GPU) / CUDA / QNN(NPU) / OpenVINO 等）を提供します。  
本 lab では、同じプロンプトを異なる EP の variant で実行し、**レイテンシ / トークン/秒 / 出力品質** を比較します。

## 5.1.1 利用可能 variant の確認

```powershell
foundry model info phi-4-mini
```

出力中の `Variants:` セクションに、`...-cpu-int4-...`, `...-gpu-fp16-...`, `...-cuda-int4-...`, `...-qnn-...` などが並びます。  
所有ハードウェアに対応するものをいくつか選んでメモします（例）:

| ラベル | variant ID 例 |
| --- | --- |
| CPU | `phi-4-mini-instruct-cpu-int4-rtn-block-32-acc-level-4` |
| GPU (DirectML) | `phi-4-mini-instruct-gpu-int4-rtn-block-32-acc-level-4` |
| CUDA | `phi-4-mini-instruct-cuda-int4-rtn-block-32` |
| NPU (QNN/Vitis) | `phi-4-mini-instruct-qnn-int4-...` |

> 自分のマシンに無い variant は選んでも DL 後に CPU フォールバックされるか、ロード時に失敗します。所有 HW に応じて選択。

## 5.1.2 variant のダウンロードとロード

```powershell
foundry model download phi-4-mini-instruct-cpu-int4-rtn-block-32-acc-level-4
foundry model download phi-4-mini-instruct-gpu-int4-rtn-block-32-acc-level-4
```

## 5.1.3 ベンチマークスクリプト

`bench.py`:

```python
import time
import statistics
from foundry_local import FoundryLocalManager
from openai import OpenAI

# 計測したい variant のフル ID を列挙
VARIANTS = [
    "phi-4-mini-instruct-cpu-int4-rtn-block-32-acc-level-4",
    "phi-4-mini-instruct-gpu-int4-rtn-block-32-acc-level-4",
    # "phi-4-mini-instruct-cuda-int4-rtn-block-32",
    # "phi-4-mini-instruct-qnn-int4-...",
]

PROMPT = "Foundry Local が提供するメリットを 5 個、箇条書きで簡潔に挙げてください。"
WARMUP = 1
RUNS   = 3
MAX_TOKENS = 256


def bench(model_id: str):
    mgr = FoundryLocalManager(model_id)  # フル ID 指定
    info = mgr.get_model_info(model_id)
    client = OpenAI(base_url=mgr.endpoint, api_key=mgr.api_key or "x")

    # warm-up
    for _ in range(WARMUP):
        client.chat.completions.create(
            model=info.id,
            messages=[{"role": "user", "content": "Hi"}],
            max_tokens=4, temperature=0,
        )

    lat = []
    tps = []
    for _ in range(RUNS):
        t0 = time.perf_counter()
        r = client.chat.completions.create(
            model=info.id,
            messages=[{"role": "user", "content": PROMPT}],
            max_tokens=MAX_TOKENS, temperature=0,
        )
        dt = time.perf_counter() - t0
        out_tok = r.usage.completion_tokens
        lat.append(dt)
        tps.append(out_tok / dt if dt > 0 else 0)

    return {
        "variant": model_id,
        "latency_med_s": statistics.median(lat),
        "tokens_per_sec_med": statistics.median(tps),
        "sample_output": r.choices[0].message.content[:120].replace("\n", " "),
    }


print(f"{'variant':70} {'latency(s)':>12} {'tok/s':>8}")
print("-" * 96)
for v in VARIANTS:
    try:
        m = bench(v)
        print(f"{m['variant']:70} {m['latency_med_s']:12.2f} {m['tokens_per_sec_med']:8.1f}")
    except Exception as e:
        print(f"{v:70} ERROR: {e}")
```

実行:

```powershell
python bench.py
```

> 一度に複数 variant をロードするとメモリを圧迫します。スクリプト終端で `foundry model stop ...` を順番に呼ぶか、VRAM 監視（タスクマネージャの「専用 GPU メモリ」）を確認しながら実施してください。

## 5.1.4 観察ポイント

| 比較軸 | 期待される傾向 |
| --- | --- |
| latency_med_s | GPU/NPU 系 < CPU 系（多くの場合） |
| tokens_per_sec | GPU/NPU 系 ≫ CPU 系 |
| 出力品質 | 同一モデルでも量子化（int4 / fp16）で微差 |
| メモリ使用 | GPU variant は VRAM、CPU variant はメイン メモリを消費 |

GPU が振るわない場合の典型原因:

- ノート PC で iGPU を使用しており、VRAM が共有メモリで遅い
- ドライバが古い（特に Intel/AMD/NVIDIA）。最新ドライバへ更新
- 他プロセス（ブラウザの動画再生、AI Toolkit for VS Code）が GPU を占有

## 5.1.5 簡易レポート（任意）

`bench.py` の結果を表に貼り付け、観察を 2〜3 行コメントしてレポート化する:

```markdown
| variant | latency (s) | tok/s | コメント |
| --- | --- | --- | --- |
| cpu-int4 | 8.2 | 22 | ベースライン。出力は十分実用。 |
| gpu-int4 | 3.1 | 60 | iGPU でも 3 倍弱。初回ロード 6s。 |
```

## チェックリスト

- [ ] 2 つ以上の variant を計測できた
- [ ] latency と tok/s の差分を表に整理できた
- [ ] 結果から、自分のマシンで推奨する EP を 1 つ決められた

次へ → [Lab 5.2: モデル追加 / 容量管理](lab5-2-models.md)
