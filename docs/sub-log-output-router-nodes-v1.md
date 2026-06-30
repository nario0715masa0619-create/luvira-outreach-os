# sub-log-output-router 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の sub-workflow `sub-log-output-router` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、親 workflow または各 sub-workflow から渡された `activity_logs`、`channel_results`、`kpi_snapshots` を受け取り、
標準出力先へ保存することである。

MVP v1 では、標準出力先を Google Sheets とし、
以下の3系統を別シートへ保存する。

- activity logs
- channel results latest
- KPI snapshots

この workflow は「どこへ保存するか」を扱うルータ兼保存処理として機能し、保存処理後の状態を返す。

---

## 役割

- `output_mode`、`activity_logs`、`channel_results`、`kpi_snapshots` を受け取る。
- 保存対象ごとにデータを分岐する。
- Google Sheets の各シートへ保存する。
- 保存成功 / 失敗の要約を返す。
- 親 workflow が execution summary に使える `output_status` と `adapter_results` を返す。

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger` を使う。
- 初期は `Accept all data` で受ける。
- MVP v1 の `output_mode` は `sheets` のみを正式サポートとする。
- `activity_logs`、`channel_results`、`kpi_snapshots` は別シートに分ける。
- `channel_results` は latest-state sheet への保存前提とする。
- 保存対象が空配列でも workflow 全体は失敗させない。
- 保存成否は item 単位ではなくカテゴリ単位で summary 化する。

---

## 入力仕様

### 想定入力
```json
{
  "output_mode": "sheets",
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "activity_type": "email_sent",
      "activity_status": "success",
      "detail": "gmail send completed"
    }
  ],
  "channel_results": [
    {
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null,
      "update_status": "updated"
    }
  ],
  "kpi_snapshots": [
    {
      "customer_id": "cust_001",
      "source_list_id": "sl_001",
      "execution_id": "ex_001",
      "snapshot_at": "2026-06-30T18:00:00.000Z",
      "processed_targets": 10,
      "active_targets": 7,
      "excluded_targets": 2,
      "invalid_targets": 1,
      "total_channel_results": 8,
      "sent_count": 6,
      "failed_count": 2
    }
  ]
}
```

---

## 想定するGoogle Sheets構成

### spreadsheet例
`luvira-outreach-os-logbook`

### sheet構成
| sheet名 | 保存内容 |
|---|---|
| `activity_logs` | 実行履歴の行単位ログ |
| `channel_results_latest` | target単位の最新状態 |
| `kpi_snapshots` | execution単位KPI |

### 補足
- `channel_results_latest` は通常 `sub-channel-result-updater` 側でも扱うため、ここでは「保存済み結果の再出力先」として扱うか、MVPでは保存対象から外す運用も可能。
- ただしルータ設計としては受け取れる形にしておく。

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Output Mode | IF | `output_mode` の確認 |
| 3 | Stop Invalid Output Mode | Stop And Error | 未対応 mode の停止 |
| 4 | Prepare Output Payload | Set | 入力配列の初期化 |
| 5 | Has Activity Logs? | IF | activity_logs 有無判定 |
| 6 | Split Activity Logs | Split Out / Item Lists | log を1件ずつ展開 |
| 7 | Append Activity Log Row | Google Sheets | `activity_logs` シートへ追加 |
| 8 | Build Activity Output Result | Set | activity 保存結果 summary |
| 9 | Has Channel Results? | IF | channel_results 有無判定 |
| 10 | Split Channel Results | Split Out / Item Lists | result を1件ずつ展開 |
| 11 | Append Channel Result Row | Google Sheets | `channel_results_latest` シートへ保存 |
| 12 | Build Channel Output Result | Set | channel 保存結果 summary |
| 13 | Has KPI Snapshots? | IF | kpi_snapshots 有無判定 |
| 14 | Split KPI Snapshots | Split Out / Item Lists | KPI を1件ずつ展開 |
| 15 | Append KPI Snapshot Row | Google Sheets | `kpi_snapshots` シートへ追加 |
| 16 | Build KPI Output Result | Set | KPI 保存結果 summary |
| 17 | Merge Output Results | Merge / Code | 保存結果の統合 |
| 18 | Build Final Output Status | Set | `output_status` と `adapter_results` 生成 |
| 19 | Return Output Router Payload | Set | 最終返却形式へ整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノードとする。

#### Input data mode
- 初期は `Accept all data`

#### 理由
- parent 側の返却形が多少増減しても受けやすい。
- output adapter を後から増やしやすい。

---

### 2. Validate Output Mode
IF ノードで `output_mode` を確認する。

#### 条件
- `output_mode == sheets`

#### true
- `Prepare Output Payload` へ進む。

#### false
- `Stop Invalid Output Mode` へ進む。

#### 補足
- 将来 `db`, `bigquery`, `notion` などを増やしてもよい。
- MVP v1 では `sheets` のみ。

---

### 3. Stop Invalid Output Mode
Stop And Error ノードで停止する。

#### メッセージ例
```text
sub-log-output-router: unsupported output_mode. Only sheets is supported in MVP v1.
```

---

### 4. Prepare Output Payload
Set ノードで各配列を初期化する。

#### 出力例
```json
{
  "output_mode": "={{$json.output_mode}}",
  "activity_logs": "={{$json.activity_logs || []}}",
  "channel_results": "={{$json.channel_results || []}}",
  "kpi_snapshots": "={{$json.kpi_snapshots || []}}"
}
```

#### 目的
- 後続 IF 判定を単純化する。
- undefined を空配列へ寄せる。

---

## activity_logs 保存系

### 5. Has Activity Logs?
IF ノード。

#### 条件
- `activity_logs.length > 0`

#### true
- `Split Activity Logs` へ進む。

#### false
- activity logs は保存対象なしとして summary のみ返す。

---

### 6. Split Activity Logs
`activity_logs` を1件ずつ展開する。

### 7. Append Activity Log Row
Google Sheets ノードで `activity_logs` シートへ `Append Row` する。

#### 保存列例
- execution_id
- target_id
- activity_type
- activity_status
- detail
- logged_at

#### 補足
- `logged_at` はここで `{{$now}}` を追加してもよい。

---

### 8. Build Activity Output Result
保存結果の summary を作る。

#### 出力例
```json
{
  "adapter": "google_sheets",
  "dataset": "activity_logs",
  "write_status": "success",
  "record_count": 12
}
```

#### 補足
- 実際の件数は Split 前件数か、Append 成功件数を採用する。
- MVP v1 では件数の厳密一致より「保存した」という結果を優先してよい。

---

## channel_results 保存系

### 9. Has Channel Results?
IF ノード。

#### 条件
- `channel_results.length > 0`

#### true
- `Split Channel Results` へ進む。

#### false
- channel results は保存対象なしとして summary のみ返す。

---

### 10. Split Channel Results
`channel_results` を1件ずつ展開する。

### 11. Append Channel Result Row
Google Sheets ノードで `channel_results_latest` シートへ保存する。

#### 保存方式
- MVP v1 で最新状態しか持たないなら、本来は update/upsert が望ましい。
- ただし `sub-channel-result-updater` 側で latest 化済みなら、ここでは append ではなく保存省略も可能。
- 設計として残す場合は `Append or Update Row` を使う選択肢がある。

#### 保存列例
- target_id
- channel
- provider
- send_status
- failure_reason
- update_status
- logged_at

#### 注意
- 二重保存が起きると責務が被る。
- 実装時は「updater 側で保存するか」「router 側で保存するか」のどちらかに寄せる。

---

### 12. Build Channel Output Result
保存結果の summary を作る。

#### 出力例
```json
{
  "adapter": "google_sheets",
  "dataset": "channel_results_latest",
  "write_status": "success",
  "record_count": 8
}
```

---

## KPI 保存系

### 13. Has KPI Snapshots?
IF ノード。

#### 条件
- `kpi_snapshots.length > 0`

#### true
- `Split KPI Snapshots` へ進む。

#### false
- KPI は保存対象なしとして summary のみ返す。

---

### 14. Split KPI Snapshots
`kpi_snapshots` を1件ずつ展開する。

### 15. Append KPI Snapshot Row
Google Sheets ノードで `kpi_snapshots` シートへ `Append Row` する。

#### 保存列例
- customer_id
- source_list_id
- execution_id
- snapshot_at
- processed_targets
- active_targets
- excluded_targets
- invalid_targets
- total_channel_results
- sent_count
- failed_count
- email_sent_count
- form_sent_count
- email_failed_count
- form_failed_count

#### 理由
- KPI snapshot は execution ごとに1行で持つ方が扱いやすい。
- 履歴性を持たせたいので append が基本。

---

### 16. Build KPI Output Result
保存結果の summary を作る。

#### 出力例
```json
{
  "adapter": "google_sheets",
  "dataset": "kpi_snapshots",
  "write_status": "success",
  "record_count": 1
}
```

---

## 保存結果統合

### 17. Merge Output Results
Merge ノードまたは Code ノードで、
activity / channel / KPI の保存結果 summary を1つにまとめる。

#### 目的
- downstream が保存カテゴリごとの成否をまとめて見られるようにする。

#### 出力イメージ
```json
{
  "adapter_results": [
    {
      "adapter": "google_sheets",
      "dataset": "activity_logs",
      "write_status": "success",
      "record_count": 12
    },
    {
      "adapter": "google_sheets",
      "dataset": "channel_results_latest",
      "write_status": "success",
      "record_count": 8
    },
    {
      "adapter": "google_sheets",
      "dataset": "kpi_snapshots",
      "write_status": "success",
      "record_count": 1
    }
  ]
}
```

---

### 18. Build Final Output Status
Set ノードで最終的な `output_status` を作る。

#### ロジック
- すべて success → `success`
- 一部 success / 一部 skipped → `partial_success`
- すべて skipped → `skipped`
- 失敗が含まれる → `partial_failed` または `failed`

#### 出力例
```json
{
  "output_status": "success",
  "adapter_results": [
    {
      "adapter": "google_sheets",
      "dataset": "activity_logs",
      "write_status": "success",
      "record_count": 12
    },
    {
      "adapter": "google_sheets",
      "dataset": "kpi_snapshots",
      "write_status": "success",
      "record_count": 1
    }
  ]
}
```

---

### 19. Return Output Router Payload
最終返却形式を固定する。

#### 返却例
```json
{
  "output_status": "success",
  "adapter_results": [
    {
      "adapter": "google_sheets",
      "dataset": "activity_logs",
      "write_status": "success",
      "record_count": 12
    },
    {
      "adapter": "google_sheets",
      "dataset": "channel_results_latest",
      "write_status": "success",
      "record_count": 8
    },
    {
      "adapter": "google_sheets",
      "dataset": "kpi_snapshots",
      "write_status": "success",
      "record_count": 1
    }
  ]
}
```

---

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Output Mode
    -> true  -> Prepare Output Payload
              -> Has Activity Logs?
                   -> true  -> Split Activity Logs
                             -> Append Activity Log Row
                             -> Build Activity Output Result
              -> Has Channel Results?
                   -> true  -> Split Channel Results
                             -> Append Channel Result Row
                             -> Build Channel Output Result
              -> Has KPI Snapshots?
                   -> true  -> Split KPI Snapshots
                             -> Append KPI Snapshot Row
                             -> Build KPI Output Result
              -> Merge Output Results
              -> Build Final Output Status
              -> Return Output Router Payload
    -> false -> Stop Invalid Output Mode
```

