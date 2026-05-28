# loops.md — AI Loop構造・自律改善設計

> このファイルには意味束AIの自律的な品質監視・更新・修復ループを書く。
> 確定仕様はspec.mdへ移動済み。仮説段階の閾値はhypotheses.mdへ。
> 数式はmath.mdを参照する。
>
> 対象：Claude Code・開発者
> Claude Codeは Loop実装時に参照する
> 変更頻度：中（Phase進行で実装が深まる）

---

## 1. Loopの全体像

意味束AIは静的なストレージではなく、定期的に自律改善を行う。

```
┌─────────────────────────────────────────┐
│           Semantic OS Loops              │
│                                          │
│  ① 整合性チェックLoop（リアルタイム）   │
│  ② ドリフト検知Loop（定期）             │
│  ③ bundle品質監視Loop（定期）           │
│  ④ 再圧縮Loop（低頻度）                │
│  ⑤ Self-HealingLoop（異常検知時）       │
└─────────────────────────────────────────┘
```

---

## 2. ① 整合性チェックLoop（Consistency Check Loop）

**タイミング：** LLM出力生成のたびに実行（リアルタイム）

```
LLM出力
    │
    ▼
bundle化（出力embedding → アンカー座標 → bundle）
    │
    ▼
参照bundle群と整合性スコア計算
    │ score = sim(output_bundle, reference_bundle)
    ▼
判定：
  score >= τ_high  → 正常出力。通過。
  τ_low <= score < τ_high → 警告フラグ。人間レビュー候補。
  score < τ_low    → HALLUCINATION_RISKフラグ。出力を保留。

(τ の値は hypotheses.md #H-007 参照)
```

**実装ファイル：** `src/loops/consistency.py`

---

## 3. ② ドリフト検知Loop（Drift Detection Loop）

**タイミング：** 定期実行（interval: hypotheses.md #H-008 参照）

### Embedding Drift 検知

```python
for concept in tracked_concepts:
    z_current = to_anchor_coordinates(embed(concept))
    z_baseline = load_baseline_coordinates(concept)
    drift_score = L2_distance(z_current, z_baseline)

    if drift_score > θ_drift:
        flag_drift(concept, drift_score)
        trigger_reanchor(concept)
```

### Anchor Self-Drift 検知

アンカー自体もドリフトしうる（math.md 6.3 参照）

```python
for anchor in anchor_set:
    drift = distance(anchor.embedding_now, anchor.embedding_baseline)
    if drift > θ_anchor_drift:
        options:
            1. anchor_update()      # アンカーを更新
            2. adaptive_anchor()    # 適応アンカーへ移行
            3. keep_old_anchor()    # 旧アンカーを保持・分岐
```

アンカードリフトは慎重に扱う。アンカーが動くと全bundle座標が変わる。

**実装ファイル：** `src/loops/drift.py`

---

## 4. ③ Bundle品質監視Loop（Quality Monitor Loop）

**タイミング：** 定期実行

監視項目：

| 項目 | 測定方法 | 閾値 |
|------|---------|------|
| bundle内整合性 | 属性ベクトル間の平均類似度 | hypotheses.md #H-009 |
| コアからの距離 | 各属性 vs コアのコサイン距離 | hypotheses.md #H-010 |
| アンカーからのズレ | anchor_coordinatesの変化量 | hypotheses.md #H-011 |
| 類似bundle重複 | 他bundleとの類似度最大値 | hypotheses.md #H-005 |
| 更新頻度 | 過去N日の更新回数 | 設定ファイル |
| 参照頻度 | 過去N日の参照回数 | 設定ファイル |

品質スコアが低いbundleは修復Loopへ送る。

**実装ファイル：** `src/loops/quality_monitor.py`

---

## 5. ④ 再圧縮Loop（Recompression Loop）

**タイミング：** 低頻度（bundles数が閾値超過時）

類似bundleの統合（merge）：
```python
# math.md 10.1 参照
for (bundle_a, bundle_b) in candidate_pairs:
    if distance(bundle_a.core, bundle_b.core) < θ_merge:
        merged = merge_bundles(bundle_a, bundle_b)
        save_bundle(merged)
        archive_bundles([bundle_a, bundle_b])
```

bundle分割（split）：
```python
# math.md 10.2 参照
for bundle in active_bundles:
    if has_multiple_centroids(bundle, θ_split):
        [bundle_x, bundle_y] = split_bundle(bundle)
        save_bundles([bundle_x, bundle_y])
        archive_bundle(bundle)
```

**注意：** merge/splitの正式仕様は非公開仕様。
Phase 2以降でhypotheses.mdのステータスを確認してから実装する。

**実装ファイル：** `src/loops/recompression.py`

---

## 6. ⑤ Self-Healing Loop

**タイミング：** 異常検知時（品質監視Loopからトリガーされる）

```
bundle異常検知
    │
    ▼
修復オプションの評価：
  1. 近傍bundleとの統合       → merge_bundles()
  2. 旧バージョンへのロールバック → rollback_bundle(version=N)
  3. アンカー再射影            → reproject_to_anchors()
  4. LLMによる説明生成         → generate_explanation()  [Phase 2]
  5. 人間承認待ちキューへ追加  → queue_for_review()

修復後:
  bundle_history に記録
  品質監視Loopで再確認
```

**実装ファイル：** `src/loops/self_healing.py`

---

## 7. Bundle更新フロー

新しい入力がある場合の更新判断：

```
New Input
    │
    ▼
既存bundleと比較
    │
    ├─ 高類似 (> θ_update_high)
    │   → 既存bundleを更新（重心再計算）
    │
    ├─ 中類似 (θ_update_low < x <= θ_update_high)
    │   → 既存bundleに属性を追加
    │
    └─ 低類似 (<= θ_update_low)
        → 新規bundleを生成
```

更新時は必ず `bundle_history` に記録する。

---

## 8. Loop間の依存関係

```
整合性チェックLoop
    └→ 異常検知時 → Self-Healing Loop

品質監視Loop
    └→ 再圧縮候補発見 → 再圧縮Loop
    └→ 異常検知時    → Self-Healing Loop

ドリフト検知Loop
    └→ アンカードリフト検知 → AnchorStore更新
    └→ 概念ドリフト検知    → Bundle更新フロー
```

---

## 9. Phase別実装優先度

| Loop | Phase 0 | Phase 1 | Phase 2 |
|------|---------|---------|---------|
| 整合性チェック | 基本版 | 完全版 | 拡張 |
| ドリフト検知 | - | 基本版 | 完全版 |
| 品質監視 | - | 基本版 | 完全版 |
| 再圧縮 | - | - | 基本版 |
| Self-Healing | - | - | 基本版 |
