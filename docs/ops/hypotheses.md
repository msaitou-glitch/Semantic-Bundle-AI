# hypotheses.md — 仮説一覧・検証状況・進捗

> このファイルには「確定していない仮説」を書く。
> 確定したものはspec.mdへ移動する。棄却したものはここに残す（記録として）。
> 確定事項をここに書かない。仮説をspec.mdに書かない。
>
> 対象：Claude Code・開発者・研究者
> Claude Codeは実装前に必ずここで仮説ステータスを確認する
> 変更頻度：高（検証のたびに更新）

---

## ステータス定義

| ステータス | 意味 |
|-----------|------|
| `CONFIRMED` | 確定。spec.mdに移動済み |
| `HYPOTHESIS` | 仮説。未検証 |
| `TESTING` | 現在検証中（誰が・いつまで） |
| `REJECTED` | 棄却。理由を記録 |
| `DEFERRED` | 将来検討。現在スコープ外 |

**重要：** `HYPOTHESIS` / `TESTING` ステータスの値をコードにハードコードしない。

---

## PoCと仮説の対応マップ

```
PoC①（意味安定座標）  →  H-101, H-102, H-103, H-104
PoC②（長期整合性）    →  H-201, H-202, H-203
PoC③（編集局所化）    →  H-301, H-302
PoC④（効率性）        →  H-401, H-402
実装設計仮説          →  H-501〜
```

各PoC実装前に対応するH番号のステータスを確認する。

---

## PoC① — 意味安定座標（Stable Coordinate Hypothesis）

> **PoCの問い：** アンカー座標化により、モデル差・文脈差があっても意味クラスタの安定性が向上するか

---

### H-101：アンカー座標はクラスタ分散を減らす

**主張：** 同義文群のアンカー座標分散が、生embedding分散より小さい

**ステータス：** `HYPOTHESIS`

**実験設定：**
```
データ：
  同義群：「機密資料をカフェで読んでいい？」系 × 20文
  ノイズ群：「パスタ」「富士山」「Python」「スマホ」等（無関係10概念）
  文化差群：同一状況を日本・米国・欧州の表現で記述（各5文）

モデル：
  all-MiniLM-L6-v2（384次元）
  all-mpnet-base-v2（768次元）

測定：
  variance_embedding = var(embeddings_of_synonyms)
  variance_anchor    = var(anchor_coordinates_of_synonyms)
```

**実装ファイル：** `notebooks/poc_01_anchor_stability.ipynb`

**成功ライン：**
```
最低：同義群が安定クラスタ形成 + ノイズ群が明確に分離される
良好：variance_anchor / variance_embedding < 0.5
強い：文化差群（日米欧）でも崩壊しない
```

**進捗：** 未着手

---

### H-102：モデル間でアンカー座標の近傍順位が保たれる

**主張：** 異なるembeddingモデルでアンカー座標変換後の近傍ランキングが一致する

**ステータス：** `HYPOTHESIS`

**これが最重要指標。** モデルが変わっても「意味の順序」が保たれることを示す。

**測定：**
```python
# モデルAとBで同一テキスト群をアンカー座標化
# 各テキストの近傍top-5の順位一致率を計算
ranking_consistency = mean(
    rank_correlation(neighbors_A[i], neighbors_B[i])
    for i in texts
)
```

**成功ライン：**
```
最低：ranking_consistency > baseline（生embeddingの一致率）
良好：ranking_consistency > 0.8
強い：アンカー数を変えても順位が安定する
```

**進捗：** 未着手

---

### H-103：アンカー除去耐性（新規追加）

**主張：** アンカーを一部除去しても意味位置が大崩壊しない

**ステータス：** `HYPOTHESIS`

**なぜ重要か：** アンカー設計の頑健性を示す。「アンカーが安定している」という前提の強化証拠になる。

**測定：**
```python
# 全アンカーm個のうちk個をランダム除去
# 残り(m-k)個でのアンカー座標を再計算
# 元の座標との相関を測定
for k in [1, 2, 4]:  # 除去数を変えて実験
    coords_full    = anchor_transform(text, all_anchors)
    coords_reduced = anchor_transform(text, anchors_minus_k)
    robustness[k]  = correlation(coords_full, coords_reduced)
```

