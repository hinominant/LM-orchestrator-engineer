# Data Protection Protocol

> Claude Code への入力情報の保護。個人情報保護法・J-SOX・IPO審査対応。

---

## 入力禁止情報（絶対原則）

Claude Code（AI）への入力は**第三者提供**に該当する可能性があります。
以下の情報は Claude Code のコンテキストに含めてはいけません。

| カテゴリ | 具体例 | 根拠 |
|---------|-------|------|
| 採用候補者データ | 適性診断スコア、評価シート、面接メモ | 改正個人情報保護法（要配慮個人情報）|
| 本番DB接続情報 | 本番環境の接続文字列、認証情報 | 機密情報保護・データ漏洩リスク |
| 未公開財務数値 | 売上予測、投資家向け資料、取締役会資料 | 内部者取引・IPO規制対応 |
| 顧客個人情報 | 氏名・住所・連絡先を含むCSV/DB | 個人情報保護法 第23条（第三者提供の制限） |
| 従業員情報 | 給与データ、人事評価、健康診断結果 | 要配慮個人情報・従業員プライバシー |

### 代替手段: マスキング・ダミーデータ

```
本物のデータ:
  名前: 田中 太郎, メール: tanaka@example.com, スコア: 78.5

マスク後（Claude Code に渡してよいデータ）:
  名前: [候補者A], メール: [masked], スコア: [数値]
  または
  名前: テスト太郎, メール: test@dummy.local, スコア: 75
```

---

## 技術的実施手段

### 1. .claudeignore（ファイル除外）

`_templates/.claudeignore` を `.claudeignore` としてプロジェクトルートに配置。
Claude Code が `--add-dir` や `@ファイル参照` でもこれらのパスを物理的に読まない。

```bash
# インストール
cp _templates/.claudeignore .claudeignore
```

### 2. tool-risk.js（本番DB接続ブロック）

本番DB接続文字列（`postgresql://user:pass@non-localhost/db`）を Safety Gate でブロック。
`localhost` / `127.0.0.1` の開発DB接続は通過する。

### 3. settings.json deny ルール（機密ファイル読み取りブロック）

```json
{
  "permissions": {
    "deny": [
      "Read(**/candidates*.csv)",
      "Read(**/personal_data/**)",
      "Read(data/production/**)",
      "Read(./.env)",
      "Read(./.env.*)"
    ]
  }
}
```

### 4. Managed Settings（組織ポリシー強制）

MDM/設定管理ツールで `managed-settings.json` を全端末に配布することで、
個人の設定変更によるバイパスを防止する。

---

## J-SOX 操作ログ保存

### 要件

日本版SOX法（金融商品取引法166条等）に基づく財務報告関連記録の**7年保存**。
Claude Code による操作も「ITシステム操作記録」として保存対象となりえる。

### 実装: post-tool-use.js の監査ログ

```json
{
  "timestamp": "2026-03-21T10:00:00.000Z",
  "session_id": "session-abc123",
  "operator": "keiji",
  "project": "my-project",
  "tool": "Edit",
  "input_summary": "src/billing/invoice.ts",
  "success": true
}
```

- `operator`: `process.env.USER`（OS ユーザー名）
- `project`: `path.basename(process.cwd())`（プロジェクト名）
- ログファイル: `.context/tool-log.jsonl`（JSONL形式）

### 7年保存のための運用手順

```bash
# 月次: ログをアーカイブ（S3/GCS等の監査対応ストレージへ）
# 例: AWS S3 に保存（暗号化・バージョニング有効化推奨）
aws s3 cp .context/tool-log.jsonl \
  s3://your-audit-bucket/claude-logs/$(date +%Y/%m)/tool-log.jsonl

# またはログ集約ツール（Splunk, Datadog Logs等）へ転送
```

---

## AIポリシー（全社向けテンプレート）

→ `_templates/AI_USAGE_POLICY.md` を参照

---

## チェックリスト（タスク開始前）

Claude Code で作業を始める前に確認:

- [ ] 扱うデータに個人情報・採用候補者データが含まれていないか
- [ ] 本番DBではなく開発DB/テストDBを使っているか
- [ ] `.env` ファイルに本番のシークレットが入っていないか
- [ ] 作業内容は `/data-guard` で事前チェックしたか

---

## 関連ドキュメント

- `_templates/.claudeignore` — 除外ファイルリスト
- `_templates/managed-settings.json` — 組織ポリシー強制設定
- `_templates/AI_USAGE_POLICY.md` — 全社AIポリシー
- `_common/TOOL_RISK.md` — Hook リスク分類
- `docs/SECURITY_ARCHITECTURE.md` — セキュリティ全体像
