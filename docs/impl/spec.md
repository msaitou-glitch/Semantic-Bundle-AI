# spec.md — 確定仕様

> このファイルには「実装として確定した事項」のみを書く。
> 仮説はhypotheses.mdへ。数理定義はmath.mdへ。
> ここに書いてあることはコードで実装してよい。
>
> 対象：Claude Code・開発者
> 変更頻度：中（仮説が確定するたびに追記）
> **このファイルが唯一の実装真実源**

---

## CONFIRMED仕様一覧

### SB-001：SemanticBundleの基本構造

```python
@dataclass
class SemanticBundle:
    bundle_id: str          # UUID v4
    core: np.ndarray        # shape: (d,)
    attributes: list[np.ndarray]  # each shape: (d,)
    anchor_coordinates: dict[str, float]  # anchor_name -> similarity score
    context_centroid: np.ndarray  # shape: (d,)
    source: str
    timestamp: str          # ISO 8601 UTC
    version: int            # 1以上
    confidence: float       # 0.0 to 1.0
    history: list[dict]     # update logs
```

**禁止：** このデータ構造に新フィールドを追加する場合は `meta/decisions.md` に記録してから行う。

---

### SB-002：アンカー座標変換

入力embedding `e ∈ R^d` をアンカー座標 `z ∈ R^m` に変換する。

```python
def to_anchor_coordinates(
    embedding: np.ndarray,         # shape: (d,)
    anchor_set: list[np.ndarray],  # each shape: (d,)
) -> dict[str, float]:
    """
    Returns anchor_name -> cosine_similarity mapping.
    Cosine similarity is used (not dot product, not euclidean).
    """
```

使用する類似度：**コサイン類似度のみ**（変更は要議論）

---

### SB-003：bundle IDの生成

```python
import uuid
bundle_id = str(uuid.uuid4())
```

手動割り当て禁止。

---

### SB-004：timestampの形式

```python
from datetime import datetime, timezone
timestamp = datetime.now(timezone.utc).isoformat()
# 例: "2025-01-15T09:30:00.123456+00:00"
```

---

### SB-005：LLMとの関係（設計原則）

意味束AIはLLMを置き換えない。LLMは以下を担当する：
- 自然言語生成
- 初期embedding生成
- 推論補助

意味束AIが担当する：
- 意味の安定座標化
- bundle構造の管理
- 意味ドリフト検知
- 長期整合性チェック

**実装ルール：** LLM出力を直接保存しない。bundle化してから保存する。

---

### SB-006：RAGとの関係（設計原則）

RAGは情報検索レイヤー。意味束AIは意味管理レイヤー。

```
RAG        : どの文書/チャンクを取り出すか
意味束AI   : 取り出した情報をどのbundleに統合するか
```

意味束AIはRAGの代替ではない。上位レイヤーとして設計する。

---

### SB-007：メモリ保存形式

テキスト全文をメモリとして蓄積しない。
bundle構造として保存する。

```python
# NG
memory = {"text": "会議でAプロジェクトが承認された。予算は1000万円。"}

# OK
memory = SemanticBundle(
    core=embed("プロジェクト承認"),
    attributes=[embed("Aプロジェクト"), embed("予算"), embed("会議")],
    ...
)
```

---

### SB-008：非公開仕様の取り扱い

以下は公開実装に含めない（`private-specs/` で管理）：
- 時間ダイナミクスの詳細アルゴリズム（忘却関数・Lyapunov項）
- Bundle merge/split の正式仕様
- 束演算の詳細パラメータ設計
- 適応軸切替アルゴリズム

**参照先：** `private-specs/internal_spec_v0.2.md`（NDA契約下でのみ共有）

公開実装では「効果」のみを実装し、「演算体系の詳細」は非公開を維持する。

---

### SB-009：束演算の公開インターフェース定義

6つの束演算を公開APIとして定義する。**内部アルゴリズムは非公開。**

