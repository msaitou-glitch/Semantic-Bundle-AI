# poc_guide.md — PoC実装ガイド

> このファイルにはPoC①〜④の具体的な実験設計・実装手順・評価方法を書く。
> 仮説のステータス管理はhypotheses.mdへ。実装ルールはimpl_guide.mdへ。
>
> 対象：Claude Code・開発者
> Claude CodeはポC実装時に必ず読む
> 変更頻度：中（実験結果を受けて更新）

---

## PoCの基本方針

**「理論を完全証明する」ではなく「比較優位性を数字で示す」**

各PoCに共通するルール：
- 必ずbaseline（通常embedding）との比較を出す
- 数字が出ること > コードが綺麗なこと
- Google Colabで動くこと（再現可能性）
- 可視化は必ず入れる（図がそのまま論文・HPに使える）

---

## 実行順序

```
PoC① → PoC③ → PoC② → PoC④
```

理由：
- PoC①が全ての基盤。最優先。
- PoC③（編集局所化）はPoC①の構造があれば比較的速く実装できる
- PoC②（長期整合性）は時系列データが必要でやや複雑
- PoC④（効率性）は①の結果を使い回せる

---

## 共通セットアップ

### 依存パッケージ

```python
# requirements_poc.txt
sentence-transformers>=2.2.0
numpy>=1.24.0
scikit-learn>=1.2.0
matplotlib>=3.7.0
seaborn>=0.12.0        # 可視化
pandas>=1.5.0          # 結果管理
```

### アンカーのembedding生成方法

アンカーは1語ではなく複数の代表文の平均で作る（1語は不安定）：

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-mpnet-base-v2")

ANCHOR_DEFINITIONS = {
    "time": [
        "time passes", "temporal sequence", "before and after",
        "duration", "past present future", "chronological order"
    ],
    "space": [
        "physical location", "spatial distance", "here and there",
        "place and position", "geographic area", "proximity"
    ],
    "risk": [
        "potential harm", "danger and safety", "probability of loss",
        "threat assessment", "hazard and mitigation", "security risk"
    ],
    "rule": [
        "regulations and policies", "rules must be followed",
        "legal requirements", "compliance standards", "prohibited actions"
    ],
    "trust": [
        "reliability and credibility", "authority and legitimacy",
        "verified information", "trustworthy source"
    ],
    "number": [
        "quantity and amount", "numerical value", "counting and measuring",
        "percentage and ratio", "scale and magnitude"
    ],
    "body": [
        "physical sensation", "bodily experience", "touch and feeling",
        "human physiology", "sensory perception"
    ],
    "emotion": [
        "emotional state", "feelings and mood", "psychological experience",
        "affective response", "sentiment and attitude"
    ],
}

def build_anchor_embeddings(model, definitions):
    anchors = {}
    for name, sentences in definitions.items():
        embeddings = model.encode(sentences)
        anchors[name] = embeddings.mean(axis=0)  # 平均で安定化
    return anchors
```

### アンカー座標変換

```python
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

def to_anchor_coordinates(embedding, anchor_embeddings):
    """
    embedding: shape (d,)
    anchor_embeddings: dict[str, ndarray(d,)]
    returns: dict[str, float]  # anchor_name -> cosine_similarity
    """
    coords = {}
    emb = embedding.reshape(1, -1)
    for name, anchor in anchor_embeddings.items():
        anc = anchor.reshape(1, -1)
        coords[name] = float(cosine_similarity(emb, anc)[0][0])
    return coords

def anchor_coords_to_vector(coords, anchor_names):
    """dict -> numpy vector（順序固定）"""
    return np.array([coords[name] for name in anchor_names])
```

---

## PoC① — 意味安定座標

**対応仮説：** H-101, H-102, H-103, H-104
**実装ファイル：** `notebooks/poc_01_anchor_stability.ipynb`

### データセット設計

```python
# 同義群：情報セキュリティ × 公共空間 のシナリオ（20文）
SYNONYM_GROUP = [
    "Can I read confidential documents at a café?",
    "Is it okay to review secret files in a coffee shop?",
    "機密書類をカフェで読んでも問題ないですか？",
    "公共の場で社外秘資料を確認することは許可されていますか？",
    "Is reviewing internal documents in public spaces against policy?",
    "カフェでの機密情報取り扱いに関する規程はありますか？",
    # ... 計20文（日英混在で文化差もカバー）
]

