# luvira-outreach-os データモデル v1

## 概要
このデータモデル v1 は、luvira-outreach-os のMVP v1を実装するための最小構成である。
目的は、CSV取込、チャネル分岐、実行ログ、KPI集計、標準出力を一貫した内部スキーマで扱えるようにすることにある。

設計方針は以下の通り。
- 入力原本は残す
- 実行用に正規化する
- ログはイベント単位で持つ
- チャネル結果は最新状態を別に持つ
- KPIはログや結果から後段で集計する
- 出力先依存の列名や形式は内部スキーマから後段変換する

---

## 1. source_lists
CSVアップロード単位を管理する親エンティティ。

| フィールド名 | 型 | 必須 | 説明 |
|---|---|---:|---|
| source_list_id | string | Yes | 取込リストID。UUID想定。 |
| customer_id | string | Yes | 顧客ID。 |
| list_name | string | Yes | リスト表示名。 |
| source_type | string | Yes | `csv_upload` / `webhook_import` など。 |
| uploaded_at | datetime | Yes | 取込日時。 |
| uploaded_by | string | No | 実行者または担当者。 |
| raw_schema_name | string | No | 元CSVフォーマット名。 |
| row_count | number | No | 元CSVの行数。 |
| notes | string | No | 補足メモ。 |

### 主な用途
- どのCSVが元データかを追跡する
- KPIをリスト単位で比較する
- 取込元フォーマット差分を管理する

---

## 2. targets
営業対象として正規化された1社1レコード。

| フィールド名 | 型 | 必須 | 説明 |
|---|---|---:|---|
| target_id | string | Yes | 対象レコードID。UUID想定。 |
| source_list_id | string | Yes | 元リストID。 |
| source_row_id | string | No | 元CSV行IDまたは行番号。 |
| company_name | string | Yes | 会社名。 |
| website_url | string | No | 公式URL。 |
| contact_email | string | No | 営業対象メールアドレス。 |
| form_url | string | No | 問い合わせフォームURL。 |
| phone | string | No | 電話番号。 |
| address | string | No | 住所。 |
| industry | string | No | 業種。 |
| normalized_status | string | Yes | `active` / `invalid` / `excluded` など。 |
| channel_eligibility_email | boolean | Yes | メール営業対象可否。 |
| channel_eligibility_form | boolean | Yes | フォーム営業対象可否。 |
| exclusion_reason | string | No | 除外理由。 |
| created_at | datetime | Yes | 作成日時。 |
| updated_at | datetime | Yes | 更新日時。 |

### 主な用途
- CSV列名の違いを吸収した標準営業対象データ
- チャネル分岐の判定元
- 実行単位とログ単位の基準レコード

---

## 3. executions
1回の実行単位を管理するエンティティ。

| フィールド名 | 型 | 必須 | 説明 |
|---|---|---:|---|
| execution_id | string | Yes | 実行ID。UUID想定。 |
| customer_id | string | Yes | 顧客ID。 |
| source_list_id | string | Yes | 対象リストID。 |
| run_mode | string | Yes | `email` / `form` / `both`。 |
| trigger_type | string | Yes | `manual` / `schedule` / `webhook`。 |
| settings_version | string | No | 実行時設定バージョン。 |
| started_at | datetime | Yes | 実行開始日時。 |
| finished_at | datetime | No | 実行終了日時。 |
| status | string | Yes | `running` / `completed` / `failed` / `partial_failed`。 |
| total_targets | number | No | 対象件数。 |
| operator | string | No | 実行者。 |
| notes | string | No | 補足。 |

### 主な用途
- いつどの設定で実行したかを管理する
- エラー追跡とKPI集計の親単位にする
- Webhook / 手動 / 定期実行の差を吸収する

---

## 4. activity_logs
実行中に発生したイベントログ。標準ログスキーマの中心。

| フィールド名 | 型 | 必須 | 説明 |
|---|---|---:|---|
| log_id | string | Yes | ログID。UUID想定。 |
| execution_id | string | Yes | 実行ID。 |
| target_id | string | Yes | 対象ID。 |
| channel | string | Yes | `email` / `form` / `system`。 |
| event_type | string | Yes | `validated` / `queued` / `attempted` / `sent` / `failed` / `delivered` など。 |
| status | string | Yes | イベント時点の状態。 |
| reason_code | string | No | 失敗や除外のコード。 |
| reason_detail | string | No | 失敗詳細。 |
| occurred_at | datetime | Yes | 発生日時。 |
| operator | string | No | `system` / `human`。 |
| payload_summary | string | No | ペイロード要約。 |
| output_adapter | string | No | `sheets` / `csv` / `webhook` / `db_map` など。 |