```python
class BundleOperations:

    def merge(self, bundle_a: SemanticBundle,
              bundle_b: SemanticBundle) -> SemanticBundle:
        """
        2つのbundleを統合する。
        要素結合 + 正規化。
        詳細アルゴリズム: private-specs/internal_spec_v0.2.md §4
        """

    def subtract(self, bundle: SemanticBundle,
                 component: np.ndarray) -> SemanticBundle:
        """
        bundle から不要成分を除去する。
        詳細アルゴリズム: private-specs/internal_spec_v0.2.md §4
        """

    def project(self, bundle: SemanticBundle,
                axis: np.ndarray) -> np.ndarray:
        """
        指定軸への成分を抽出する。
        詳細アルゴリズム: private-specs/internal_spec_v0.2.md §4
        """

    def rotate(self, bundle: SemanticBundle,
               context: np.ndarray) -> SemanticBundle:
        """
        文脈座標系へ変換する。
        回転行列 + スケーリング行列で表現。
        詳細アルゴリズム: private-specs/internal_spec_v0.2.md §4
        """

    def compress(self, bundle: SemanticBundle,
                 k: int = 64) -> SemanticBundle:
        """
        要素数を削減しつつ重心を保持する。
        PoCで実証済み：K=64でコサイン類似度0.963・メモリ削減91.7%
        詳細アルゴリズム: private-specs/internal_spec_v0.2.md §4
        """

    def reconstruct(self, bundle_id: str,
                    context: np.ndarray = None) -> SemanticBundle:
        """
        保存済み参照から意味束を再現する（参照再構成）。
        PoCで実証済み：K=64で類似度0.963
        詳細アルゴリズム: private-specs/internal_spec_v0.2.md §4
        """
```

**実装ルール：**
- メソッドのシグネチャと戻り値の型は公開仕様
- 内部実装は `private-specs/` に従う
- Phase 1でFastAPIエンドポイントとして公開する

---

### SB-010：圧縮次元の確定値

PoCで実証済みのパラメータ：

```python
BUNDLE_COMPRESSION = {
    "standard":   64,   # K=64: 類似度0.963・メモリ削減91.7%（推奨）
    "high_quality": 100, # K=100: 類似度0.998・メモリ削減87.0%
    "lightweight":  32,  # K=32: 類似度0.855・メモリ削減95.8%
}
# 256次元（高精度版）は仮説段階 → hypotheses.md H-501
```

---

### SB-011：bundle更新率の確定値

PoCで実証済みのパラメータ：

```python
BUNDLE_UPDATE_RATE = {
    "recommended": 0.1,   # ρ=0.1: 局所化比率0.326・整合性0.931
    "max_safe":    0.15,  # ρ<0.15: H-302 PASS条件
    # ρ≥0.2は意味汚染リスクあり
}
```

---

### SB-012：API評価指標の定義

3つの評価指標をPoCで実証済みとして確定する：

```python
class BundleMetrics:
    score_reconstruction: float  # 参照再現率（0.0〜1.0）
    # PoCで0.963を達成（H-401）

    score_stability: float       # アンカー安定度（0.0〜1.0）
    # PoCで0.931を達成（H-203）

    score_coherence: float       # 意味整合度（0.0〜1.0）
    # PoCで0.967を達成（H-301/302）
```

---

## Phase別確定仕様

### Phase 0（完了）：概念PoC ✅

| 完了条件 | 結果 |
|---------|------|
| アンカー座標分散 < embedding分散 | ✅ 比率0.052（目標0.5） |
| モデル間座標相関 > 0.8 | ❌ トレードオフとして記録 |
| 参照再構成コサイン類似度 > 0.9 | ✅ K=64で0.963 |
| 編集局所化・汚染抑制 | ✅ ρ<0.15で達成 |
| drift削減率 > 30% | ✅ 38.6%削減 |
| 整合性スコア > 0.7 | ✅ 0.931 |

### Phase 1（次フェーズ）：最小Bundle Store

確定予定：
- DB schema（→ `impl/db.md` で管理）
- FastAPI エンドポイント基本セット（→ `impl/api.md`）
- bundle CRUD操作 + 6演算のAPI化
- BundleMetrics の実装

---

## 変更ログ

| 日付 | 変更内容 | 理由 |
|------|---------|------|
| 2026-xx-xx | SB-001〜SB-008 初版作成 | プロジェクト開始 |
| 2026-05-19 | SB-009〜SB-012 追加 | PoC完了・非公開仕様書v0.2との統合 |
