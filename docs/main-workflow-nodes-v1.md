# main-ingest-and-run 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の親ワークフロー `main-ingest-and-run` を n8n 上で最初に実装するための実ノード構成案である。

MVP v1 では、CSV主導の実行を前提にしつつ、Webhook入口にも将来的に対応できるように構成する。
ただし初期実装では、Manual Trigger + Set ノードによるダミーデータ投入で end-to-end を確認しながら進める。

---

## 実装方針

- 最初は Manual Trigger で動く形を作る
- 入口データは Set ノードでダミー投入できるようにする
- parent workflow は orchestration に徹する
- 実処理は sub-workflow 側に寄せる
- logging と output は分離する
- エラー処理は error-handler workflow に委譲する
- main workflow では validation error を早期に止める

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Manual Trigger | Manual Trigger | 手動テスト起動 |
| 2 | Set Input Payload | Set | テスト用入力データ生成 |
| 3 | Validate Entry Payload | IF | 必須入力チェック |
| 4 | Stop Invalid Input | Stop And Error | 必須入力不足時に停止 |
| 5 | Set Execution Context | Set | execution_id 等の実行文脈生成 |
| 6 | Call sub-csv-normalizer | Execute Workflow | CSV正規化 sub-workflow 呼び出し |
| 7 | Call sub-target-validator | Execute Workflow | 対象判定 sub-workflow 呼び出し |
| 8 | Call sub-channel-router | Execute Workflow | チャネル分岐 sub-workflow 呼び出し |
| 9 | Has Email Targets? | IF | email 対象有無判定 |
| 10 | Call sub-email-outreach | Execute Workflow | メール営業 sub-workflow 呼び出し |
| 11 | Has Form Targets? | IF | form 対象有無判定 |
| 12 | Call sub-form-outreach | Execute Workflow | フォーム営業 sub-workflow 呼び出し |
| 13 | Merge Channel Results | Merge | email / form 結果統合 |
| 14 | Call sub-channel-result-updater | Execute Workflow | 最新状態更新 |
| 15 | Call sub-kpi-aggregator | Execute Workflow | KPI集計 |
| 16 | Call sub-log-output-router | Execute Workflow | 出力先ルーティング |
| 17 | Set Final Execution Status | Set | 実行完了データ生成 |
| 18 | Return Summary | No Op / last node | 最終出力 |

---

## 初期入力ノード設計

### 1. Manual Trigger
最初の実装では Manual Trigger を使用する。
本ノードはエディタ上で `Execute Workflow` を押したときだけ実行される。

#### 用途
- 実装初期のテスト
- sub-workflow 接続確認
- ダミーデータ検証

---

### 2. Set Input Payload
Manual Trigger の直後に置き、テスト用 payload を作る。

#### 設定例
```json
{
  "customer_id": "cust_001",
  "source_type": "csv_upload",
  "run_mode": "both",
  "source_list_name": "tokyo_saas_list_2026_06",
  "settings_version": "v1",
  "raw_rows": [
    {
      "company": "Example Inc.",
      "email": "info@example.com",
      "website": "https://example.com",
      "form": "https://example.com/contact"
    },
    {
      "company": "Demo LLC",
      "email": "",
      "website": "https://demo.jp",
      "form": "https://demo.jp/contact"
    }
  ]
}
```

#### ポイント
- raw_rows は配列のまま渡す
- 将来は CSV Parse 結果や Webhook payload に置き換える
- まずは1〜3件程度で動作確認する

---

## 入力検証ノード

### 3. Validate Entry Payload
IF ノードで以下をチェックする。

#### 判定条件
- `customer_id` が空でない
- `run_mode` が `email` / `form` / `both` のいずれか
- `raw_rows` が配列で1件以上

#### true
- 次へ進む

#### false
- Stop Invalid Input へ進む

---

### 4. Stop Invalid Input
Stop And Error ノードで明示停止する。

#### エラーメッセージ例
```text
Invalid entry payload: customer_id, run_mode, or raw_rows is missing or invalid.
```

#### 意図
- validation error を後段へ流さない
- error-handler workflow で検知しやすくする
- 実装中の誤入力を早く発見する

---

## 実行文脈生成

### 5. Set Execution Context
ここで親 workflow 全体で引き回す共通値を作る。

#### 作る項目
```json
{
  "execution_id": "ex_{{$now}}",
  "started_at": "{{$now}}",
  "status": "running",
  "trigger_type": "manual",
  "customer_id": "={{$json.customer_id}}",
  "source_type": "={{$json.source_type}}",
  "run_mode": "={{$json.run_mode}}",
  "source_list_name": "={{$json.source_list_name}}",
  "settings_version": "={{$json.settings_version}}",
  "raw_rows": "={{$json.raw_rows}}"
}
```

#### ポイント
- 実装時は execution_id 生成ルールを統一する
- `customer_id`, `run_mode`, `raw_rows` は以降ずっと保持する

---

## sub-workflow 呼び出し

### 6. Call sub-csv-normalizer
Execute Workflow ノードで `sub-csv-normalizer` を呼び出す。

#### 渡す入力
```json
{
  "source_list_id": "sl_{{$now}}",
  "raw_rows": "={{$json.raw_rows}}"
}
```

#### 期待出力
- `targets`

---

### 7. Call sub-target-validator
Execute Workflow ノードで `sub-target-validator` を呼び出す。

#### 渡す入力
```json
{
  "targets": "={{$json.targets}}"
}
```

#### 期待出力
- `targets` with eligibility fields

---

