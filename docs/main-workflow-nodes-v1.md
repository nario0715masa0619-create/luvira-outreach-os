# main-ingest-and-run 実ノード構成案 v1

## 概要

この文書は、luvira-outreach-os の parent workflow `main-ingest-and-run` を n8n 上で実装するための実ノード構成案である。

この workflow の目的は、入力データの受け取りから、target 正規化、実行対象判定、チャネル振り分け、送信実行、結果更新、KPI 集計、最終出力までを統括することである。

MVP v1 では、parent workflow は orchestration に徹し、実処理の大半は sub-workflow に委譲する。

---

## 役割

- 実行入口を受け持つ
- 実行コンテキストを作る
- 正規化 sub-workflow を呼ぶ
- target validation を呼ぶ
- channel routing を呼ぶ
- email / form 送信 sub-workflow を呼ぶ
- channel result 更新を呼ぶ
- KPI 集計を呼ぶ
- 出力 router を呼ぶ
- 最終 execution summary を作る

---

## 実装方針

- parent workflow は `Execute Workflow` による sub-workflow 呼び出しを中核にする
- child 側は `Execute Sub-workflow Trigger` を入口にし、初期は `Accept all data` で受ける
- 実処理のロジックは parent に寄せすぎない
- parent では schema を束ね、sub-workflow 間の受け渡しを整理する
- email と form は routing 後に別 sub-workflow として実行する
- 実行順序は同期的に進め、各 sub-workflow の返却値を次段へ渡す
- エラー時は n8n の error workflow と `Stop And Error` を併用する

---

## workflow 全体像

```text
input
  -> build execution context
  -> sub-csv-normalizer
  -> sub-target-validator
  -> sub-channel-router
     -> sub-email-outreach
     -> sub-form-outreach
  -> sub-channel-result-updater
  -> sub-kpi-aggregator
  -> sub-log-output-router
  -> build final execution summary
```

---

## 想定入力

### パターンA: Webhook 起点
```json
{
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "input_type": "json",
  "targets": [
    {
      "company_name": "Example Inc.",
      "email": "test@example.com",
      "contact_name": "Yamada",
      "channel_hint": "email"
    }
  ]
}
```

### パターンB: CSV 読み込み後の配列起点
```json
{
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "input_type": "csv_rows",
  "rows": [
    {
      "company_name": "Example Inc.",
      "email": "test@example.com",
      "contact_name": "Yamada",
      "channel_hint": "email"
    }
  ]
}
```

---

## 想定出力

```json
{
  "execution_id": "ex_001",
  "output_status": "success",
  "adapter_results": [
    {
      "adapter": "google_sheets",
      "dataset": "activity_logs",
      "write_status": "success",
      "record_count": 10
    },
    {
      "adapter": "google_sheets",
      "dataset": "kpi_snapshots",
      "write_status": "success",
      "record_count": 1
    }
  ],
  "execution_summary": {
    "processed_targets": 10,
    "sent_count": 7,
    "failed_count": 2,
    "excluded_count": 1
  }
}
```

---

## 呼び出す sub-workflow

