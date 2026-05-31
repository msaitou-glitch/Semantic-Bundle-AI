# CLAUDE.md — Semantic Bundle AI 実装ガイド

> このファイルはClaude Codeが最初に読む統合ガイドである。
> 実装に必要な「判断基準・制約・参照先」をここに集約する。
> 思想・数理・ビジネスはここに書かない。

---

## サーバー接続情報

### HP（ontheshouldersofgiants.jp）

```
プロトコル : SSH / SFTP
ホスト     : sv16586.xserver.jp
ポート     : 10022
ユーザー   : xs814768
SSH鍵      : C:\Users\satou\.ssh\xserver_claude
公開HTML   : /home/xs814768/ontheshouldersofgiants.jp/public_html/
FTPユーザー: claude-edit@ontheshouldersofgiants.jp
FTPパスワード: VjCaxd6uZnJLY85
```

HP編集の標準手順：
```bash
# 1. ファイル取得
scp -i ~/.ssh/xserver_claude -P 10022 \
    xs814768@sv16586.xserver.jp:/home/xs814768/ontheshouldersofgiants.jp/public_html/index.html \
    ./index_hp.html

# 2. 編集（Claude Codeが行う）

# 3. アップロード
scp -i ~/.ssh/xserver_claude -P 10022 \
    ./index_hp.html \
    xs814768@sv16586.xserver.jp:/home/xs814768/ontheshouldersofgiants.jp/public_html/index.html
```

接続確認コマンド：
```bash
ssh -i ~/.ssh/xserver_claude -p 10022 -o StrictHostKeyChecking=no \
    xs814768@sv16586.xserver.jp "ls /home/xs814768/ontheshouldersofgiants.jp/public_html/"
```

## ログ作成ルール

作業区切り時に以下を実行する。「ログ作成して」と言われたら自動で全部やる。

### 手順

```bash
# 1. ログファイル作成
# 場所：C:\Users\satou\semantic-bundle-ai\logs\
# 命名：log_YYYYMMDD.md（同日2回目以降は log_YYYYMMDD_2.md）

# 2. git commit & push
git add logs/log_YYYYMMDD.md
git commit -m "Add session log YYYYMMDD"
git push origin main
```

### ログの記載内容

```markdown
# セッションログ YYYY-MM-DD

## 前回からの継続
前回ログ：logs\log_YYYYMMDD.md

## 本日完了したこと
（ファイル名・数値・発見を具体的に）

## 現在の仮説ステータス
（変更があったもののみ）

## ファイル一覧（変更・追加したもの）

## 次回やること
- [ ] 優先度高
- [ ] 優先度中

## 次回Claudeへの引き継ぎ
（貼り付けるだけで再開できる一文）
```

### 注意
- 数値は必ず記載する（「改善した」ではなく「38.6%削減」）
- 仮説ステータス変更は必ず記録する
- git pushまで自動でやる（手動確認不要）

---

Semantic Bundle AI（意味束AI）は、LLMが扱う意味表現を**安定アンカーベースの座標系でbundle化**し、長期的に管理可能な意味構造OSを作るプロジェクトである。

LLMを置き換えない。LLMの上に乗る意味管理レイヤーを作る。

---

## 最重要制約（実装前に必ず確認）

```
1. LLMを置き換えない設計にする
2. RAGの代替ではなく上位レイヤーとして設計する
3. 確定仕様と仮説を混在させない（spec.md vs hypotheses.md）
4. bundle IDは全システムで一意
5. アンカー集合は変更頻度が低い設計にする
6. 意味の保存はテキスト全文ではなくbundle構造
```

---

## ファイル参照マップ

```
実装する前に読むファイル：
  impl/spec.md          ← 確定仕様（唯一の真実）
  impl/architecture.md  ← システム構造
  impl/db.md            ← DBスキーマ
  core/glossary.md      ← 用語定義（迷ったらここ）

APIを実装するとき：
  impl/api.md

何かを変更するとき：
  meta/decisions.md     ← 過去の判断ログを確認してから変更

仮説をコードにするとき：
  ops/hypotheses.md     ← 仮説ステータスを確認してから実装
```

---

## 現在のPhase

**Phase 0 — 概念PoC**

目標：
- 安定座標変換が機能することを確認する
- 同義文群がアンカー座標上で収束することを確認する
- モデル間で安定座標が保たれることを確認する

完了条件：
- `impl/spec.md` の「Phase 0 完了条件」を全て満たす

---

## 技術スタック（Phase 0〜1）

```
言語：       Python 3.11+
embedding：  sentence-transformers
数値計算：   numpy, scikit-learn
可視化：     matplotlib
DB：         PostgreSQL + pgvector（Phase 1から）
API：        FastAPI（Phase 1から）
```