---

## 実装上の注意

### 1. updater と router の責務重複を避ける
`channel_results_latest` を updater で保存するなら、router 側では保存しない方がよい。
ここは実装時に片方へ寄せる。

### 2. 空配列は失敗ではない
activity logs や KPI が空でも、その execution の結果としては正常な場合がある。
空配列は skipped 扱いでよい。

### 3. category 単位の保存結果を返す
item 単位の詳細な保存成否は downstream には不要なことが多い。
MVP v1 では dataset 単位の summary で十分。

### 4. append と upsert を使い分ける
- 履歴を残すもの: append
- 最新状態だけ持つもの: update / append-or-update

---

## 初期テストケース

### テスト1: 3種類すべてあり
```json
{
  "output_mode": "sheets",
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "activity_type": "email_sent",
      "activity_status": "success",
      "detail": "gmail send completed"
    }
  ],
  "channel_results": [
    {
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null,
      "update_status": "updated"
    }
  ],
  "kpi_snapshots": [
    {
      "customer_id": "cust_001",
      "source_list_id": "sl_001",
      "execution_id": "ex_001",
      "snapshot_at": "2026-06-30T18:00:00.000Z",
      "processed_targets": 10,
      "active_targets": 7,
      "excluded_targets": 2,
      "invalid_targets": 1,
      "total_channel_results": 8,
      "sent_count": 6,
      "failed_count": 2
    }
  ]
}
```