**成功ライン：**
```
最低：k=1除去で相関 > 0.9
良好：k=2除去で相関 > 0.8
強い：k=4除去でも意味的順位が保たれる
```

**進捗：** 未着手

---

### H-104：ノイズアンカー耐性（新規追加）

**主張：** 無関係なアンカーを混入させても意味クラスタが崩壊しない

**ステータス：** `HYPOTHESIS`

**測定：**
```python
# 正規アンカーにランダムベクトル（意味的に無関係）をN個追加
# クラスタ分離スコアの変化を測定
for n_noise in [1, 2, 4]:
    score_clean = cluster_separation(texts, clean_anchors)
    score_noisy = cluster_separation(texts, clean_anchors + noise_anchors_n)
    degradation[n_noise] = score_clean - score_noisy
```

**成功ライン：**
```
最低：ノイズ1個追加でクラスタ分離スコア低下 < 10%
良好：ノイズ2個追加でも分離スコア維持
```

**進捗：** 未着手

---

## PoC② — 長期整合性（Longitudinal Consistency Hypothesis）

> **PoCの問い：** bundle更新構造により、継時更新でも意味重心のドリフトが減少するか

---

### H-201：bundle更新は通常embedding平均よりdriftを抑制する

**主張：** 逐次更新時、bundle重心のdriftが生embeddingの移動平均より小さい

**ステータス：** `HYPOTHESIS`

**実験設定：**
```
初期状態：「企業セキュリティポリシー」bundle を構築
更新系列：新ルール・社内例外・業界変化を順次追加（10ステップ）

比較：
  baseline：通常embedding平均（毎ステップ再計算）
  bundle  ：bundle重心の加重更新

測定：
  cosine_drift(t, t+n) = distance(centroid_t, centroid_{t+n})
```

**実装ファイル：** `notebooks/poc_02_longitudinal.ipynb`

**成功ライン：**
```
最低：bundle更新のdrift < embedding平均のdrift
良好：drift 30%以上の低減
強い：10ステップ後も初期bundleとの整合性スコア > 0.8
```

**進捗：** 未着手

---

### H-202：Semantic Collapse率（新規追加）

**主張：** bundle更新では概念が別の意味に化けるsemantic collapseが起きにくい

**ステータス：** `HYPOTHESIS`

**定義：**
```python
# semantic collapse = 更新後の重心が
# 元の概念クラスタから外れ、別クラスタに近くなること

def semantic_collapse(centroid_before, centroid_after, all_clusters):
    original_cluster = nearest_cluster(centroid_before, all_clusters)
    new_cluster      = nearest_cluster(centroid_after, all_clusters)
    return original_cluster != new_cluster  # Trueなら崩壊
```

**成功ライン：**
```
最低：bundle更新でのcollapse率 < baseline（embedding平均）
良好：collapse率 < 5%（10ステップ × 10概念 = 100更新中5回未満）
```

**進捗：** 未着手

---

### H-203：更新後の整合性スコア維持

**主張：** 更新を重ねても初期bundleとの整合性スコアが一定以上を維持する

**ステータス：** `HYPOTHESIS`

**測定：**
```python
consistency_score(t) = cosine_similarity(centroid_t, centroid_0)
# これが単調減少しないことを確認
```

**成功ライン：** 10ステップ後の consistency_score > 0.7

**進捗：** 未着手

---

## PoC③ — 編集局所化（Localized Edit Hypothesis）

> **PoCの問い：** bundle単位更新により、局所編集が全体意味空間へ波及しにくくなるか

---

### H-301：編集影響がbundle内に局所化される

**主張：** あるbundleを更新した時、無関係な他bundleへの影響スコアが低い

**ステータス：** `HYPOTHESIS`

