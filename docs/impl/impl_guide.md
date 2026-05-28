# impl_guide.md — Claude Code向け実装ガイド

> このファイルにはコーディング規則・実装パターン・禁止事項を書く。
> アーキテクチャはarchitecture.mdへ。DB詳細はdb.mdへ。
>
> 対象：Claude Code（最優先）・開発者
> Claude Codeは実装前に必ず読む
> 変更頻度：中

---

## Phase 0 実装タスク一覧

現在の目標（完了条件は `impl/spec.md`、仮説は `ops/hypotheses.md` 参照）：

**実行順序：PoC① → PoC③ → PoC② → PoC④**

```
[ ] src/core/bundle.py
[ ] src/core/anchor.py          ← アンカーembeddingの生成・管理
[ ] src/core/coordinate.py      ← アンカー座標変換

[ ] notebooks/poc_01_anchor_stability.ipynb   ← 最優先（H-101〜H-104）
[ ] notebooks/poc_03_edit_locality.ipynb      ← 次（H-301〜H-302）
[ ] notebooks/poc_02_longitudinal.ipynb       ← その次（H-201〜H-203）
[ ] notebooks/poc_04_efficiency.ipynb         ← 最後（H-401〜H-402）

[ ] tests/test_bundle.py
[ ] tests/test_anchor.py
```

**各PoC実装前に `impl/poc_guide.md` を参照すること。**
**各仮説実装前に `ops/hypotheses.md` のステータスを確認すること。**

---

## コアクラスの実装仕様

### SemanticBundle

```python
# src/core/bundle.py

import uuid
import numpy as np
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Optional

@dataclass
class SemanticBundle:
    """
    意味の構造単位。
    仕様: impl/spec.md SB-001 参照。
    数理定義: core/math.md 1.2 参照。
    """
    # 必須フィールド
    core: np.ndarray                    # shape: (d,)
    source: str

    # 自動生成フィールド
    bundle_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    timestamp: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )
    version: int = 1

    # オプションフィールド
    attributes: list = field(default_factory=list)          # list[np.ndarray]
    anchor_coordinates: dict = field(default_factory=dict)  # str -> float
    context_centroid: Optional[np.ndarray] = None
    confidence: float = 1.0
    history: list = field(default_factory=list)             # list[dict]

    def __post_init__(self):
        assert 0.0 <= self.confidence <= 1.0, "confidence must be 0.0 to 1.0"
        assert self.version >= 1, "version must be >= 1"
```

### AnchorSet

```python
# src/core/anchor.py

import numpy as np
from dataclasses import dataclass

@dataclass
class AnchorSet:
    """
    安定アンカー集合。
    仕様: impl/spec.md SB-002 参照。
    数理定義: core/math.md 2.1 参照。
    """
    anchors: dict  # anchor_name -> np.ndarray (shape: d,)

    @classmethod
    def from_config(cls, config: dict, embedder) -> "AnchorSet":
        """設定ファイルからアンカーを生成する（ハードコード禁止）"""
        ...

    def to_coordinates(self, embedding: np.ndarray) -> dict[str, float]:
        """
        embedding -> anchor_coordinates 変換。
        コサイン類似度のみ使用。
        数理定義: core/math.md 2.2 参照。
        """
        ...
```

---

## 実装パターン集

### パターン1：bundle生成の標準フロー

```python
def create_bundle(text: str, anchor_set: AnchorSet, embedder) -> SemanticBundle:
    # Step 1: embedding生成
    embedding = embedder.encode(text)

    # Step 2: アンカー座標変換
    anchor_coords = anchor_set.to_coordinates(embedding)

    # Step 3: bundle生成
    bundle = SemanticBundle(
        core=embedding,
        source=text,
        anchor_coordinates=anchor_coords,
    )

    return bundle
```

### パターン2：類似bundle検索

```python
def search_similar_bundles(
    query_embedding: np.ndarray,
    bundle_store: list[SemanticBundle],
    top_k: int = 5,
) -> list[tuple[SemanticBundle, float]]:
    """
    コサイン類似度でbundleを検索する。
    anchor座標での検索は search_by_anchor() を使う（別関数）。
    """
    ...
```

### パターン3：整合性チェック

```python
def check_consistency(
    output: SemanticBundle,
    reference: SemanticBundle,
    threshold: float,
) -> dict:
    """
    数理定義: core/math.md 7 参照。
    thresholdの値はhypotheses.md #H-007を確認してから使う。
    """
    score = cosine_similarity(output.core, reference.core)
    return {
        "score": score,
        "passed": score >= threshold,
        "flag": "HALLUCINATION_RISK" if score < threshold * 0.5 else None,
    }
```

---

## 禁止事項（コードに書いてはいけないこと）

```python
# NG: アンカー名のハードコード
anchors = ["time", "space", "number"]  # 設定ファイルから読む

# NG: bundle IDの手動割り当て
bundle_id = "bundle_001"              # uuid.uuid4()を使う

# NG: テキスト全文の保存
memory.append({"text": full_text})    # bundle構造で保存する

# NG: 仮説を実装する（hypotheses.mdを確認してから）
θ_merge = 0.9  # 閾値が HYPOTHESIS ステータスなら使わない

# NG: 非公開仕様の実装
def time_dynamics_update(...):        # 非公開仕様。実装しない
    ...

# NG: dot productで類似度計算（コサイン類似度を使う）
similarity = np.dot(a, b)             # cosine_similarity(a, b) を使う
```

---

## ファイル生成規則

```
src/core/bundle.py        ← SemanticBundle クラス
src/core/anchor.py        ← AnchorSet クラス
src/core/coordinate.py    ← 座標変換ユーティリティ
src/storage/             ← Phase 1 から
src/api/                 ← Phase 1 から
src/loops/               ← Phase 1〜2 から
notebooks/poc_*.ipynb    ← Phase 0 の実験ノート
tests/test_*.py          ← テスト（各モジュールに対応）
config/db.py             ← DB設定
config/anchors.yaml      ← アンカー定義
```

---

## テスト規則

```python
# tests/test_bundle.py の例

def test_bundle_id_is_uuid():
    bundle = SemanticBundle(core=np.zeros(768), source="test")
    assert len(bundle.bundle_id) == 36  # UUID v4 format

def test_confidence_range():
    with pytest.raises(AssertionError):
        SemanticBundle(core=np.zeros(768), source="test", confidence=1.5)

def test_anchor_coordinates_cosine():
    """anchor座標はコサイン類似度であること"""
    ...
```

---

## PoC実験ノートの規則

各notebookの先頭に以下を書く：

```markdown
## 実験名：[名前]
## 目的：[何を確認するか]
## 対応仕様：[impl/spec.md の完了条件番号]
## 完了条件：[この実験が「成功」と言える数値基準]
## 結果：[実験後に記入]
## 結論：[仮説を確定・棄却・継続のどれか]
```
