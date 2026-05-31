# チャット引き継ぎ手順書

> 最終更新：2026-05-31
> 対象：斉藤（プロジェクトオーナー）

---

## 新しいチャットを開始するタイミング

| 状況 | 対応 |
|------|------|
| コンテキスト 40〜50% 到達 | `/compact` 実行 → それでも重い場合は新チャット |
| 新しいフェーズ・論文に移行する時 | 新チャット推奨 |
| 応答が遅くなってきた・ループした時 | 新チャット |
| タスクが完全に変わる時（例：実装 → 論文執筆） | 新チャット |

---

## 手順

### 1. 現在のセッションでログを作成・プッシュ

```
ログ作成してgit pushして
```

→ Claude Code がセッション内容を `logs/log_YYYYMMDD.md` に記録・プッシュ

### 2. 新しいチャットを開く

- Claude Code（CLI）：`claude` コマンドで起動（同じディレクトリから）
- Claude.ai チャット：新規会話を開く

### 3. 冒頭に必須ファイルの内容を貼る

以下のファイルを順番に読み込ませる（Claude.ai チャットの場合は内容をコピペ）：

1. `docs/meta/CLAUDE.md`
2. `docs/ops/research_strategy.md`
3. `logs/log_YYYYMMDD.md`（直近のログ）

### 4. 初回プロンプトを貼る

```
現在地を把握して。
今日やること：[タスク内容を1行で記載]
```

---

## 必須ファイル一覧

| 優先度 | ファイル | 内容 |
|--------|---------|------|
| **最高** | `docs/meta/CLAUDE.md` | Claude Code統合ガイド・サーバー情報・ログルール・OSG方針 |
| **最高** | `docs/ops/research_strategy.md` | 論文パイプライン・Zenodo DOI・先行公開記録 |
| **最高** | `logs/log_YYYYMMDD.md`（直近） | 完了事項・仮説ステータス・次回アクション |
| 高 | `docs/ops/hypotheses.md` | 仮説ステータス一覧（CONFIRMED/HYPOTHESIS/REJECTED） |
| 高 | `docs/ops/roadmap.md` | Phase/Stage全体地図 |
| 推奨 | `docs/meta/decisions.md` | 設計判断ログ ADR（なぜそう決めたか） |
| 推奨 | `docs/core/thesis.md` | コア命題・主張の骨格 |
| 推奨 | `docs/impl/spec.md` | 確定仕様（唯一の真実） |

---

## 現在のプロジェクト状態（2026-05-31 時点）

### 論文公開状況

| 論文 | DOI（全版） | 公開日 | 状態 |
|------|------------|--------|------|
| Paper 0 理論 | 10.5281/zenodo.20417222 | 2026-02-24 | ✅ 公開済み |
| Paper 1 PoC実証 | 10.5281/zenodo.20417714 | 2026-05-27 | ✅ 公開済み（SSRN却下） |
| Paper 2 アンカー設計 | 10.5281/zenodo.20434834 | 2026-05-28 | ✅ 公開済み |
| Paper 2.5 多言語汎化 | 取得済み | 2026-05-31 | ✅ 公開済み |

### 4軸フレームワーク確定状態

| 軸 | 状態 | 証拠 |
|----|------|------|
| A軸（モデル間） | ✅ 完了 | Paper 2 region anchor 0.704、大規模化不要（理論確定） |
| B軸（時系列） | △ 設計原則確立 | PoC②：drift 38.6%削減・整合性0.931、各モデル空間内で独立管理が正 |
| C軸（文化横断） | ✅ 完了 | Paper 2.5（0.995）＋Paper 3大規模（0.945、1012文×9言語） |
| D軸（摂動耐性） | ✅ 完了 | Paper 1・2：1.000 |

### 次回やること

- [ ] Paper 3 論文化（C軸大規模検証）
- [ ] `docs/meta/decisions.md` に ADR追加（A軸理論確定・B軸設計原則・人間介入の有効性）
- [ ] baseline 比較の検討（既存多言語手法との比較）
- [ ] bundle 操作本体の実験設計（意味の合成・分解・部分編集）

---

## Claude Code 起動方法

### 作業ディレクトリ

```
C:\Users\satou\Semantic-Bundle-AI
```

### CLAUDE.md の場所

```
C:\Users\satou\Semantic-Bundle-AI\docs\meta\CLAUDE.md
```

Claude Code 起動時に自動で読まれる `CLAUDE.md` はプロジェクトルートに置く必要があるが、
現在は `docs/meta/CLAUDE.md` に置いている（非公開管理のため）。
新セッション開始時は最初に手動で `docs/meta/CLAUDE.md` を読み込ませること。

### よく使うコマンド

```bash
# ログ作成 & push
ログ作成してgit pushして

# HP更新
docs/meta/CLAUDE.md のHP編集手順を参照

# LaTeXコンパイル
cd papers/draft
pdflatex main_xxx.tex && bibtex main_xxx && pdflatex main_xxx.tex && pdflatex main_xxx.tex
```

---

## 引き継ぎ文テンプレート（ログから毎回コピー）

各ログの末尾「次回Claudeへの引き継ぎ」欄をそのまま新チャットの冒頭に貼ればOK。

**例（2026-05-31 時点）：**
> 4軸フレームワーク完成。C軸（文化横断）は Paper 2.5（小規模0.995）＋Paper 3大規模検証（1012文PASS、0.945）で二重証明済み。A軸大規模化は理論的に不要と確定（各モデル内の安定参照系がB解釈・正しい立場）。次は Paper 3 論文化と decisions.md ADR追加。

---

## ログの場所と命名規則

```
logs/log_YYYYMMDD.md          # 当日1回目
logs/log_YYYYMMDD_2.md        # 当日2回目
logs/log_YYYYMMDD_afternoon.md # 午後セッション（旧命名）
```

※ `logs/` は `.gitignore` で非公開管理。コミット時は `git add -f` を使用。
