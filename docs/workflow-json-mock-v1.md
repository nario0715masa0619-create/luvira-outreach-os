# luvira-outreach-os Workflow JSON Mock v1

## 概要
この文書は、luvira-outreach-os のMVP v1を n8n 上で実装する前段として作成する JSON モック仕様書である。

目的は以下の3点である。

- 実装前に workflow の分割単位を固定する
- parent workflow と sub-workflow の入出力構造を明文化する
- 後で n8n 実装者が export/import 可能な workflow JSON に落とし込みやすくする

本書で扱う JSON は、n8n の実際の export JSON をそのまま再現するものではなく、実装設計用の mock JSON である。

---

## 設計前提

- parent workflow が全体の制御を行う
- sub-workflow は責務単位で分離する
- sub-workflow は `Execute Sub-workflow Trigger` を先頭 node に置く
- parent は `Execute Workflow` で child を呼び出す
- 標準内部スキーマを流し、出力先差分は adapter 層で処理する
- error workflow は別 workflow として定義する
- MVP v1 では直列実行を前提とする

---

## 1. main-ingest-and-run

### 役割
- CSVまたはWebhookから開始
- 実行コンテキストを生成
- targets を正規化
- validation と routing を行う
- outreach / logging / result update / output / KPI まで制御する

### mock JSON
```json
{
  "workflow_name": "main-ingest-and-run",
  "workflow_type": "parent",
  "triggers": [
    {
      "node": "Manual Trigger",
      "purpose": "manual test run"
    },
    {
      "node": "Webhook",
      "purpose": "external ingestion",
      "mode": "optional"
    }
  ],
  "input": {
    "customer_id": "string",
    "source_type": "csv_upload | webhook_import",
    "run_mode": "email | form | both",
    "source_list_name": "string",
    "raw_rows": "array<object>",
    "settings_version": "string"
  },
  "steps": [
    "create execution context",
    "call sub-csv-normalizer",
    "call sub-target-validator",
    "call sub-channel-router",
    "call sub-email-outreach if needed",
    "call sub-form-outreach if needed",
    "call sub-channel-result-updater",
    "call sub-log-output-router",
    "call sub-kpi-aggregator",
    "mark execution completed"
  ],
  "output": {
    "execution_id": "string",
    "status": "completed | failed | partial_failed",
    "processed_targets": "number",
    "channel_results": "array<object>",
    "kpi_snapshots": "array<object>"
  }
}
```