| Workflow | 役割 |
|---|---|
| `sub-csv-normalizer` | 入力行の標準 schema 化 |
| `sub-target-validator` | 実行対象妥当性判定 |
| `sub-channel-router` | email / form / excluded 分岐 |
| `sub-email-outreach` | email 送信 |
| `sub-form-outreach` | form 送信 |
| `sub-channel-result-updater` | latest channel result 更新 |
| `sub-kpi-aggregator` | execution 単位 KPI 集計 |
| `sub-log-output-router` | activity logs / KPI の保存 |

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Start / Trigger | Manual Trigger / Webhook / Schedule | 実行開始 |
| 2 | Validate Root Input | IF | 最低限の入力確認 |
| 3 | Stop Missing Root Input | Stop And Error | 必須入力不足で停止 |
| 4 | Build Execution Context | Set | execution context 生成 |
| 5 | Execute sub-csv-normalizer | Execute Workflow | 正規化 |
| 6 | Validate Normalized Targets | IF | 正規化結果確認 |
| 7 | Stop Missing Normalized Targets | Stop And Error | 正規化失敗時停止 |
| 8 | Execute sub-target-validator | Execute Workflow | 対象妥当性判定 |
| 9 | Execute sub-channel-router | Execute Workflow | チャネル分岐 |
| 10 | Has Email Targets? | IF | email 対象有無判定 |
| 11 | Execute sub-email-outreach | Execute Workflow | email 送信 |
| 12 | Build Empty Email Result | Set | email 対象なし時の空結果 |
| 13 | Has Form Targets? | IF | form 対象有無判定 |
| 14 | Execute sub-form-outreach | Execute Workflow | form 送信 |
| 15 | Build Empty Form Result | Set | form 対象なし時の空結果 |
| 16 | Merge Outreach Results | Merge / Code | email と form の結果統合 |
| 17 | Execute sub-channel-result-updater | Execute Workflow | channel result 更新 |
| 18 | Execute sub-kpi-aggregator | Execute Workflow | KPI 集計 |
| 19 | Build Output Payload | Set | 出力 router 用 payload 整形 |
| 20 | Execute sub-log-output-router | Execute Workflow | 最終出力保存 |
| 21 | Build Final Execution Summary | Set / Code | 最終 summary 作成 |
| 22 | Return Final Result | Set / Respond | 最終返却 |

---

## 各ノード詳細

### 1. Start / Trigger
入口ノード。

#### 候補
- Manual Trigger
- Webhook
- Schedule Trigger

#### MVP v1 推奨
- 初期は Manual Trigger または Webhook
- 定期処理は後から追加

---

### 2. Validate Root Input
IF ノードで最低限の入力を確認する。

#### 条件候補
- `customer_id` が存在する
- `source_list_id` が存在する
- `targets` または `rows` のどちらかが存在する

#### true
- `Build Execution Context` へ進む

#### false
- `Stop Missing Root Input` へ進む

---

### 3. Stop Missing Root Input
Stop And Error ノードで停止する。

#### メッセージ例
```text
main-ingest-and-run: customer_id, source_list_id, and input rows or targets are required.
```

---

### 4. Build Execution Context
Set ノードで親 workflow の基準 payload を作る。

#### 出力例
```json
{
  "execution_id": "={{'ex_' + $now.toMillis()}}",
  "customer_id": "={{$json.customer_id}}",
  "source_list_id": "={{$json.source_list_id}}",
  "input_type": "={{$json.input_type || 'json'}}",
  "rows": "={{$json.rows || []}}",
  "targets": "={{$json.targets || []}}",
  "output_mode": "sheets",
  "started_at": "={{$now}}"
}
```

#### 補足
- `execution_id` は親側で先に発行する
- sub-workflow へ同じ ID を渡す

---

### 5. Execute sub-csv-normalizer
`sub-csv-normalizer` を呼ぶ。

#### 渡す想定
- execution_id
- customer_id
- source_list_id
- input_type
- rows または targets

#### 期待返却
```json
{
  "targets": [...]
}
```

---

### 6. Validate Normalized Targets
IF ノードで正規化結果を確認する。

#### 条件
- `targets` が存在する

#### true
- `Execute sub-target-validator` へ進む

#### false
- `Stop Missing Normalized Targets` へ進む

---

### 7. Stop Missing Normalized Targets
Stop And Error ノードで停止する。

#### メッセージ例
```text
main-ingest-and-run: normalized targets is missing.
```

---

### 8. Execute sub-target-validator
`sub-target-validator` を呼ぶ。

#### 期待返却
```json
{
  "targets": [...]
}
```

#### 役割
- active / excluded / invalid を含む target 正規化済み状態を返す

---

### 9. Execute sub-channel-router
`sub-channel-router` を呼ぶ。

#### 期待返却
```json
{
  "email_targets": [...],
  "form_targets": [...],
  "excluded_targets": [...]
}
```

---

### 10. Has Email Targets?
IF ノードで email 対象有無を判定する。

#### 条件
- `email_targets.length > 0`

#### true
- `Execute sub-email-outreach` へ進む

#### false
- `Build Empty Email Result` へ進む

---

### 11. Execute sub-email-outreach
`sub-email-outreach` を呼ぶ。