**実験設定：**
```
概念セット（例）：
  Apple（企業）、Apple（果物）、Google、Microsoft、
  セキュリティ、プライバシー、AI、クラウド

操作：
  Apple（企業）bundleに「Vision Pro」情報を追加

比較：
  baseline：embedding再学習 → 全空間の座標が変化
  bundle  ：Apple企業bundleのみ更新

測定（Edit Locality Score）：
  直接関連bundle（Apple果物、Microsoft等）への影響
  無関係bundle（セキュリティ、AI等）への影響
```

**実装ファイル：** `notebooks/poc_03_edit_locality.ipynb`

**成功ライン：**
```
最低：無関係bundleへの影響 < baselineの影響
良好：edit_locality_score = 
      (影響あるべきbundleの変化) / (全bundle変化量) > 0.8
強い：無関係bundleへの座標変化 < 0.05
```

**進捗：** 未着手

---

### H-302：Semantic Contamination（意味汚染）の抑制

**主張：** 局所更新による意味汚染（無関係概念への波及）がbundleで抑制される

**ステータス：** `HYPOTHESIS`

**定義：**
```python
semantic_contamination = mean(
    impact_score(edited_bundle, unrelated_bundle)
    for unrelated_bundle in unrelated_bundles
)
# 低いほど良い
```

**成功ライン：** semantic_contamination < baseline × 0.5

**進捗：** 未着手

---

## PoC④ — 効率性（Efficiency Hypothesis）

> **PoCの問い：** bundle構造がメモリ・トークン効率に寄与するか

---

### H-401：参照再構成はフル保存より少ないメモリで同等精度を達成する

**主張：** K個の属性ベクトルでの再構成が、フルembedding保存と同等の検索精度を出す

**ステータス：** `HYPOTHESIS`

**実験設定：**
```
K = 8, 16, 32（使用属性数）
d = 768（embedding次元）

測定：
  再構成誤差：cosine_similarity(x, x_hat)
  メモリ比：  K*d / (N*d) = K/N  ※Nは全概念数
  検索精度：  再構成後の近傍top-5のrecall@5
```

**実装ファイル：** `notebooks/poc_04_efficiency.ipynb`

**成功ライン：**
```
最低：K=32でcosine_similarity(x, x_hat) > 0.9
良好：K=16でrecall@5 > 0.8（フル保存と同等）
強い：K=8でも意味的順位が保たれる
```

**進捗：** 未着手

---

### H-402：bundle検索はフルコンテキストより少ないトークンで同等回答を得る

**主張：** LLMへ渡すコンテキストをbundle要約にすることでトークン削減できる

**ステータス：** `HYPOTHESIS`

**測定：**
```python
# baseline：全文コンテキストをLLMに渡す
# bundle  ：anchor座標 + sparse bundle reconstructionをLLMに渡す

token_reduction = 1 - (bundle_tokens / full_context_tokens)
answer_quality  = semantic_similarity(answer_bundle, answer_full)
```

**成功ライン：**
```
最低：token削減率 > 30% かつ answer_quality > 0.85
良好：token削減率 > 50% かつ answer_quality > 0.9
```

**進捗：** 未着手

---

## 実装設計仮説

### H-501：アンカー数 m = 8〜16 で十分

**主張：** m = 8〜16 で意味空間の主要軸を十分にカバーできる

**ステータス：** `HYPOTHESIS`

**候補アンカー（PoC①で使用する初期セット）：**
```yaml
anchors:
  - name: time      # 時間・時制・継続性
  - name: space     # 空間・場所・距離
  - name: number    # 数・量・規模
  - name: body      # 身体・感覚・物理的状態
  - name: risk      # 危険・損害・リスク
  - name: trust     # 信頼・権威・規範
  - name: rule      # 規則・制度・制約
  - name: emotion   # 感情・状態・心理
```

**注意：** アンカーのembeddingは複数の代表文の平均を使う（1語だと不安定）

**進捗：** 未着手

---

### H-502：整合性チェック閾値 τ

**現在の暫定値（PoC専用）：**
```
τ_high = 0.7   # これ以上: 正常
τ_low  = 0.4   # これ以下: HALLUCINATION_RISK
```

**ステータス：** `HYPOTHESIS`

**注意：** コードにハードコードしない。設定ファイルで管理する。

