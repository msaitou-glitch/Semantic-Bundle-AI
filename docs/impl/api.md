# api.md — API仕様

> このファイルにはFastAPIエンドポイント・リクエスト/レスポンススキーマを書く。
> 確定仕様はspec.mdへ。DB設計はdb.mdへ。
> 内部アルゴリズムの詳細はここに書かない（private-specs/参照）。
>
> 対象：Claude Code・開発者・API利用者
> Claude Codeは実装時に必ず参照する
> 変更頻度：中（Phase 1から本格化）

---

## 基本方針

```
公開するもの：エンドポイント・スキーマ・評価指標の定義
公開しないもの：束演算の内部アルゴリズム・時間ダイナミクス詳細
```

---

## エンドポイント一覧

| メソッド | パス | 説明 | Phase |
|---------|------|------|-------|
| POST | `/bundle/create` | bundle生成 | 1 |
| POST | `/bundle/search` | 類似bundle検索 | 1 |
| GET  | `/bundle/{id}` | bundle取得 | 1 |
| PUT  | `/bundle/{id}` | bundle更新 | 1 |
| DELETE | `/bundle/{id}` | bundle削除 | 1 |
| POST | `/bundle/merge` | bundle統合（束演算） | 1 |
| POST | `/bundle/project` | 軸射影（束演算） | 1 |
| POST | `/bundle/compress` | bundle圧縮（束演算） | 1 |
| POST | `/bundle/reconstruct` | 参照再構成（束演算） | 1 |
| POST | `/bundle/consistency` | 整合性チェック | 1 |
| POST | `/bundle/drift` | ドリフト検知 | 2 |
| GET  | `/bundle/{id}/metrics` | 評価指標取得 | 1 |

---

## スキーマ定義

### SemanticBundle（レスポンス共通）

```json
{
  "bundle_id": "550e8400-e29b-41d4-a716-446655440000",
  "source": "string",
  "timestamp": "2026-05-19T09:30:00+00:00",
  "version": 1,
  "confidence": 0.95,
  "anchor_coordinates": {
    "time": 0.82,
    "space": 0.31,
    "risk": 0.67,
    "rule": 0.71,
    "trust": 0.55,
    "number": 0.12,
    "body": 0.08,
    "emotion": 0.23
  },
  "metrics": {
    "score_reconstruction": 0.963,
    "score_stability": 0.931,
    "score_coherence": 0.967
  }
}
```

### BundleMetrics（評価指標）

```json
{
  "score_reconstruction": 0.963,
  "score_stability": 0.931,
  "score_coherence": 0.967
}
```

**定義：**
- `score_reconstruction`：参照再現率。PoCでK=64時0.963達成（H-401）
- `score_stability`：アンカー安定度。PoCで0.931達成（H-203）
- `score_coherence`：意味整合度。PoCで達成（H-301/302）

---

## エンドポイント詳細

### POST `/bundle/create`

**リクエスト：**
```json
{
  "text": "Confidential documents must not be taken outside the office.",
  "source": "security_policy_v2",
  "confidence": 1.0
}
```

**レスポンス：**
```json
{
  "bundle_id": "550e8400-...",
  "source": "security_policy_v2",
  "timestamp": "2026-05-19T09:30:00+00:00",
  "version": 1,
  "anchor_coordinates": {...},
  "metrics": {...}
}
```

---

### POST `/bundle/search`

**リクエスト：**
```json
{
  "query": "機密書類の取り扱いについて",
  "top_k": 5,
  "threshold": 0.7
}
```

**レスポンス：**
```json
{
  "results": [
    {
      "bundle_id": "...",
      "similarity": 0.921,
      "source": "security_policy_v2"
    }
  ]
}
```

---

### POST `/bundle/compress`

**リクエスト：**
```json
{
  "bundle_id": "550e8400-...",
  "k": 64
}
```

**レスポンス：**
```json
{
  "bundle_id": "550e8400-...",
  "compression_k": 64,
  "memory_reduction_rate": 0.917,
  "score_reconstruction": 0.963
}
```

**注意：** kの推奨値はSB-010参照。内部アルゴリズムは非公開。

---

### POST `/bundle/reconstruct`

**リクエスト：**
```json
{
  "bundle_id": "550e8400-...",
  "context": "optional context text"
}
```

**レスポンス：**
```json
{
  "bundle_id": "550e8400-...",
  "score_reconstruction": 0.963,
  "reconstructed": true
}
```

---

### POST `/bundle/consistency`

**リクエスト：**
```json
{
  "bundle_id_a": "550e8400-...",
  "bundle_id_b": "661f9511-..."
}
```

**レスポンス：**
```json
{
  "score_coherence": 0.967,
  "passed": true,
  "flag": null
}
```

`flag` の値：
- `null`：正常
- `"WARNING"`：要確認（0.4〜0.7）
- `"HALLUCINATION_RISK"`：低整合性（< 0.4）

---

### PUT `/bundle/{id}`（bundle更新）

**リクエスト：**
```json
{
  "new_text": "Apple Vision Pro is a spatial computing headset.",
  "rho": 0.1
}
```

**注意：** `rho`はSB-011参照。推奨値0.1。最大安全値0.15。

**レスポンス：**
```json
{
  "bundle_id": "...",
  "version": 2,
  "drift_score": 0.054,
  "score_stability": 0.946
}
```

---

## エラーコード

| コード | 意味 |
|--------|------|
| 4001 | 無効なデータ形式 |
| 4002 | bundle_idが存在しない |
| 4102 | 次元不一致 |
| 4201 | rhoが範囲外（0.0〜1.0） |
| 4202 | kが範囲外（1〜embedding次元） |
| 5001 | bundle生成失敗 |
| 5002 | 圧縮失敗 |

---

## 実装上の注意

```python
# rhoの検証（必須）
if not 0.0 < rho <= 1.0:
    raise HTTPException(status_code=4201)

# 推奨値チェック（警告として返す）
if rho > 0.15:
    response.headers["X-Warning"] = "rho > 0.15: contamination risk"

# kの推奨値（SB-010）
RECOMMENDED_K = 64
```

---

## Phase 2以降で追加予定

- `POST /bundle/drift` — ドリフト検知
- `POST /bundle/merge` — bundle統合（詳細仕様は非公開）
- `POST /bundle/rotate` — 文脈座標系変換（詳細仕様は非公開）
- `GET /anchor/stability` — アンカー安定性レポート
- WebSocket — リアルタイム整合性チェック