#### 渡す想定
```json
{
  "execution_id": "ex_001",
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "targets": [...]
}
```

#### 期待返却
```json
{
  "channel_results": [...],
  "activity_logs": [...]
}
```

---

### 12. Build Empty Email Result
email 対象なしの場合の空結果を返す。

#### 出力例
```json
{
  "channel_results": [],
  "activity_logs": []
}
```

---

### 13. Has Form Targets?
IF ノードで form 対象有無を判定する。

#### 条件
- `form_targets.length > 0`

#### true
- `Execute sub-form-outreach` へ進む

#### false
- `Build Empty Form Result` へ進む

---

### 14. Execute sub-form-outreach
`sub-form-outreach` を呼ぶ。

#### 渡す想定
```json
{
  "execution_id": "ex_001",
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "targets": [...]
}
```

#### 期待返却
```json
{
  "channel_results": [...],
  "activity_logs": [...]
}
```

---

### 15. Build Empty Form Result
form 対象なしの場合の空結果を返す。

#### 出力例
```json
{
  "channel_results": [],
  "activity_logs": []
}
```

---

### 16. Merge Outreach Results
email と form の返却結果を 1 つに統合する。

#### 統合対象
- email `channel_results`
- form `channel_results`
- email `activity_logs`
- form `activity_logs`

#### 出力例
```json
{
  "channel_results": [
    {
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null
    },
    {
      "target_id": "t_010",
      "channel": "form",
      "provider": "http_request",
      "send_status": "failed",
      "failure_reason": "form_submit_failed"
    }
  ],
  "activity_logs": [
    {
      "target_id": "t_001",
      "activity_type": "email_sent",
      "activity_status": "success",
      "detail": "gmail send completed"
    },
    {
      "target_id": "t_010",
      "activity_type": "form_submitted",
      "activity_status": "failed",
      "detail": "form submit failed"
    }
  ]
}
```

#### 補足
- excluded target も activity log に残したいならここで追加してもよい
- 配列結合は Code ノードの方がわかりやすい場合がある

---

### 17. Execute sub-channel-result-updater
`sub-channel-result-updater` を呼ぶ。

#### 渡す想定
```json
{
  "execution_id": "ex_001",
  "channel_results": [...]
}
```

#### 期待返却
```json
{
  "channel_results": [...]
}
```

---

### 18. Execute sub-kpi-aggregator
`sub-kpi-aggregator` を呼ぶ。

#### 渡す想定
```json
{
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "execution_id": "ex_001",
  "targets": [...],
  "channel_results": [...],
  "activity_logs": [...]
}
```

#### 期待返却
```json
{
  "execution_id": "ex_001",
  "kpi_snapshots": [...]
}
```

---

### 19. Build Output Payload
`sub-log-output-router` に渡す payload を整形する。

#### 出力例
```json
{
  "output_mode": "sheets",
  "activity_logs": "={{$json.activity_logs || []}}",
  "channel_results": "={{$json.channel_results || []}}",
  "kpi_snapshots": "={{$json.kpi_snapshots || []}}"
}
```

---

### 20. Execute sub-log-output-router
`sub-log-output-router` を呼ぶ。

#### 期待返却
```json
{
  "output_status": "success",
  "adapter_results": [...]
}
```

---

### 21. Build Final Execution Summary
Set ノードまたは Code ノードで最終 summary を作る。

#### 集計例
- processed_targets
- sent_count
- failed_count
- excluded_count
- output_status

#### 出力例
```json
{
  "execution_id": "ex_001",
  "output_status": "success",
  "adapter_results": [
    {
      "adapter": "google_sheets",
      "dataset": "activity_logs",
      "write_status": "success",
      "record_count": 10
    }
  ],
  "execution_summary": {
    "processed_targets": 10,
    "sent_count": 7,
    "failed_count": 2,
    "excluded_count": 1
  }
}
```

---

### 22. Return Final Result
最終返却ノード。