### parent input example
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
    }
  ]
}
```

---

## 2. sub-csv-normalizer

### 役割
- CSV列名揺れ吸収
- 標準 target schema へ変換
- source_row_id 付与

### mock JSON
```json
{
  "workflow_name": "sub-csv-normalizer",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "source_list_id": "string",
    "raw_rows": "array<object>"
  },
  "output": {
    "targets": [
      {
        "target_id": "string",
        "source_list_id": "string",
        "source_row_id": "string",
        "company_name": "string",
        "website_url": "string|null",
        "contact_email": "string|null",
        "form_url": "string|null",
        "phone": "string|null",
        "industry": "string|null",
        "normalized_status": "active"
      }
    ]
  }
}
```

### output example
```json
{
  "targets": [
    {
      "target_id": "t_001",
      "source_list_id": "sl_001",
      "source_row_id": "row_001",
      "company_name": "Example Inc.",
      "website_url": "https://example.com",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact",
      "phone": null,
      "industry": null,
      "normalized_status": "active"
    }
  ]
}
```

---

## 3. sub-target-validator

### 役割
- 実行対象可否判定
- email/form eligibility の設定
- exclusion_reason の設定

### mock JSON
```json
{
  "workflow_name": "sub-target-validator",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Define using JSON example"
  },
  "input": {
    "targets": "array<object>"
  },
  "output": {
    "targets": [
      {
        "target_id": "string",
        "company_name": "string",
        "contact_email": "string|null",
        "form_url": "string|null",
        "channel_eligibility_email": "boolean",
        "channel_eligibility_form": "boolean",
        "normalized_status": "active | invalid | excluded",
        "exclusion_reason": "string|null"
      }
    ]
  }
}
```

---

## 4. sub-channel-router

### 役割
- run_mode と target 条件から送信チャネルを決定する

### mock JSON
```json
{
  "workflow_name": "sub-channel-router",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Define using fields below"
  },
  "input": {
    "execution_id": "string",
    "run_mode": "email | form | both",
    "targets": "array<object>"
  },
  "output": {
    "email_targets": "array<object>",
    "form_targets": "array<object>",
    "excluded_targets": "array<object>"
  }
}
```

### output example
```json
{
  "email_targets": [
    {
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "contact_email": "info@example.com"
    }
  ],
  "form_targets": [
    {
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "form_url": "https://example.com/contact"
    }
  ],
  "excluded_targets": []
}
```

---

## 5. sub-email-outreach

### 役割
- メール営業送信
- activity log と channel result を返す

### mock JSON
```json
{
  "workflow_name": "sub-email-outreach",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "execution_id": "string",
    "email_targets": "array<object>"
  },
  "output": {
    "activity_logs": "array<object>",
    "channel_results": "array<object>"
  }
}
```

### output example
```json
{
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "event_type": "sent",
      "status": "success",
      "occurred_at": "2026-06-30T16:00:00+09:00"
    }
  ],
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "attempted": true,
      "sent": true,
      "delivered": true,
      "last_status": "success",
      "last_updated_at": "2026-06-30T16:00:00+09:00"
    }
  ]
}
```

---

## 6. sub-form-outreach

### 役割
- フォーム営業送信
- activity log と channel result を返す

### mock JSON
```json
{
  "workflow_name": "sub-form-outreach",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "execution_id": "string",
    "form_targets": "array<object>"
  },
  "output": {
    "activity_logs": "array<object>",
    "channel_results": "array<object>"
  }
}
```

---

## 7. sub-activity-logger

### 役割
- event payload を標準 activity_logs schema に統一する

### mock JSON
```json
{
  "workflow_name": "sub-activity-logger",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "execution_id": "string",
    "target_id": "string",
    "channel": "email | form | system",
    "event_type": "string",
    "status": "string",
    "reason_code": "string|null",
    "reason_detail": "string|null",
    "payload_summary": "string|null"
  },
  "output": {
    "log_id": "string",
    "activity_log": {
      "log_id": "string",
      "execution_id": "string",
      "target_id": "string",
      "channel": "string",
      "event_type": "string",
      "status": "string",
      "reason_code": "string|null",
      "reason_detail": "string|null",
      "occurred_at": "datetime",
      "payload_summary": "string|null"
    }
  }
}
```

---

## 8. sub-channel-result-updater

### 役割
- channel_results の最新状態を整理する

### mock JSON
```json
{
  "workflow_name": "sub-channel-result-updater",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "execution_id": "string",
    "channel_results": "array<object>"
  },
  "output": {
    "channel_results": "array<object>"
  }
}
```

---

## 9. sub-log-output-router

### 役割
- 標準ログ・結果・KPIを出力先別に振り分ける

### mock JSON
```json
{
  "workflow_name": "sub-log-output-router",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "output_mode": "sheets | csv | webhook | db_map",
    "activity_logs": "array<object>",
    "channel_results": "array<object>",
    "kpi_snapshots": "array<object>|null"
  },
  "routes": [
    "log-output-sheets",
    "log-output-csv",
    "log-output-webhook",
    "log-output-db-map"
  ],
  "output": {
    "output_status": "success | failed",
    "adapter_results": "array<object>"
  }
}
```

---

## 10. log-output-sheets

### 役割
- Google Sheets 標準出力

### mock JSON
```json
{
  "workflow_name": "log-output-sheets",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "activity_logs": "array<object>",
    "channel_results": "array<object>",
    "kpi_snapshots": "array<object>|null"
  },
  "write_policy": {
    "activity_logs": "append",
    "channel_results": "append_or_update",
    "kpi_snapshots": "append_or_update"
  },
  "output": {
    "sheet_write_status": "success | failed"
  }
}
```

---

## 11. sub-kpi-aggregator

### 役割
- activity_logs と channel_results から KPI を計算する

### mock JSON
```json
{
  "workflow_name": "sub-kpi-aggregator",
  "workflow_type": "sub",
  "trigger": {
    "node": "Execute Sub-workflow Trigger",
    "input_mode": "Accept all data"
  },
  "input": {
    "customer_id": "string",
    "source_list_id": "string",
    "execution_id": "string",
    "targets": "array<object>",
    "channel_results": "array<object>",
    "activity_logs": "array<object>"
  },
  "output": {
    "kpi_snapshots": [
      {
        "customer_id": "cust_001",
        "period_month": "2026-06",
        "source_list_id": "sl_001",
        "total_targets": 100,
        "email_coverage_rate": 0.72,
        "email_success_rate": 0.64,
        "form_coverage_rate": 0.48,
        "form_success_rate": 0.31,
        "list_fit_score": 0.58
      }
    ]
  }
}
```

---

## 12. error-handler

### 役割
- 全 workflow 共通の error workflow
- production error を受けて記録と通知を行う

### mock JSON
```json
{
  "workflow_name": "error-handler",
  "workflow_type": "error",
  "trigger": {
    "node": "Error Trigger"
  },
  "input": {
    "execution": "object",
    "workflow": "object",
    "lastNodeExecuted": "string",
    "error": "object"
  },
  "steps": [
    "normalize error payload",
    "convert to activity_log compatible schema",
    "write error log",
    "notify operator"
  ],
  "output": {
    "error_logged": true,
    "notification_sent": true
  }
}
```

### error input example
```json
{
  "execution": {
    "id": "ex_001"
  },
  "workflow": {
    "name": "main-ingest-and-run"
  },
  "lastNodeExecuted": "sub-form-outreach",
  "error": {
    "message": "Webhook timeout",
    "stack": "..."
  }
}
```

---

## workflow 間接続ルール

### parent → child
- parent は `Execute Workflow` で child を呼ぶ
- child は `Execute Sub-workflow Trigger` を先頭に置く
- child の最後の node の output を parent が受け取る

### child output の基本原則
- `execution_id` を維持する
- `target_id` を維持する
- `source_list_id` を必要に応じて維持する
- 内部スキーマを書き換えずに返す

---

## 実装順序の推奨
MVP v1 は以下の順で作る。

1. main-ingest-and-run
2. sub-csv-normalizer
3. sub-target-validator
4. sub-channel-router
5. sub-email-outreach
6. sub-form-outreach
7. sub-channel-result-updater
8. log-output-sheets
9. sub-kpi-aggregator
10. error-handler

---

## MVP v1 実装メモ
- 最初は Manual Trigger 中心でテストする
- Webhook は production URL を使う前に test で検証する
- 実データ前に dummy rows で end-to-end を確認する
- Google Sheets 書き込み系はキー列を先に固定する
- 本物の export JSON は実装後に n8n UI から出力する