### 期待結果
- 各 dataset が保存される。
- `output_status = success`

---

### テスト2: KPIだけある
```json
{
  "output_mode": "sheets",
  "activity_logs": [],
  "channel_results": [],
  "kpi_snapshots": [
    {
      "customer_id": "cust_001",
      "source_list_id": "sl_001",
      "execution_id": "ex_002",
      "snapshot_at": "2026-06-30T18:10:00.000Z",
      "processed_targets": 0,
      "active_targets": 0,
      "excluded_targets": 0,
      "invalid_targets": 0,
      "total_channel_results": 0,
      "sent_count": 0,
      "failed_count": 0
    }
  ]
}
```

### 期待結果
- activity / channel は skipped
- KPI は success
- `output_status = partial_success` または `success` のどちらかに設計統一する

---

### テスト3: 未対応 output_mode
```json
{
  "output_mode": "database",
  "activity_logs": [],
  "channel_results": [],
  "kpi_snapshots": []
}
```

### 期待結果
- Stop Invalid Output Mode で停止

---

## MVP v1 時点の割り切り

- BigQuery, Notion, DB 出力は行わない。
- dataset ごとの retry 制御は行わない。
- item 単位保存失敗の詳細分析は行わない。
- file export は行わない。
- 複数 spreadsheet への分散保存は行わない。
- 保存先の自動作成は行わない。

---

## 親 workflow への返却前提

この workflow の出力は、親 workflow の `Set Final Execution Status` がそのまま受け取れる形にする。

### 前提出力
- `output_status`
- `adapter_results[]`

これにより親 workflow 側は、
「送信実行」「結果更新」「KPI集計」「最終出力」のすべてを終えた summary を簡単に組み立てられる。
