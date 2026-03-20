---
name: data-guard
description: データ保護事前チェック。個人情報・本番データ・機密情報が含まれないかをタスク開始前に確認する
model: haiku
---

# Data Guard — データ保護事前チェック

## 目的

Claude Code のコンテキストに**入力禁止情報**が含まれないことをタスク開始前に確認する。
改正個人情報保護法・J-SOX・IPO審査対応のための DLP（Data Loss Prevention）チェック。

## 実行タイミング

- 本番データ・CSVエクスポート・DBダンプを扱う作業の前
- 採用・HR関連のコードを作業する前
- 財務・請求・決済システムを触る前
- 外部APIのデータを処理するタスクの前

---

## チェックリスト

### Step 1: 入力予定データの確認

以下の情報が作業で扱うデータに含まれていないか確認する:

```
□ 採用候補者の個人情報（氏名・適性診断スコア・評価・連絡先）
□ 顧客の個人情報（氏名・住所・電話番号・メールアドレス）
□ 従業員情報（給与・評価・健康診断）
□ 本番DB接続文字列（非localhostのDB URL）
□ 未公開財務数値（売上予測・取締役会資料）
□ APIキー・パスワード・秘密鍵（.env含む）
```

**いずれかに該当する場合**: ダミーデータ・マスキングデータに置き換えてから作業を進める。

### Step 2: ファイルパスの確認

作業対象ファイルが `.claudeignore` の除外パターンに含まれていないか確認する:

```bash
# .claudeignore の内容を確認
cat .claudeignore 2>/dev/null || echo ".claudeignore が存在しません。_templates/.claudeignore をコピーしてください"
```

**除外対象パスへのアクセスが必要な場合**: マスクしたサンプルデータを使うか、タスクを分解して機密部分を切り離す。

### Step 3: 接続先環境の確認

```bash
# .env の DATABASE_URL が本番ではないことを確認（値は表示しない）
if [ -f .env ]; then
  if grep -q "DATABASE_URL" .env; then
    grep "DATABASE_URL" .env | grep -q "localhost\|127\.0\.0\.1" \
      && echo "✅ 開発DB（localhost）" \
      || echo "⚠️ 本番DBの可能性があります。接続先を確認してください"
  fi
fi
```

### Step 4: 作業前宣言

以下のいずれかを確認して作業を開始する:

```
A) 扱うデータに個人情報・機密情報は含まれていない → 作業開始 OK
B) ダミー/マスキングデータに置き換えた → 作業開始 OK
C) 該当データが含まれる → タスクを分解 or データ準備から始める
```

---

## 判断フロー

```
データ確認
    ↓
個人情報・機密情報が含まれる？
    ├── No  → ✅ 作業開始
    └── Yes → マスキング可能？
                  ├── Yes → マスク後に作業開始
                  └── No  → タスク分解・別アプローチを提案
```

---

## マスキング例

### 採用候補者データ

```csv
# 元データ（使用禁止）
名前,メール,適性診断スコア
田中太郎,tanaka@gmail.com,78.5
山田花子,yamada@example.com,85.0

# マスク後（使用可）
名前,メール,適性診断スコア
候補者A,[masked],78.5
候補者B,[masked],85.0
```

### DB接続文字列

```bash
# 禁止: 本番DB
DATABASE_URL=postgresql://admin:prod_password@prod-db.company.com:5432/myapp

# OK: 開発DB
DATABASE_URL=postgresql://user:password@localhost:5432/myapp_dev
```

---

## 関連ドキュメント

- `_common/DATA_PROTECTION.md` — データ保護プロトコル全文
- `_templates/.claudeignore` — 除外ファイルリスト
- `_templates/AI_USAGE_POLICY.md` — 組織AIポリシー
