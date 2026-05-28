# glossary.md — 用語定義

> このファイルは全ドキュメント・全コードの用語共通参照元である。
> 新しい用語を使う前にここに定義を追加する。
> 定義の変更は必ずここで行い、他ファイルはここを参照する。
>
> 対象：全員（Claude Code・開発者・研究者・投資家）
> 変更頻度：低（追加はあるが既存定義を変更する場合は要議論）

---

## コア概念

### Semantic Bundle（意味束）

安定座標系上に配置された意味の構造単位。

単一ベクトルやRAGチャンクとは異なり、以下の要素から成る複合構造体：

```
SemanticBundle = {
  bundle_id:           UUID,          // 一意識別子
  core:                vector | str,  // コア概念（識別子または中心ベクトル）
  attributes:          [vector],      // 属性ベクトル群
  anchor_coordinates:  {str: float},  // アンカーへの類似度マップ
  context_centroid:    vector,        // 文脈重心
  source:              str,           // 出典
  timestamp:           ISO8601,       // 生成・更新日時
  version:             int,           // 更新バージョン
  confidence:          float,         // 0.0〜1.0
  history:             [update_log]   // 更新履歴
}
```

**使ってはいけない言い方：** 「embeddings」「チャンク」「ドキュメント」と混同しない。

---

### Stable Anchor（安定アンカー）

意味空間内で**比較的安定した認知基準**として機能する参照点の集合。

**重要な立場：**
アンカーは「絶対に不変な真理」ではなく、「他の概念より相対的に変化しにくい認知基準」である。文化差・時代変化・モデル更新があっても、完全には崩壊しにくい概念群を採用する。

**選定の基本方針：**
人間の身体性・基礎認知に強く結びついた概念が候補となる。統計的頻度ではなく、認知的安定性を基準とする。

```
選定候補の類型：
  空間認知系：近い／遠い、上／下、左／右
  時間認知系：過去／現在／未来、早い／遅い
  数量認知系：多い／少ない、増加／減少
  身体感覚系：熱い／冷たい、痛い、明るい／暗い
  危険回避系：安全／危険、脅威
  基礎行動系：食べる、危険、安全
```

**アンカーではないもの：**
ドメイン固有の専門概念（confidentiality, diagnosisなど）はアンカーではなく「意味基底（Semantic Basis）」として分類する。→ 下記参照

```
AnchorSet = [a_1, a_2, ..., a_m]
```

**注意：** アンカー自体もドリフトしうる。→ `ops/loops.md` 参照

---

### Semantic Basis（意味基底）

特定ドメインの意味を分解・投影するための軸ベクトル群。アンカーとは異なる概念。

| 概念 | 役割 | 数式 | 例 |
|------|------|------|-----|
| アンカー | 安定参照（どこに位置するか） | `sim(e, anchor_i)` | 近い・危険・時間 |
| 意味基底 | 意味分解（どの成分を持つか） | `proj(e, basis_j)` | 感情軸・論理軸・文化軸 |
| PCA基底 | 圧縮最適化（工学的効率化） | PCA変換 | K=64主成分 |

**重要：** この3つを混同しない。特に「ドメイン特化アンカー」と呼んでいたものは、実際には意味基底に近い。

---

### Anchor Coordinate（アンカー座標）

入力embeddingをアンカー集合との類似度ベクトルに変換した表現。

```
z = [sim(e, a_1), sim(e, a_2), ..., sim(e, a_m)]
z ∈ R^m  (m = アンカー数、通常 m << d)
```

従来のembedding（他語との相対位置）とは異なり、アンカーとの絶対的距離関係を表す。

---

### Context Centroid（文脈重心）

複数の文脈入力から形成されるbundle内の中心ベクトル。

単純平均ではなく、重要度・時間・信頼度・使用頻度を反映した加重平均：

```
C = Σ(w_i * e_i) / Σ(w_i)
```

---

### Semantic Drift（意味ドリフト）

時間・再学習・微調整・文脈変化によって、同一概念のベクトル位置や意味関係が変動すること。

測定：
```
drift(concept, t1, t2) = distance(embedding(concept, t1), embedding(concept, t2))
```

種類：
- **Embedding Drift**：モデル更新による座標変化
- **Context Drift**：長期対話での文脈ズレ
- **Anchor Drift**：アンカー自体のズレ

---

### Semantic Consistency（意味整合性）

出力bundleまたはLLM出力が、既存の参照bundleとどの程度一致しているかのスコア。

```
consistency_score = sim(output_bundle, reference_bundle)
range: 0.0〜1.0
threshold: (仮説中、impl/spec.mdで確定値を管理)
```

---

### Referential Reconstruction（参照再構成）

意味を全文保存するのではなく、安定座標・コア・属性ベクトルから必要時に再構成する方式。

```
x_hat = Σ(α_i * v_i)
```

目的：メモリ削減・推論コスト削減・再利用性向上

---

### Bundle Network（意味ネットワーク）

意味束同士が関係を持つグラフ構造。

```
Node:   SemanticBundle
Edge:   {type: similar|causal|hierarchical|reference, weight: float}
```

---

## 区別すべき類似概念

| 用語 | 本プロジェクトでの扱い |
|------|----------------------|
| Embedding | LLMが生成する相対的ベクトル。そのままでは使わない。アンカー座標へ変換する |
| RAG chunk | 情報検索単位。意味束の下位概念。意味管理はしない |
| Memory | テキストログではなくbundle群として実装する |
| Context window | LLM側の一時的な保持。意味束AIとは別レイヤー |
| Fine-tuning | モデル内部の変更。意味束AIはモデル外部レイヤー |

---

## Layer用語

| Layer | 名称 | 説明 |
|-------|------|------|
| 0 | Raw Input | テキスト・データ入力 |
| 1 | Embedding | LLMによるベクトル化 |
| 2 | Anchor Coordinate | アンカーへの座標変換 |
| 3 | Semantic Bundle | bundle構造化・保存 |
| 4 | Bundle Network | bundle間のグラフ構造 |
| 5 | Application | Agent・API・アプリ出力 |

---

## ステータス用語（hypotheses.md で使用）

| ステータス | 意味 |
|-----------|------|
| `CONFIRMED` | 確定仕様。spec.mdに移動済み |
| `HYPOTHESIS` | 仮説。検証前 |
| `TESTING` | 現在検証中 |
| `REJECTED` | 棄却された仮説 |
| `DEFERRED` | 将来検討。現在はスコープ外 |

---

## 略語

| 略語 | 正式名称 |
|------|---------|
| SB | Semantic Bundle（意味束） |
| SA | Stable Anchor（安定アンカー） |
| AC | Anchor Coordinate（アンカー座標） |
| CC | Context Centroid（文脈重心） |
| SC | Semantic Consistency（意味整合性） |
| RR | Referential Reconstruction（参照再構成） |