# ノイズ群：全く無関係な概念（各5文）
NOISE_GROUPS = {
    "pasta":    ["I love spaghetti", "pasta with tomato sauce", ...],
    "mountain": ["Mount Fuji is beautiful", "hiking in the mountains", ...],
    "python":   ["Python is a programming language", "pip install numpy", ...],
    "phone":    ["smartphone battery life", "mobile phone screen", ...],
}

# 文化差群：同一状況の文化別表現（各5文）
CULTURAL_VARIANTS = {
    "japan":   ["公共の場での配慮が必要です", "場の雰囲気を読む", ...],
    "us":      ["privacy liability in public spaces", "personal responsibility", ...],
    "europe":  ["GDPR compliance in public areas", "data protection regulations", ...],
}
```

### 実験手順

```python
# Step 1: 2モデルでembedding生成
models = {
    "mpnet": SentenceTransformer("all-mpnet-base-v2"),
    "minilm": SentenceTransformer("all-MiniLM-L6-v2"),
}

# Step 2: 通常embeddingと アンカー座標の両方を計算
results = {}
for model_name, model in models.items():
    anchors = build_anchor_embeddings(model, ANCHOR_DEFINITIONS)
    
    embeddings = model.encode(SYNONYM_GROUP)
    anchor_coords = [
        anchor_coords_to_vector(to_anchor_coordinates(e, anchors), list(anchors.keys()))
        for e in embeddings
    ]
    results[model_name] = {
        "embeddings": embeddings,
        "anchor_coords": np.array(anchor_coords),
    }

# Step 3: 指標計算
def cluster_variance(vectors):
    centroid = vectors.mean(axis=0)
    return np.mean([np.linalg.norm(v - centroid)**2 for v in vectors])

for model_name, data in results.items():
    var_embed  = cluster_variance(data["embeddings"])
    var_anchor = cluster_variance(data["anchor_coords"])
    print(f"{model_name}: var_embed={var_embed:.4f}, var_anchor={var_anchor:.4f}, ratio={var_anchor/var_embed:.3f}")
```

### H-102：モデル間ランキング一致率の測定

```python
from scipy.stats import spearmanr

def ranking_consistency(coords_A, coords_B, query_idx=0):
    """
    query_idxの文から見た、他の文への距離ランキングの一致率
    """
    from sklearn.metrics.pairwise import cosine_distances
    
    dist_A = cosine_distances(coords_A[query_idx:query_idx+1], coords_A)[0]
    dist_B = cosine_distances(coords_B[query_idx:query_idx+1], coords_B)[0]
    
    rank_A = np.argsort(dist_A)
    rank_B = np.argsort(dist_B)
    
    corr, _ = spearmanr(rank_A, rank_B)
    return corr

# baselineとbundleの両方で計算して比較
```

### 可視化（必須）

```python
import matplotlib.pyplot as plt
from sklearn.decomposition import PCA

fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# 左上：通常embedding（PCA 2D）
# 右上：アンカー座標（2D）
# 左下：モデルA vs B の座標比較
# 右下：分散比較バーチャート

# 色分け：同義群=青、ノイズ群=各色、文化差群=形状違い
```

### ノートブックの冒頭に必ず書く

```markdown
## 実験名：PoC① 意味安定座標
## 対応仮説：H-101, H-102, H-103, H-104
## 問い：アンカー座標化で意味クラスタは安定するか？
## 完了条件：
  - H-101: variance_anchor / variance_embedding < 0.5
  - H-102: モデル間ranking_consistency > baseline
  - H-103: アンカー1個除去で相関 > 0.9
  - H-104: ノイズアンカー1個追加でスコア低下 < 10%
## 結果：（実験後に記入）
## 結論：（仮説を CONFIRMED / REJECTED / CONTINUE のどれかで）
```

---

## PoC② — 長期整合性

**対応仮説：** H-201, H-202, H-203
**実装ファイル：** `notebooks/poc_02_longitudinal.ipynb`

### 更新シミュレーション設計

```python
# 企業セキュリティポリシーの逐次更新シナリオ
INITIAL_DOCS = [
    "Confidential documents must not be taken outside the office.",
    "Company data should only be accessed on secure networks.",
]