---

## ディレクトリ構造（実装コード側）

```
semantic-bundle-ai/
├── docs/               ← ドキュメント（このファイルはここ）
├── src/
│   ├── core/
│   │   ├── bundle.py       ← SemanticBundle クラス
│   │   ├── anchor.py       ← StableAnchor クラス
│   │   └── coordinate.py   ← 安定座標変換
│   ├── storage/
│   │   ├── vector_store.py
│   │   ├── relational.py
│   │   └── graph_store.py
│   ├── api/
│   │   └── routes.py
│   └── loops/
│       ├── consistency.py  ← 整合性チェック
│       └── drift.py        ← ドリフト検知
├── tests/
├── notebooks/          ← PoC実験
└── scripts/
```

---

## コーディング規則

```python
# bundle IDはUUID v4
bundle_id: str  # "550e8400-e29b-41d4-a716-446655440000"

# timestampはISO 8601 UTC
timestamp: str  # "2025-01-15T09:30:00Z"

# embeddingはnumpy array
embedding: np.ndarray  # shape: (d,)

# anchor_coordinatesはdict
anchor_coordinates: dict[str, float]  # {"time": 0.82, "space": 0.31, ...}

# confidence scoreは0.0〜1.0
confidence: float  # 0.0 to 1.0
```

---

## 絶対にやってはいけないこと

```
- テキスト全文をメモリとして蓄積する（bundle構造で保持する）
- 仮説をspecに書く（hypotheses.mdに書く）
- bundle IDを手動で割り当てる（UUIDを使う）
- アンカー集合をコード内にハードコードする（設定ファイルから読む）
- 非公開仕様（時間ダイナミクス詳細・bundle merge/split正式仕様）を実装する
  → ops/hypotheses.md でステータスを確認してから着手する
```

---

## 用語が不明な場合

→ `core/glossary.md` を参照する。そこに無い用語を使う場合はglossaryに追加してから使う。

---

## 変更を加える場合のチェックリスト

- [ ] `core/glossary.md` の用語と整合しているか
- [ ] `impl/spec.md` の確定仕様に違反していないか
- [ ] `ops/hypotheses.md` で仮説ステータスを確認したか
- [ ] `meta/decisions.md` に変更理由を記録したか
- [ ] テストを書いたか

---

## Git 公開・非公開方針

### 非公開（.gitignore に登録）

| パターン | 理由 |
|---|---|
| `logs/` | 作業ログは社外秘 |
| `private-specs/` | 非公開仕様 |
| `docs/meta/CLAUDE.md` | このファイル自体（接続情報含む） |
| `papers/` | 論文原稿・投稿前資料 |
| `*.pub` | SSH公開鍵 |
| `xserver_claude` | SSH秘密鍵 |
| `index_hp.html` | HPローカル作業用コピー |
| `*.bat` | ローカル作業スクリプト |
| `*.png`（notebooks/ 以外） | 論文図・中間出力 |
| `main*.tex / .pdf / .aux / .bbl / .blg / .log / .out` | LaTeX原稿・ビルド成果物 |

### 公開（GitHubに公開）

| パターン | 内容 |
|---|---|
| `notebooks/*.ipynb` | PoC実験ノートブック |
| `notebooks/*.png` | PoC実験の可視化結果 |
| `requirements.txt` | 依存パッケージ |
| `README.md` | プロジェクト概要 |
| `docs/core/` | 用語・理論ドキュメント |
| `docs/impl/` | 実装仕様ドキュメント |
| `docs/ops/` | 運用・仮説管理ドキュメント |
| `docs/business/` | ビジネス文書 |
| `docs/README.md` | docs インデックス |

---
## OSG統合本部 共通方針（2026-05-23追記）

### 著者情報
- 著者名：M. Saitou
- ORCID iD：0009-0009-0865-8193
- ORCID URL：https://orcid.org/0009-0009-0865-8193
- 所属：ON THE SHOULDERS OF GIANTS PROJECT
- メール：m.saitou@ontheshouldersofgiants.jp

### ブランド理念（Section 7）
- 社名：On the Shoulders of Giants / 略称：gAIants
- 理念：世界の構造を再定義する
- タグライン："On the shoulders of gAIants"
- 全ての成果物はgAIantsブランドと整合させること
- 詳細：C:\Users\satou\osg-headquarters\osg-master-strategy.md

### Claude Design活用方針（Section 8）
- UIデザイン・図解・資料が必要な場面ではClaude Designを先に使う
- Claude Design → ハンドオフ → Claude Code の順で進める
- このPJでのClaude Design活用場面：
  C:\Users\satou\osg-headquarters\claude-design-instructions.md を参照
