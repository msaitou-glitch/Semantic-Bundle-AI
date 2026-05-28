# roadmap.md — 統合ロードマップ

> このファイルには開発・研究・展開の全フェーズを統合して管理する。
> 技術仕様はspec.mdへ。論文戦略はresearch_strategy.mdへ。外部展開はoutreach.mdへ。
> このファイルは「何がいつ起きるか」の全体地図。
>
> 対象：自分・開発者・投資家
> Claude Codeは現在Phaseの開発タスクのみ参照する
> 変更頻度：中（Phase完了時に更新）

---

## 現在地

```
[✅ SSRN公開]  →  [🔄 Phase 0: PoC]  →  [Phase 1]  →  [Phase 2]  →  [Phase 3+]
```

**現在：Phase 0 実行中**

---

## 3軸の並行管理

```
軸A：開発（実装）  →  spec.md / impl/ 参照
軸B：研究（論文）  →  research_strategy.md 参照
軸C：展開（発信）  →  outreach.md 参照
```

---

## Phase 0 — 概念PoC【現在】

**主軸：軸A + 軸B準備**

### 開発タスク
- [ ] `src/core/bundle.py`
- [ ] `src/core/anchor.py`
- [ ] `src/core/coordinate.py`
- [ ] `notebooks/poc_01_anchor_stability.ipynb`
- [ ] `notebooks/poc_02_model_consistency.ipynb`
- [ ] `notebooks/poc_03_reconstruction.ipynb`
- [ ] `notebooks/poc_04_edit_locality.ipynb`
- [ ] `tests/test_bundle.py` / `tests/test_anchor.py`

### 完了条件
- [ ] H-001：アンカー座標分散 < embedding分散の50%
- [ ] H-003：モデル間座標相関 > 0.8
- [ ] H-002：参照再構成コサイン類似度 > 0.9
- [ ] H-004：1概念更新時の他bundle影響スコア < 0.1

### 研究タスク
- [ ] 既存研究リスト作成（Anchored Embedding / Drift Detection / KG+LLM）
- [ ] 差分マトリクス作成（research_strategy.md Paper 1の表を充実させる）
- [ ] arXiv投稿カテゴリ決定（cs.CL or cs.AI）
- [ ] Google Colab対応確認

---

## Phase 1 — 最小Bundle Store + 公開準備

**主軸：軸A + 軸C**

**前提：** Phase 0完了

### 開発タスク
- [ ] PostgreSQL + pgvector セットアップ
- [ ] `src/storage/vector_store.py`
- [ ] `src/storage/relational.py`
- [ ] `src/api/routes.py`（基本エンドポイント）
- [ ] 整合性チェックLoop（基本版）
- [ ] `impl/api.md` 作成

### 研究タスク
- [ ] Paper 1（PoC論文）執筆・arXiv投稿

### 展開タスク
- [ ] GitHubリポジトリ名確定・作成
- [ ] PyPIパッケージ名仮押さえ（`semantic-bundle-ai`）
- [ ] GitHub public公開
- [ ] README完成（Quick Start + PoC結果 + 論文リンク）
- [ ] r/MachineLearning 初投稿（arXiv公開と同時）

---

## Phase 2 — RAG接続 + Loop完全化 + コミュニティ

**主軸：軸A + 軸C**

**前提：** Phase 1完了

### 開発タスク
- [ ] Dify / LangChain / LlamaIndex 接続
- [ ] ドリフト検知Loop実装
- [ ] 品質監視Loop実装
- [ ] Bundle merge基本版
- [ ] デモ環境構築（公開URL）

### 展開タスク
- [ ] Python SDK α版公開（pip install）
- [ ] r/LocalLLaMA 投稿
- [ ] Hacker News（Show HN）投稿
- [ ] Discord開設（GitHub Stars 100以上が条件）

### 研究タスク
- [ ] 共同研究者コンタクト開始
- [ ] Paper 2（アーキテクチャ論文）構想開始

---

## Phase 3 — GraphDB + API提供 + 共同研究

**前提：** Phase 2完了

- [ ] Neo4j接続・Bundle Network可視化
- [ ] SDK β版 → 1.0正式リリース
- [ ] ドキュメントサイト公開
- [ ] 共同研究正式開始

---

## Phase 4 — 事業化

**前提：** Phase 3完了

→ 詳細は `business/monetization.md` / `ops/research_strategy.md`

---

## 全体タイムライン（3軸統合）

| マイルストーン | 軸 | 状態 |
|---|---|---|
| SSRN英語版公開 | B | ✅ 完了 |
| PoC実験（4軸） | A | 🔄 進行中 |
| GitHub public | C | 待機中 |
| arXiv Paper 1 | B | 待機中 |
| r/ML 初投稿 | C | 待機中 |
| Bundle Store | A | 待機中 |
| SDK α版 | C | 待機中 |
| Hacker News | C | 待機中 |
| 共同研究開始 | B | 待機中 |
| Discord開設 | C | 待機中 |
| ライセンス交渉 | Biz | 将来 |
| 企業SaaS | Biz | 将来 |

---

## 今すぐ決めるべき事項

```
[ ] GitHubリポジトリ名を確定する
    候補: semantic-bundle-ai / semantic-bundle / sbai

[ ] arXiv投稿カテゴリを決める
    推奨: cs.CL（意味・言語が中心のため）

[ ] PyPIパッケージ名を仮押さえする
    候補: semantic-bundle-ai

[ ] PoCノートブックをGoogle Colabで動くか確認する
```

---

## 研究ロードマップ（2026-05-19 更新）

詳細は `ops/research_strategy.md` を参照。

| Stage | 内容 | 期間 | 状態 |
|-------|------|------|------|
| Stage 0 | Paper 0（理論）+ Paper 1（PoC） | 完了 | ✅ |
| Stage 1 | Paper 2（ドメイン適応アンカー） | 1〜2ヶ月 | 🔄 着手中 |
| Stage 2 | Paper 3（大規模検証・共著） | 2〜4ヶ月 | 待機中 |
| Stage 3 | Paper 4（Agent応用） | 4〜8ヶ月 | 待機中 |
| Stage 4 | Paper 5（査読投稿） | 8〜12ヶ月 | 待機中 |
