# math.md — 数理モデル・定式化

> このファイルには数式・定義・定理を書く。
> 哲学的解釈はvision.mdへ。実装詳細はspec.mdへ。
> 「なぜこの数式か」の背景はthesis.mdへ。
>
> 対象：数理系研究者・実装者（数式を理解して実装する人）
> Claude Codeは実装時に参照する
> 変更頻度：低（数理基盤は安定しているべき）

---

## 記法

| 記号 | 意味 |
|------|------|
| `d` | embeddingの次元数 |
| `m` | アンカー数 |
| `n` | 入力数（bundle生成時） |
| `K` | 疎合成時に使用する属性数 |
| `N` | 管理する概念の総数 |
| `e` | embedding vector |
| `a_i` | i番目のアンカーベクトル |
| `v_i` | i番目の属性ベクトル |
| `α_i` | 再構成時の係数 |
| `w_i` | 重みスカラー |
| `sim(·,·)` | コサイン類似度（デフォルト） |

---

## 1. 基本表現

### 1.1 通常のembedding

```
e ∈ R^d
```

### 1.2 意味束（Semantic Bundle）の定義

基本形：
```
S_B = (C, {v_i})
```

拡張形（標準定義）：
```
S_B = (C, {v_i}, A, M, H)

  C  : コア識別子またはコアベクトル
  {v_i} : 属性ベクトル群  v_i ∈ R^d
  A  : アンカー座標  A ∈ R^m
  M  : 文脈重心  M ∈ R^d
  H  : 更新履歴
```

---

## 2. 安定アンカー座標

### 2.1 アンカー集合

```
AnchorSet = {a_1, a_2, ..., a_m}  where a_i ∈ R^d
```

### 2.2 アンカー座標変換

入力embedding `e` をアンカー座標 `z` に変換する：

```
z = [sim(e, a_1), sim(e, a_2), ..., sim(e, a_m)]
z ∈ R^m
```

`sim` はデフォルトでコサイン類似度：
```
sim(u, v) = (u · v) / (||u|| · ||v||)
```

### 2.3 次元削減の効果

`m << d` のとき、アンカー座標は高次元embeddingの低次元安定表現となる。
解釈可能性と安定性を兼ねる。

---

## 3. 文脈重心

### 3.1 単純形

```
C = (1/n) * Σ e_i   for i = 1..n
```

### 3.2 加重形（正式定義）

```
C = Σ(w_i * e_i) / Σ(w_i)
```

重み `w_i` の構成要素：
- `w_time(i)` : 時間減衰重み（新しいほど重い）
- `w_conf(i)` : 信頼度スコア
- `w_ctx(i)`  : 文脈重要度
- `w_src(i)`  : 出典信頼性
- `w_freq(i)` : 使用頻度

結合方法（仮説段階）：
```
w_i = w_time(i) * w_conf(i) * w_ctx(i) * w_src(i) * w_freq(i)
```
→ 詳細は `ops/hypotheses.md` #H-004 参照

---

## 4. 参照再構成

### 4.1 基本形

```
x_hat = Σ(α_i * v_i)   for i = 1..K
```

`K` は疎合成で使用する属性数（`K << d`）

### 4.2 拡張形（構造補正あり）

```
x_hat(t) = Σ(α_i(t) * v_i) + correction(C, A, context)
```

**注意：** 非線形弾性項・関係テンソルは非公開仕様。実装しない。

### 4.3 係数の決定

基本：最小二乗法または内積による投影
```
α = argmin ||x - Σ(α_i * v_i)||^2
```

---

## 5. 計算量

### 5.1 疎合成

K個の属性を使用する場合：
```
O(K * d)
```

### 5.2 メモリ効率

N概念をd次元全保持する場合との比較：
```
節約率 = 1 - (K / d)
```
K=32, d=768 のとき: 節約率 ≈ 95.8%（仮説段階）
→ `ops/hypotheses.md` #H-002 参照

---

## 6. 意味ドリフト測定

### 6.1 Embedding Drift

```
drift_embed(concept, t1, t2) = ||e(concept, t1) - e(concept, t2)||_2
```

### 6.2 Anchor Coordinate Drift

```
drift_anchor(concept, t1, t2) = ||z(concept, t1) - z(concept, t2)||_2
```

仮説：`drift_anchor < drift_embed`（アンカー座標の方が安定）

### 6.3 Anchor Self-Drift

アンカー自体のドリフト：
```
drift(a_i, t1, t2) = ||a_i(t1) - a_i(t2)||_2
```
閾値を超えたらアンカー更新を検討 → `ops/loops.md` 参照

---

## 7. 意味整合性スコア

```
SC(output, reference) = sim(bundle(output), bundle(reference))
range: [0.0, 1.0]
```

閾値 `τ` は `impl/spec.md` で管理（現在は仮説段階）

---

## 8. Bundle更新則

### 8.1 概念式

```
S_B(t+1) = Update(S_B(t), input_t, anchor_state_t)
```

### 8.2 簡易指数移動平均形

```
C_{t+1} = (1 - ρ) * C_t + ρ * NewInput_t
```

`ρ` : 更新率（0.0〜1.0）

**注意：** 正式な時間ダイナミクスは非公開仕様。この式はPoC用の簡易形。

---

## 9. 文脈変形

```
ContextualState = C + Δ_context
```

長期保存時の安定化：
```
C_new = Stabilize(C, Δ_context, AnchorSet)
```

`Stabilize` の実装はPhase 1以降で定義する。

---

## 10. Bundle統合・分割の判定

### 10.1 統合条件

```
if dist(bundle_a, bundle_b) < θ_merge:
    merge(bundle_a, bundle_b)
```

### 10.2 分割条件

bundle内の意味重心が複数に分岐した場合：
```
if has_multiple_centroids(bundle, θ_split):
    split(bundle)
```

`θ_merge`, `θ_split` の具体値は仮説段階 → `ops/hypotheses.md` #H-005, #H-006