- デザインシステム設定済み（gAIantsブランド）：claude.ai/design

### セッション管理ルール（Section 6）
- 作業区切りごとにセッションログを自動生成
- 保存先：docs/logs/YYYY-MM-DD-topic.md（SBAは当面 logs/ を併用）
- フォーマット：完了事項・ファイル一覧・設計判断・次回やること・引き継ぎ文
- GitコミットはTask完了ごとに実施（1作業1コミット）
- コミットメッセージ：feat/fix/docs/refactor/chore
- セッション終了時に必ず出力：完了事項・コミット履歴・project-status.md更新差分・次回引き継ぎ文

### 使用可能なMCPツール一覧
Claudeがこれらのツールを持っていても、明示しなければ使われない。
必ずMCPツールの活用を検討してから手動作業に入ること。

#### 導入済み
- Gmail MCP：メール読み取り・下書き・送信
- Google Drive MCP：ファイル読み取り・検索
- Google Calendar MCP：スケジュール確認

#### 導入予定（Step1・最優先）
- filesystem：ローカルフォルダへの直接アクセス
- git：GitHubリポジトリ操作
- memory：セッションをまたいだ記憶
- sequential thinking：複雑な設計判断の推論強化

#### 導入予定（Step2）
- Supabase MCP：DB直接操作（イベント通知・Re:Sync）
- GitHub MCP：PR作成・Issue管理・コードレビュー
- Vercel MCP：ログ・環境変数・デプロイ状態の直接確認

#### 導入予定（Step4）
- Firecrawl：高度なWebスクレイピング
- ElevenLabs：音声生成（文明論YouTube化）
- Neo4j：グラフDB（意味束AI）
- Qdrant：ベクトル検索（意味束AI）

#### 重要ルール
- 手動確認・手動コピペをする前に、まずMCPツールで対応できないか確認すること
- 特にVercel・Supabase・GitHubの操作は必ずMCP経由を優先すること
- MCPツールが使えない場合のみ手動作業に切り替えること

### 三者役割分担（Section 9）

| 役割 | 担当 | やること |
|------|------|---------|
| 設計・判断・指示 | 専用チャット（Claude.ai） | 何を作るか・どう設計するかを決める |
| 実装・デプロイ | Claude Code（あなた） | 決まったものを作る |
| UIデザイン | Claude Design | 作る前に見せる |

**正しいフロー**
```
専用チャット（設計決定）
→ Claude Design（UIが必要な場合・プロトタイプ）
→ Claude Code（実装・デプロイ）
→ 専用チャット（結果確認・次の判断）
```

**Claude Code（あなた）がやってはいけないこと**
- 設計方針の大きな変更を自己判断で実装する → 専用チャットに確認を取ること
- UIを自分で設計・実装する（Claude Designを使わずに） → Claude Designでプロトタイプを先に作ること
- 統合本部が決める事項を自己判断する → 専用チャット経由で統合本部に持ち帰ること

**このCLAUDE.mdを読んだClaude Codeへ**
あなたはClaude Codeです。「実装・デプロイ・報告」が役割です。
設計変更は専用チャットに確認を取り、UIはClaude Designを先に使うこと。
「了解」とだけ返し、この役割分担を守って作業してください。

### コンテキスト管理ルール（Section 10）

コンテキスト腐敗（Context Rot）は50%超で劣化ゾーン。100%まで待たないこと。

| 状況 | 対応 |
|------|------|
| タスクが変わった | /clear（新規セッション） |
| アプローチが失敗した | /rewind → 学んだことを含めて再指示 |
| セッションが40〜50%に到達 | /compact（「次にやること」を添えて手動実行） |
| 大量の中間出力が出る作業 | サブエージェントに委任・結論だけ返す |

**OSG運用ルール**
- 1セッション1タスク
- セッション開始時に必ずCLAUDE.mdを読む（安定コンテキスト確保）
- 作業区切りごとにセッションログをファイル保存（会話が揮発してもファイルで復元）
- /compactは40〜50%で手動実行（自動compactを待たない）

---

## 公開済み論文（Zenodo DOI）

| 論文 | DOI（全版） | DOI（v1） | DOI（v2） |
|------|------------|----------|----------|
| Paper 0 理論 | 10.5281/zenodo.20417222 | 10.5281/zenodo.20417223 | — |
| Paper 1 PoC実証 | 10.5281/zenodo.20417714 | 10.5281/zenodo.20417715 | — |
| Paper 2 アンカー設計 | 10.5281/zenodo.20434834 | 10.5281/zenodo.20434835 | 10.5281/zenodo.20476703（2026-05-31公開・metric conflation 修正版） |