#### 返却形式
```json
{
  "execution_id": "ex_001",
  "output_status": "success",
  "adapter_results": [
    {
      "adapter": "google_sheets",
      "dataset": "activity_logs",
      "write_status": "success",
      "record_count": 10
    },
    {
      "adapter": "google_sheets",
      "dataset": "kpi_snapshots",
      "write_status": "success",
      "record_count": 1
    }
  ],
  "execution_summary": {
    "processed_targets": 10,
    "sent_count": 7,
    "failed_count": 2,
    "excluded_count": 1
  }
}
```

---

## 接続順序

```text
Start / Trigger
  -> Validate Root Input
    -> true  -> Build Execution Context
              -> Execute sub-csv-normalizer
              -> Validate Normalized Targets
                   -> true  -> Execute sub-target-validator
                             -> Execute sub-channel-router
                             -> Has Email Targets?
                                  -> true  -> Execute sub-email-outreach
                                  -> false -> Build Empty Email Result
                             -> Has Form Targets?
                                  -> true  -> Execute sub-form-outreach
                                  -> false -> Build Empty Form Result
                             -> Merge Outreach Results
                             -> Execute sub-channel-result-updater
                             -> Execute sub-kpi-aggregator
                             -> Build Output Payload
                             -> Execute sub-log-output-router
                             -> Build Final Execution Summary
                             -> Return Final Result
                   -> false -> Stop Missing Normalized Targets
    -> false -> Stop Missing Root Input
```

---

## 実装上の注意

### 1. parent は orchestration に徹する
変換ロジックや判定ロジックを parent に持ち込みすぎない。
sub-workflow の責務を守る。

### 2. child は `Accept all data` で受け始める
初期実装時は柔軟性を優先し、後で schema を厳格化する。

### 3. Execute Workflow は同期的に扱う
parent が child の返却値を次段で使う前提にすると設計が単純になる。

### 4. Stop And Error を入力境界で使う
root input や normalized targets のような致命的欠損は早めに止める。

### 5. 空結果を許容する
email 対象なし、form 対象なしは正常系として扱う。
空配列を返せるようにする。

### 6. excluded target の扱いを決める
KPI 母数に含めるか、activity log に残すかを parent で統一する。

---

## 初期テストケース

### テスト1: email のみ
```json
{
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "input_type": "json",
  "targets": [
    {
      "company_name": "Example Inc.",
      "email": "test@example.com",
      "contact_name": "Yamada",
      "channel_hint": "email",
      "body_text": "本文"
    }
  ]
}
```

### 期待結果
- email sub-workflow が実行される
- form sub-workflow は空結果
- output_status = success

---

### テスト2: form のみ
```json
{
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "input_type": "json",
  "targets": [
    {
      "company_name": "Example Inc.",
      "submit_url": "https://example.com/contact/submit",
      "channel_hint": "form",
      "form_payload": {
        "name": "Yamada",
        "message": "本文"
      }
    }
  ]
}
```

### 期待結果
- form sub-workflow が実行される
- email sub-workflow は空結果

---

### テスト3: mixed
```json
{
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "input_type": "json",
  "targets": [
    {
      "company_name": "A Inc.",
      "email": "a@example.com",
      "channel_hint": "email",
      "body_text": "本文A"
    },
    {
      "company_name": "B Inc.",
      "submit_url": "https://example.com/contact/submit",
      "channel_hint": "form",
      "form_payload": {
        "name": "Yamada",
        "message": "本文B"
      }
    }
  ]
}
```

### 期待結果
- email / form の両方が実行される
- channel_results が統合される
- KPI が1 snapshot 生成される

---

### テスト4: 入力不足
```json
{
  "customer_id": "cust_001"
}
```

### 期待結果
- `Stop Missing Root Input` で停止

---

## MVP v1 時点の割り切り

- 並列 fan-out は行わない
- retry orchestration は持たない
- 顧客別 workflow 分岐は持たない
- advanced scheduling は持たない
- 入力ソースの自動判別は高度には行わない
- Browser Use 連携は parent 本体には入れない
- DB 永続化は標準では持たない

---

## error-handler 連携前提

この parent workflow で `Stop And Error` により停止した場合、
共通 `error-handler` が以下を拾える前提にする。

- workflow_name
- execution_id
- failed_node
- error_message
- timestamp

これにより、parent 側の境界エラーも共通運用で監視できる。
