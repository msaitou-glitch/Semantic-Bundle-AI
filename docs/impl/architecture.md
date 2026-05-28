# architecture.md — システム構造・レイヤー設計

> このファイルにはシステムの構造・レイヤー・コンポーネント関係を書く。
> DB詳細はdb.mdへ。APIエンドポイントはapi.mdへ。
> 数理定義はmath.mdへ。実装コードの規則はimpl_guide.mdへ。
>
> 対象：Claude Code・開発者・研究者
> Claude Codeは実装前に必ず読む
> 変更頻度：中（Phase進行で拡張される）

---

## 1. 全体レイヤー構造

```
┌─────────────────────────────────────────┐
│  Layer 5: Application / Agent / API      │
│  (出力・アクション・外部インターフェース)  │
├─────────────────────────────────────────┤
│  Layer 4: Bundle Network                 │
│  (bundle間グラフ・意味ネットワーク)       │
├─────────────────────────────────────────┤
│  Layer 3: Semantic Bundle Store          │ ← 意味束AIの中心
│  (bundle生成・保存・整合性管理)           │
├─────────────────────────────────────────┤
│  Layer 2: Anchor Coordinate              │
│  (安定座標変換)                           │
├─────────────────────────────────────────┤
│  Layer 1: Embedding                      │
│  (LLM / Sentence Transformer)            │
├─────────────────────────────────────────┤
│  Layer 0: Raw Input                      │
│  (テキスト・データ)                       │
└─────────────────────────────────────────┘
```

意味束AIが担当するのは **Layer 2〜4** である。
Layer 1（Embedding）はLLMまたはsentence-transformersに委譲する。
Layer 5（Application）は意味束AIのAPIを使うが、意味束AIの責務ではない。

---

## 2. データフロー：bundle生成

```
User Input / Document
        │
        ▼
[Layer 1] Embedding Model
        │  e ∈ R^d
        ▼
[Layer 2] Anchor Coordinate Projection
        │  z ∈ R^m
        ▼
[Layer 3] Bundle Generator
        │  ├── Core Identification
        │  ├── Attribute Extraction
        │  └── Centroid Calculation
        ▼
[Layer 3] Consistency Check
        │  └── Compare with existing bundles
        ▼
[Layer 3] Bundle Store (DB)
        │  ├── Vector Store  (embedding, anchor_coords)
        │  ├── Relational DB (metadata, version, source)
        │  └── Graph Store   (bundle relationships)
        ▼
  bundle_id を返す
```

---

## 3. データフロー：推論時参照

```
User Query
        │
        ▼
[Layer 1] Embedding Model
        │  e_query ∈ R^d
        ▼
[Layer 2] Anchor Coordinate Projection
        │  z_query ∈ R^m
        ▼
[Layer 3] Bundle Search
        │  └── 関連bundle群を取得
        ▼
[Layer 3] Context Centroid Construction
        │  └── 文脈重心を構築
        ▼
[Layer 1] LLM に構造化contextとして渡す
        │
        ▼
[Layer 1] LLM Response Generation
        │
        ▼
[Layer 3] Semantic Consistency Check
        │  └── 出力 vs 参照bundleの整合性スコア
        ▼
出力 / bundle更新判断
```

---

## 4. コンポーネント構成

### 4.1 Core Components（Phase 0〜1で実装）

```
SemanticBundleEngine
├── AnchorStore         # アンカー集合の管理
│   ├── load_anchors()
│   ├── add_anchor()
│   └── check_anchor_drift()
│
├── BundleGenerator     # bundle生成
│   ├── create_bundle()
│   ├── extract_attributes()
│   └── calculate_centroid()
│
├── BundleStore         # bundle保存・検索
│   ├── save_bundle()
│   ├── search_bundles()
│   ├── get_bundle()
│   └── update_bundle()
│
└── ConsistencyEngine   # 整合性チェック
    ├── check_consistency()
    ├── detect_drift()
    └── flag_hallucination_risk()
```

### 4.2 Loop Components（Phase 1〜2で実装）

```
SemanticLoopEngine
├── DriftMonitor        # ドリフト監視
├── BundleQualityMonitor # bundle品質監視
├── SelfHealingEngine   # 自動修復
└── RecompressionEngine # 再圧縮
```

### 4.3 Network Components（Phase 3で実装）

```
BundleNetworkEngine
├── GraphBuilder        # グラフ構築
├── NetworkSearch       # グラフ検索
└── NetworkVisualizer   # 可視化
```

---

## 5. DB構成（概要）

詳細は `impl/db.md` を参照。

```
Vector DB    : embedding・anchor座標の類似検索
Relational DB: bundle metadata・バージョン管理
Graph DB     : bundle間関係（Phase 3から）
```

Phase 0〜1では Vector DB + Relational DB のみ使用。

---

## 6. LLMとの接続点

```
接続点1: Embedding生成
  LLM / sentence-transformers → embedding → Anchor Coordinate変換

接続点2: コンテキスト注入
  Bundle Store → 関連bundle → LLMのプロンプトに構造化して渡す

接続点3: 出力チェック
  LLM出力 → bundle化 → ConsistencyEngineで整合性確認
```

意味束AIはLLMのファインチューニングや内部変更を行わない。

---

## 7. RAGとの接続点

```
RAG（情報取得）
    ↓ チャンク群
意味束AI（意味統合）
    ↓ bundle化・整合性管理
LLM（言語生成）
```

意味束AIはRAGの出力を受け取り、bundle化して整理する上位レイヤー。

---

## 8. Phase別実装範囲

| コンポーネント | Phase 0 | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|---|
| AnchorStore | ✅ | ✅ | ✅ | ✅ |
| BundleGenerator | ✅ | ✅ | ✅ | ✅ |
| BundleStore | PoC | DB実装 | 拡張 | 拡張 |
| ConsistencyEngine | 基本 | 拡張 | 完全 | 完全 |
| DriftMonitor | - | 基本 | 拡張 | 完全 |
| SelfHealingEngine | - | - | 基本 | 拡張 |
| BundleNetworkEngine | - | - | - | 基本 |
| API Layer | - | 基本 | 拡張 | 完全 |

---

## 9. 非機能要件（Phase 0〜1）

- bundle検索レイテンシ: < 100ms（目標）
- bundle生成レイテンシ: < 500ms（目標）
- スケール: Phase 1は〜10,000 bundles を想定
- 可用性: Phase 0〜1はシングルインスタンスで可
