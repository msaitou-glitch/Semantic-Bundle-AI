# Semantic Bundle AI — ドキュメントOS

> このREADMEは「どのファイルに何が書かれているか」を管理する索引である。
> 新しいファイルを追加するときは必ずここを更新すること。

---

## ディレクトリ構造

```
docs/
├── README.md               ← 今ここ（索引・管理方針）
│
├── core/                   ← 意味束AI本体の理論・数理（変更頻度：低）
│   ├── vision.md           ← 意味束AIの問題意識・技術的立場（文明論は書かない）
│   ├── thesis.md           ← 理論の核心・新規性・既存研究との差
│   ├── math.md             ← 数理モデル・定式化
│   ├── glossary.md         ← 用語定義（全ファイルの共通参照元）
│   └── theory_map.md       ← 斉藤理論群の接続マップ（文明論・構造心理学・感情推論API退避先）
│                             ※ 技術仕様・論文・実装では参照しない
│
├── impl/                   ← 実装仕様（変更頻度：中）
│   ├── architecture.md     ← システム構造・レイヤー設計
│   ├── spec.md             ← 確定仕様（実装の確定事項）
│   ├── db.md               ← DB設計（Vector/Relational/Graph）
│   ├── api.md              ← API仕様（エンドポイント・スキーマ）★追加
│   ├── impl_guide.md       ← Claude Code向け実装ガイド
│   └── poc_guide.md        ← PoC①〜④の実験設計・実装手順・評価方法
│
├── ops/                    ← 運用・Loop・戦略（変更頻度：中高）
│   ├── loops.md            ← AI Loop構造・自律改善設計
│   ├── roadmap.md          ← 開発・研究・展開の統合ロードマップ（3軸）
│   ├── hypotheses.md       ← 仮説一覧・検証方法・進捗
│   ├── research_strategy.md← 論文パイプライン・共同研究・ライセンス戦略
│   └── outreach.md         ← GitHub・Reddit・SDK・コミュニティ展開戦略
│
├── business/               ← 事業・マネタイズ（変更頻度：中）
│   ├── monetization.md     ← 収益モデル・市場・顧客
│   └── barriers.md         ← 参入障壁・知財・競合分析
│
└── meta/                   ← ドキュメント管理・プロンプト
    ├── CLAUDE.md           ← Claude Code最初に読む統合ガイド
    ├── prompts.md          ← 標準プロンプト集
    └── decisions.md        ← 設計判断ログ（ADR形式）
```

---

## Claude Codeが読む順番

```
1. meta/CLAUDE.md           ← 必須・最初に読む
2. core/glossary.md         ← 用語の共通認識
3. impl/spec.md             ← 確定仕様の把握
4. impl/architecture.md     ← 構造の理解
5. impl/impl_guide.md       ← 実装ルール
6. impl/db.md               ← DBスキーマ
7. impl/api.md              ← APIエンドポイント
```

思想・理論（core/）はClaude Codeが参照する必要があれば読む。
ビジネス系（business/）は実装時には不要。

---

## 人間（研究者・投資家・共同開発者）が読む順番

| 目的 | 読む順 |
|------|--------|
| プロジェクト理解 | vision → thesis → glossary |
| 研究協力 | thesis → math → hypotheses → spec |
| 開発参加 | CLAUDE.md → spec → architecture → impl_guide |
| 投資・事業評価 | vision → thesis → monetization → barriers → roadmap |

---

## MVP時点で必須のファイル（Phase 0〜1）

- [x] `meta/CLAUDE.md`
- [x] `core/glossary.md`
- [x] `core/thesis.md`
- [x] `impl/spec.md`
- [x] `impl/architecture.md`
- [x] `impl/db.md`
- [x] `ops/roadmap.md`
- [x] `ops/research_strategy.md`
- [x] `ops/outreach.md`

## Phase 2以降で追加するファイル

- [ ] `impl/api.md`（API化開始時）
- [ ] `ops/hypotheses.md`（PoC開始後）→ 既に作成済み
- [ ] `business/monetization.md`（外部公開前）→ 既に作成済み
- [ ] `business/barriers.md`（競合分析時）
- [ ] `meta/prompts.md`（Claude Code運用が定常化した時）

---

## 「これは混ぜるな」絶対ルール

| 混ぜてはいけない組み合わせ | 理由 |
|---|---|
| 世界観 × 仕様 | 哲学は変わらないが仕様は変わる |
| 数理モデル × DB設計 | 抽象と実装の責務が違う |
| 仮説 × 確定事項 | 信頼度が違う。混在すると実装が壊れる |
| APIエンドポイント × アーキテクチャ | 粒度が違う |
| ループ設計 × マネタイズ | 技術とビジネスを分離 |
| 将来構想 × MVP仕様 | 今やることと将来やることを分離 |
| **非公開仕様 × docs/** | **private-specs/に保管。docs/にはシグネチャのみ記載** |
| **構造心理学 × 意味束AI仕様** | **姉妹理論だが独立理論。論文・仕様書に混入禁止** |
| **感情推論API × 意味束AI仕様** | **将来連携可能だが現時点では別プロジェクト** |

---

## ドキュメント更新ルール

1. **仕様変更は `spec.md` に先に書く**。コードより先。
2. **用語追加は `glossary.md` を先に更新**。他ファイルはglossaryを参照する。
3. **仮説は `hypotheses.md` に書く**。確定したら `spec.md` へ移動。
4. **設計判断は `decisions.md` に記録**。なぜそう決めたかを残す。
5. **このREADMEは構造変更のたびに更新**。索引が古くなると全体が崩壊する。
