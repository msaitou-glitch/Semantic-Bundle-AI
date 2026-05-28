# db.md — DB設計

> このファイルにはDB schema・テーブル定義・インデックス・選定理由を書く。
> システム構造はarchitecture.mdへ。APIはapi.mdへ。
>
> 対象：Claude Code・開発者
> Claude Codeは実装時に必ず参照する
> 変更頻度：中（Phase進行でスキーマが拡張される）

---

## DB構成の全体方針

| DB種別 | 役割 | Phase 0〜1 | Phase 2〜 |
|--------|------|------------|----------|
| Vector DB | embedding・アンカー座標の類似検索 | pgvector | pgvector / FAISS |
| Relational DB | bundle metadata・バージョン管理 | PostgreSQL | PostgreSQL |
| Graph DB | bundle間関係・意味ネットワーク | 使用しない | Neo4j |

---

## Phase 0：インメモリ（PoC専用）

Phase 0ではDBを使用しない。numpy/dict でインメモリ管理。

```python
# Phase 0 のデータ構造（PoC用）
bundle_store: dict[str, dict] = {}
anchor_store: list[np.ndarray] = []
```

---

## Phase 1：PostgreSQL + pgvector

### テーブル設計

#### `bundles` テーブル

```sql
CREATE TABLE bundles (
    bundle_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    core_embedding  VECTOR(768),        -- pgvector型
    context_centroid VECTOR(768),       -- pgvector型
    source          TEXT NOT NULL,
    raw_summary     TEXT,               -- 全文ではなく要約のみ
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version         INTEGER NOT NULL DEFAULT 1,
    confidence      FLOAT NOT NULL DEFAULT 1.0
                    CHECK (confidence >= 0.0 AND confidence <= 1.0),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);

-- インデックス
CREATE INDEX idx_bundles_core_embedding
    ON bundles USING ivfflat (core_embedding vector_cosine_ops);

CREATE INDEX idx_bundles_updated_at
    ON bundles (updated_at DESC);
```

#### `bundle_attributes` テーブル

```sql
CREATE TABLE bundle_attributes (
    attr_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bundle_id       UUID NOT NULL REFERENCES bundles(bundle_id) ON DELETE CASCADE,
    attr_embedding  VECTOR(768),
    attr_label      TEXT,               -- 属性の意味ラベル（オプション）
    attr_index      INTEGER NOT NULL,   -- bundle内での順序
    weight          FLOAT DEFAULT 1.0
);

CREATE INDEX idx_bundle_attributes_bundle_id
    ON bundle_attributes (bundle_id);
```

#### `anchor_coordinates` テーブル

```sql
CREATE TABLE anchor_coordinates (
    coord_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bundle_id       UUID NOT NULL REFERENCES bundles(bundle_id) ON DELETE CASCADE,
    anchor_name     TEXT NOT NULL,
    similarity      FLOAT NOT NULL
                    CHECK (similarity >= -1.0 AND similarity <= 1.0),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_anchor_coords_unique
    ON anchor_coordinates (bundle_id, anchor_name);
```

#### `bundle_history` テーブル

```sql
CREATE TABLE bundle_history (
    history_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bundle_id       UUID NOT NULL REFERENCES bundles(bundle_id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    change_type     TEXT NOT NULL,      -- 'create' | 'update' | 'merge' | 'split'
    change_summary  TEXT,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    changed_by      TEXT                -- system / user_id
);

CREATE INDEX idx_bundle_history_bundle_id
    ON bundle_history (bundle_id, version DESC);
```

#### `anchors` テーブル（アンカー管理）

```sql
CREATE TABLE anchors (
    anchor_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    anchor_name     TEXT UNIQUE NOT NULL,
    embedding       VECTOR(768),
    description     TEXT,
    stability_score FLOAT DEFAULT 1.0,  -- アンカー安定性スコア
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE
);
```

---

### embedding次元の設定

デフォルト：`768`（sentence-transformers `all-mpnet-base-v2`）

モデル変更時はここを更新する（マイグレーション必要）：

| モデル | 次元数 |
|--------|-------|
| all-mpnet-base-v2 | 768 |
| all-MiniLM-L6-v2 | 384 |
| text-embedding-3-small | 1536 |
| text-embedding-3-large | 3072 |

---

### クエリ例

#### 類似bundle検索

```sql
SELECT bundle_id, source, confidence,
       1 - (core_embedding <=> $1::vector) AS similarity
FROM bundles
WHERE is_active = TRUE
ORDER BY core_embedding <=> $1::vector
LIMIT 10;
```

#### アンカー座標での検索

```sql
SELECT b.bundle_id, b.source,
       ac.anchor_name, ac.similarity
FROM bundles b
JOIN anchor_coordinates ac ON b.bundle_id = ac.bundle_id
WHERE ac.anchor_name = 'time'
  AND ac.similarity > 0.7
ORDER BY ac.similarity DESC
LIMIT 20;
```

---

## Phase 3：Graph DB（Neo4j）

### ノード定義

```cypher
(:Bundle {
    bundle_id: String,   // UUID
    label: String,       // 人間可読ラベル
    created_at: DateTime
})

(:Anchor {
    anchor_name: String,
    stability: Float
})
```

### リレーション定義

```cypher
(:Bundle)-[:SIMILAR_TO {weight: Float, direction: String}]->(:Bundle)
(:Bundle)-[:CAUSED_BY {confidence: Float}]->(:Bundle)
(:Bundle)-[:HIERARCHICAL {type: 'parent'|'child'}]->(:Bundle)
(:Bundle)-[:REFERENCES]->(:Bundle)
(:Bundle)-[:ANCHORED_TO {similarity: Float}]->(:Anchor)
```

---

## マイグレーション方針

- スキーマ変更はマイグレーションファイルとして管理（`scripts/migrations/`）
- `bundle_id` は永続的。変更禁止。
- embedding次元変更は全データの再計算が必要（要注意）

---

## 設定ファイル

```python
# config/db.py
DB_CONFIG = {
    "host": "localhost",
    "port": 5432,
    "database": "semantic_bundle",
    "embedding_dim": 768,
    "vector_index_type": "ivfflat",  # or 'hnsw'
}

ANCHOR_NAMES = [
    "time", "space", "number", "body",
    "risk", "trust", "rule", "emotion"
]
# アンカー名はここで管理。コードにハードコードしない。
```