### 8. Call sub-channel-router
Execute Workflow ノードで `sub-channel-router` を呼び出す。

#### 渡す入力
```json
{
  "execution_id": "={{$json.execution_id}}",
  "run_mode": "={{$json.run_mode}}",
  "targets": "={{$json.targets}}"
}
```

#### 期待出力
- `email_targets`
- `form_targets`
- `excluded_targets`

---

## チャネル別分岐

### 9. Has Email Targets?
IF ノード。

#### 条件
- `email_targets.length > 0`

#### true
- `Call sub-email-outreach` へ

#### false
- email branch をスキップ

---

### 10. Call sub-email-outreach
Execute Workflow ノード。

#### 渡す入力
```json
{
  "execution_id": "={{$json.execution_id}}",
  "email_targets": "={{$json.email_targets}}"
}
```

#### 期待出力
- `activity_logs`
- `channel_results`

---

### 11. Has Form Targets?
IF ノード。

#### 条件
- `form_targets.length > 0`

#### true
- `Call sub-form-outreach` へ

#### false
- form branch をスキップ

---

### 12. Call sub-form-outreach
Execute Workflow ノード。

#### 渡す入力
```json
{
  "execution_id": "={{$json.execution_id}}",
  "form_targets": "={{$json.form_targets}}"
}
```

#### 期待出力
- `activity_logs`
- `channel_results`

---

## 結果統合

### 13. Merge Channel Results
Merge ノードで email branch と form branch の出力を統合する。

#### 統合対象
- activity_logs
- channel_results

#### 注意
- branch のどちらかが空でも壊れない構成にする
- 初期はシンプルに email / form を別々にまとめ、その後 Set ノードで整形してもよい

---

### 14. Call sub-channel-result-updater
Execute Workflow ノード。

#### 渡す入力
```json
{
  "execution_id": "={{$json.execution_id}}",
  "channel_results": "={{$json.channel_results}}"
}
```

#### 期待出力
- 最新化済み `channel_results`

---

### 15. Call sub-kpi-aggregator
Execute Workflow ノード。

#### 渡す入力
```json
{
  "customer_id": "={{$json.customer_id}}",
  "source_list_id": "={{$json.source_list_id}}",
  "execution_id": "={{$json.execution_id}}",
  "targets": "={{$json.targets}}",
  "channel_results": "={{$json.channel_results}}",
  "activity_logs": "={{$json.activity_logs}}"
}
```

#### 期待出力
- `kpi_snapshots`

---

### 16. Call sub-log-output-router
Execute Workflow ノード。

#### 渡す入力
```json
{
  "output_mode": "sheets",
  "activity_logs": "={{$json.activity_logs}}",
  "channel_results": "={{$json.channel_results}}",
  "kpi_snapshots": "={{$json.kpi_snapshots}}"
}
```

#### 期待出力
- `output_status`
- `adapter_results`

---

## 実行終了処理

### 17. Set Final Execution Status
最後に execution summary を作る。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "status": "completed",
  "processed_targets": "={{$json.targets.length}}",
  "email_target_count": "={{$json.email_targets.length}}",
  "form_target_count": "={{$json.form_targets.length}}",
  "output_status": "={{$json.output_status}}",
  "finished_at": "{{$now}}"
}
```

---

### 18. Return Summary
最後のノード出力を親 workflow の実行結果として扱う。

#### 目的
- テスト実行時に結果を見やすくする
- 後で execution レコード保存にも使えるようにする

---

## 接続順序

```text
Manual Trigger
  -> Set Input Payload
  -> Validate Entry Payload
    -> true  -> Set Execution Context
              -> Call sub-csv-normalizer
              -> Call sub-target-validator
              -> Call sub-channel-router
              -> Has Email Targets?
                   -> true -> Call sub-email-outreach
              -> Has Form Targets?
                   -> true -> Call sub-form-outreach
              -> Merge Channel Results
              -> Call sub-channel-result-updater
              -> Call sub-kpi-aggregator
              -> Call sub-log-output-router
              -> Set Final Execution Status
              -> Return Summary
    -> false -> Stop Invalid Input
```

---

## Error Workflow 設定

この workflow には `error-handler` を Settings で紐づける。

### 理由
- 実行失敗を共通処理に集約する
- main workflow 内に通知ロジックを埋め込まない
- validation error も Stop And Error 経由で収集できる

---

## Google Sheets に関する前提

MVP v1 では標準出力先を Google Sheets とする。

### 保存方針
- `activity_logs` は Append Row
- `channel_results` は Update Row または Append or Update Row
- `kpi_snapshots` は Append or Update Row

### 注意点
- `Append or Update Row` を使う場合は match column を固定する
- 非一意の列を match key にしない
- ID列は文字列型の揺れを避ける

---

## 実装順のおすすめ

最初の parent workflow 実装順は以下。

1. Manual Trigger
2. Set Input Payload
3. Validate Entry Payload
4. Stop Invalid Input
5. Set Execution Context
6. sub-csv-normalizer まで接続
7. sub-target-validator まで接続
8. sub-channel-router まで接続
9. sub-email-outreach / sub-form-outreach の片方ずつ接続
10. Merge と summary 作成
11. KPI と output router 接続
12. error-handler 紐づけ

---

## MVP v1 時点の割り切り

- 最初は並列実行を狙わない
- 初期は CSVダミーデータ中心で検証する
- Webhook入口は parent workflow が安定してから有効化する
- Google Sheets 出力は最低限のシートだけ先に作る
- ログの完全性より end-to-end 成功を先に確認する