---

### H-503：加重重心の重み結合方法

**ステータス：** `HYPOTHESIS`

**懸念：** 積（×）では一つが低いと全体が潰れる。加重和（+）の方が安定する可能性。

**検証方法：** Phase 1以降でA/Bテスト

---

### H-504〜H-506：Bundle merge/split/品質監視閾値群

**ステータス：** `DEFERRED`（Phase 2以降で決定）

---

## Paper 2 — Semantic Stability 4軸フレームワーク + アンカー選定

> **研究の核心：** semantic stabilityを評価可能な問題へ変換する
>
> **Paper 2の正確な主張：**
> 「本研究では、semantic stabilityを4軸で整理する評価フレームワークを提案する。
>  今回のPoCでは、その一部について初期検証を行った。」
>
> **スコープ：** フレームワーク提案＋初期検証まで。完全検証はPaper 3以降。
>
> **最終更新：** 2026-05-20（アンカー = 安定意味領域として再設計）

---

### 重要な設計変更（2026-05-20）

旧設計（H-601〜H-603）の問題点：
- アンカーを「単語」として扱っていた
- 「ドメイン特化アンカー」はアンカーではなく意味基底（Semantic Basis）だった
- 4軸stability frameworkで測っていなかった

新設計の方針：
- アンカー = 安定意味領域（stable semantic region）として再定義
- 4軸stability scoreで候補を評価する
- 既存実験（汎用 vs 特化）は「安定性 vs 識別力のトレードオフ」として再解釈・流用

---

### H-601（修正）：安定意味領域として構築したアンカーは単語ベースより安定する

**主張：** 近傍意味群・類義語・多言語表現の平均として構築した安定意味領域が、
単一単語embeddingより高い安定性スコアを示す

**ステータス：** `HYPOTHESIS`

**実験設定：**
```
比較：
  A) 単語ベースアンカー：1単語のembedding
     例："danger" の embedding
  B) 安定意味領域アンカー：近傍意味群の平均
     例：["danger", "risk", "unsafe", "hazardous",
          "危険", "リスク", "threat"] の平均embedding
  C) 身体性ベースアンカー：身体感覚・基礎認知語群の平均
     例：["hot/cold", "near/far", "heavy/light"...] の平均

測定（4軸のうち測定可能な軸）：
  A. モデル間安定性：mpnet vs minilm でのアンカー座標相関
  D. 文脈摂動耐性：アンカー除去・ノイズ追加への耐性（H-103・H-104相当）
```

**実装ファイル：** `notebooks/paper2_01_semantic_region_anchor.ipynb`

**成功ライン：**
```
最低：安定意味領域 > 単語ベース（モデル間相関で）
良好：身体性ベース ≥ 抽象概念ベース（安定性スコアで）
強い：安定意味領域でH-102（モデル間ランキング）が改善
```

**進捗：** 未着手（旧H-601を廃止・再設計）

---

### H-602（修正）：4軸stability scoreがアンカー候補の選定基準として機能する

**主張：** 以下のstability scoreが高い概念ほど、
アンカーとして使った時のクラスタ安定性が高い

**ステータス：** `HYPOTHESIS`

**stability scoreの定義：**
```python
stability_score(anchor_candidate) =
    w1 * inter_model_consistency(anchor)    # A軸
    + w2 * perturbation_resistance(anchor)  # D軸
    # B軸（temporal）・C軸（cross-cultural）はPaper 3以降

# Paper 2で測定可能な暫定版：
stability_score_v1(anchor) =
    inter_model_correlation(anchor, mpnet, minilm)
    * perturbation_robustness(anchor)
```

**候補アンカー群の比較：**
```
身体性ベース：
  hot/cold, near/far, heavy/light, pain/comfort,
  bright/dark, fast/slow

時間認知ベース：
  past/future, before/after, duration, sequence

空間認知ベース：
  up/down, inside/outside, distance, direction

数量認知ベース：
  many/few, increase/decrease, zero/infinity

危険回避ベース：
  safe/dangerous, threat, protection, harm
```