UPDATE_SERIES = [
    # t=1: 新ルール追加
    "Remote work is now permitted with VPN usage.",
    # t=2: 例外追加
    "Certain documents may be reviewed in designated secure zones.",
    # t=3: 業界変化
    "Cloud storage solutions are approved for specific data categories.",
    # t=4〜10: 継続更新...
]

# baseline：毎ステップ全文書のembedding平均を再計算
# bundle  ：加重移動平均で重心を更新
#   C_{t+1} = (1 - ρ) * C_t + ρ * new_embedding
#   ρ = 0.3（仮値、H-503の検証材料にもなる）
```

### Semantic Collapse検出

```python
def detect_semantic_collapse(centroid_before, centroid_after, all_cluster_centroids, threshold=0.5):
    """
    更新後に概念が別クラスタに近くなったかを検出
    """
    sim_before = {name: cosine_similarity(centroid_before, c) 
                  for name, c in all_cluster_centroids.items()}
    sim_after  = {name: cosine_similarity(centroid_after, c) 
                  for name, c in all_cluster_centroids.items()}
    
    closest_before = max(sim_before, key=sim_before.get)
    closest_after  = max(sim_after,  key=sim_after.get)
    
    return closest_before != closest_after  # Trueなら崩壊
```

---

## PoC③ — 編集局所化

**対応仮説：** H-301, H-302
**実装ファイル：** `notebooks/poc_03_edit_locality.ipynb`

### Edit Locality Score

```python
def edit_locality_score(
    bundles_before,      # dict[name, centroid_vector]
    bundles_after,       # dict[name, centroid_vector]
    edited_bundle_name,  # "apple_company"
    related_names,       # ["google", "microsoft"]  直接関連
    unrelated_names,     # ["security", "pasta"]    無関係
):
    """
    高いほど「編集が正しい場所にだけ効いている」
    """
    def change(name):
        return cosine_distance(bundles_before[name], bundles_after[name])
    
    edited_change    = change(edited_bundle_name)
    related_change   = mean([change(n) for n in related_names])
    unrelated_change = mean([change(n) for n in unrelated_names])
    
    # 編集bundle変化 >> 関連変化 >> 無関係変化  が理想
    locality = 1 - (unrelated_change / (edited_change + 1e-8))
    return locality  # 1に近いほど局所化されている
```

---

## PoC④ — 効率性

**対応仮説：** H-401, H-402
**実装ファイル：** `notebooks/poc_04_efficiency.ipynb`

### 再構成精度とメモリ効率

```python
def sparse_reconstruction(target_embedding, attribute_vectors, k):
    """
    k個の属性ベクトルで target を再構成
    最小二乗法で係数を決定
    """
    # 上位k個の属性を選択（目標との類似度で）
    sims = cosine_similarity(
        target_embedding.reshape(1, -1),
        np.array(attribute_vectors)
    )[0]
    top_k_idx = np.argsort(sims)[-k:]
    selected  = np.array(attribute_vectors)[top_k_idx]
    
    # 最小二乗法で係数計算
    coeffs, _, _, _ = np.linalg.lstsq(selected.T, target_embedding, rcond=None)
    
    reconstruction = (coeffs.reshape(-1, 1) * selected).sum(axis=0)
    return reconstruction

# K = 8, 16, 32 で再構成誤差とメモリ比を測定
for k in [8, 16, 32]:
    x_hat = sparse_reconstruction(target, attributes, k)
    similarity = cosine_similarity(target.reshape(1,-1), x_hat.reshape(1,-1))[0][0]
    memory_ratio = k / len(target)  # K/d
    print(f"K={k}: similarity={similarity:.3f}, memory_ratio={memory_ratio:.3f}")
```

---

## 結果の記録フォーマット

各PoC完了後に以下を `docs/poc_results/` に保存する（Phase 1以降でディレクトリ作成）：

```markdown
# poc_01_results.md

## 実験日：YYYY-MM-DD
## モデル：all-mpnet-base-v2, all-MiniLM-L6-v2
## アンカー数：8

## H-101 結果
variance_ratio（mpnet）: X.XXX
variance_ratio（minilm）: X.XXX
→ 判定：CONFIRMED / REJECTED

## H-102 結果
ranking_consistency（embedding baseline）: X.XXX
ranking_consistency（anchor coords）: X.XXX
→ 判定：CONFIRMED / REJECTED

## 総合所見：
（何が予想通りで、何が予想外だったか）

## 次のアクション：
（結果を受けてhypotheses.mdをどう更新するか）
```