### 主な用途
- KPI計算の一次ソース
- デバッグ
- 顧客向けの実行証跡
- 出力アダプタとは独立した内部イベント管理

---

## 5. channel_results
チャネルごとの最新結果サマリ。イベントログとは別に保持する。

| フィールド名 | 型 | 必須 | 説明 |
|---|---|---:|---|
| result_id | string | Yes | 結果ID。UUID想定。 |
| execution_id | string | Yes | 実行ID。 |
| target_id | string | Yes | 対象ID。 |
| channel | string | Yes | `email` / `form`。 |
| attempted | boolean | Yes | 実行対象になったか。 |
| sent | boolean | No | 送信したか。 |
| delivered | boolean | No | メール配信成功またはフォーム送信確認。 |
| replied | boolean | No | 返信があったか。 |
| positive_reply | boolean | No | 前向き返信か。 |
| form_submit_confirmed | boolean | No | フォーム送信確認済みか。 |
| last_status | string | Yes | 最新状態。 |
| last_reason_code | string | No | 最新理由コード。 |
| last_updated_at | datetime | Yes | 最終更新日時。 |

### 主な用途
- 対象ごとの最終状態を早く参照する
- ダッシュボードや一覧表示に使う
- ログ全件を読まずに状況確認できるようにする

---

## 6. kpi_snapshots
月次または集計単位で保存するKPIサマリ。

| フィールド名 | 型 | 必須 | 説明 |
|---|---|---:|---|
| snapshot_id | string | Yes | スナップショットID。 |
| customer_id | string | Yes | 顧客ID。 |
| period_month | string | Yes | `YYYY-MM`。 |
| source_list_id | string | No | リスト単位で切る場合のID。 |
| total_targets | number | Yes | 総対象件数。 |
| email_coverage_rate | number | No | メール保有率。 |
| email_success_rate | number | No | メール送信成功率または配信成功率。 |
| form_coverage_rate | number | No | フォーム保有率。 |
| form_success_rate | number | No | フォーム送信成功率。 |
| list_fit_score | number | No | リスト適合スコア。 |
| created_at | datetime | Yes | 作成日時。 |

### 主な用途
- 月次レポート
- 顧客向け定点比較
- リスト品質比較
- フォーム営業向き / メール営業向きの判断材料

---

## キー設計
最低限固定するキーは以下の3つ。

- `source_list_id`：CSV単位を識別する親キー
- `target_id`：企業単位を識別する主キー
- `execution_id`：実行単位を識別する親キー

この3つを軸に、ログ・結果・KPIをすべて関連づける。

---

## 推奨リレーション
- `source_lists` 1 : N `targets`
- `source_lists` 1 : N `executions`
- `executions` 1 : N `activity_logs`
- `executions` 1 : N `channel_results`
- `targets` 1 : N `activity_logs`
- `targets` 1 : N `channel_results`

---

## n8n 内部データ形の考え方
n8nではノード間データは配列アイテムとして流れ、各アイテムは通常 `json` キーを持つオブジェクトで表現される。
そのため、各sub-workflowの入出力は以下のようにそろえると扱いやすい。

### csv-normalizer 出力例
```json
[
  {
    "json": {
      "target_id": "t_001",
      "source_list_id": "sl_001",
      "company_name": "Example Inc.",
      "website_url": "https://example.com",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact",
      "channel_eligibility_email": true,
      "channel_eligibility_form": true
    }
  }
]
```

### activity-logger 出力例
```json
[
  {
    "json": {
      "log_id": "log_001",
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "event_type": "sent",
      "status": "success",
      "occurred_at": "2026-06-30T10:00:00+09:00"
    }
  }
]
```

---

## Google Sheets 保存方針
標準出力先がGoogle Sheetsの場合、用途ごとに操作を分ける。

- `activity_logs`：Append Row
- `channel_results`：Update Row または Append or Update Row
- `kpi_snapshots`：Append or Update Row

### 運用意図
- イベントログは追記中心
- 最新状態や月次集計はキー一致で更新
- `Append or Update Row` を使う場合は、マッチ列を明示する

---

## MVP v1 時点での注意
- 顧客独自DB向けの列マッピングはこの内部スキーマの後段で吸収する
- まずは内部スキーマを固定し、出力アダプタで形式差分を処理する
- KPIはGoogle Sheetsの列構造ではなく、内部イベントと結果データから計算する