**成功ライン：**
```
最低：stability_score上位候補でクラスタ安定性が向上する
良好：身体性ベース候補が抽象概念候補より安定性スコアが高い
強い：stability_score → クラスタ安定性の相関が有意
```

**進捗：** 未着手

---

### H-603（修正）：安定意味領域アンカーでH-102のトレードオフが改善する

**主張：** 安定意味領域として構築したアンカーは、
単語ベースアンカーより「クラスタ安定性」と「モデル間ランキング一致率」の
両立度が高い

**ステータス：** `HYPOTHESIS`

**背景（重要）：**
今日の実験（paper2_01）で判明したこと：
```
汎用アンカー（単語ベース）：
  クラスタ安定性◎（ratio 0.047〜0.067）
  ドメイン識別力△（分離スコア 0.166）

ドメイン特化概念（意味基底）：
  クラスタ安定性△（ratio 0.110〜0.235）
  ドメイン識別力◎（分離スコア 0.739）
```

これは「単語 vs 意味基底」の比較だった。
「安定意味領域アンカー」はまだ試していない。

**測定：**
```python
# Paper 1の結果（baseline）
ranking_consistency_generic = 0.572〜0.643

# Paper 2で測定
ranking_consistency_semantic_region = ?  # これを確認
cluster_stability_semantic_region    = ?  # ratio

# 両立度（stability × discriminability）
tradeoff_score = cluster_stability × ranking_consistency
```

**成功ライン：**
```
最低：semantic_region アンカーのtrade-off scoreが単語ベースより高い
良好：ranking_consistency > 0.7 かつ cluster_ratio < 0.1
```

**進捗：** 未着手

---

### H-604（新規）：安定性 vs 識別力のトレードオフは用途依存である

**主張：** 今日の実験で発見した「汎用（安定）vs 特化（識別力）」のトレードオフは、
用途に応じた使い分けの指針として定式化できる

**ステータス：** `CONFIRMED`（実験済み・論文記載済み）

**実験結果（paper2_01より）：**
```
クラスタ安定性（低いほど安定）：
  汎用アンカー : 0.047〜0.067  ← 安定性重視ならこちら
  特化概念    : 0.110〜0.235

ドメイン間分離スコア（高いほど識別力高い）：
  汎用アンカー : 0.166
  特化概念    : 0.739（4.4倍）← 識別力重視ならこちら
```

**設計指針（Paper 2で提案）：**
```
長期記憶・概念管理  → 汎用アンカー（安定性優先）
ドメイン分類・検索  → 特化意味基底（識別力優先）
両立が必要な場合    → 今後の研究課題（H-603）
```

**進捗：** ✅ 実験完了（paper2_01_domain_anchor.ipynb）

---

## 確定済み仮説（→spec.mdに移動済み）

| 仮説ID | 内容 | 確定日 | 数値 |
|--------|------|--------|------|
| H-101 | アンカー座標クラスタ安定性 | 2026-05-18 | ratio 0.052〜0.081 |
| H-103 | アンカー除去耐性 | 2026-05-18 | 相関1.0000 |
| H-104 | ノイズアンカー耐性 | 2026-05-18 | 劣化0.00% |
| H-201 | bundle更新のdrift削減 | 2026-05-19 | 38.6%削減 |
| H-202 | semantic collapse率 | 2026-05-19 | 0% |
| H-203 | 整合性スコア維持 | 2026-05-19 | 0.931 |
| H-301 | 編集影響局所化 | 2026-05-19 | bundle < baseline |
| H-302 | semantic contamination | 2026-05-19 | ρ<0.15で32.6% |
| H-401 | 参照再構成精度 | 2026-05-19 | K=64で0.963 |
| H-402 | メモリ効率 | 2026-05-19 | 91.7%削減 |

---

## 棄却済み仮説

| 仮説ID | 内容 | 棄却理由 | 棄却日 |
|--------|------|---------|--------|
| H-102 | モデル間ランキング一致率改善 | 汎用アンカーではドメイン不適合。トレードオフとして論文記載。Paper 2で再検証（H-603） | 2026-05-18 |
