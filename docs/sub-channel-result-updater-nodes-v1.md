# sub-channel-result-updater 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の sub-workflow `sub-channel-result-updater` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、email / form 各 workflow から返ってきた `channel_results` を受け取り、
target ごとの最新状態を標準出力先へ反映することである。

MVP v1 では、標準出力先を Google Sheets とし、target 単位の最新チャネル状態を 1 行に集約して保持する。
activity_logs はこの workflow では扱わず、ここでは `channel_results` の最新化だけに責務を限定する。

---

## 役割

- `channel_results` を受け取る。
- target 単位で結果を1件ずつ処理する。
- Google Sheets 上の既存行を特定する。
- 既存行があれば更新し、なければ新規追加する。
- 最新の `channel`, `provider`, `send_status`, `failure_reason` を反映する。
- 後続 workflow が使いやすい更新済み `channel_results` を返す。

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger` を使う。
- 初期は `Accept all data` で受ける。
- Google Sheets を latest-state store として使う。
- match key は `target_id` を基本にする。
- 1 item ごとに lookup → update / append の分岐で進める。
- `Append or Update Row` を使う案もあるが、MVP v1 の段階では lookup と明示分岐の方が挙動を追いやすい。
- この workflow では KPI 集計はしない。
- email と form の両方を同じスキーマで扱う。

---

## 入力仕様

### 想定入力
```json
{
  "execution_id": "ex_001",
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null
    },
    {
      "execution_id": "ex_001",
      "target_id": "t_002",
      "channel": "form",
      "provider": "http_request",
      "send_status": "failed",
      "failure_reason": "form_submit_failed"
    }
  ]
}
```

---

## 想定するGoogle Sheetsの保存列

### sheet名例
`channel_results_latest`

### 列例
| column | 意味 |
|---|---|
| target_id | 一意キー |
| last_execution_id | 最終実行ID |
| last_channel | 最終チャネル |
| last_provider | 最終送信プロバイダ |
| last_send_status | sent / failed |
| last_failure_reason | failure 内容 |
| updated_at | 最終更新日時 |

### 補足
- MVP v1 では `target_id` を一意キーにする。
- 将来は `customer_id` や `campaign_id` を含めた複合キー設計に変更してもよい。

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Channel Results Exists | IF | `channel_results` の存在確認 |
| 3 | Stop Missing Channel Results | Stop And Error | 必須入力不足時に停止 |
| 4 | Split Channel Results | Split Out / Item Lists | result を1件ずつ処理 |
| 5 | Validate Result Item | IF | `target_id`, `channel`, `send_status` の確認 |
| 6 | Build Invalid Result Item | Set | 不正 item のスキップ結果生成 |
| 7 | Build Sheet Upsert Payload | Set | Sheets 更新用項目整形 |
| 8 | Lookup Existing Target Row | Google Sheets | `target_id` で既存行検索 |
| 9 | Existing Row Found? | IF | 既存行の有無判定 |
| 10 | Update Existing Target Row | Google Sheets | 既存行更新 |
| 11 | Append New Target Row | Google Sheets | 新規行追加 |
| 12 | Build Updated Result Item | Set | 更新済み item 生成 |
| 13 | Aggregate Updated Channel Results | Aggregate / Item Lists | 配列へ再集約 |
| 14 | Return Updated Channel Results Payload | Set | 最終返却形式へ整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノードとする。

#### Input data mode
- 初期は `Accept all data`

#### 理由
- parent 側から柔軟に `channel_results` を受けられる。
- 安定後に schema 定義を厳格化できる。

---

### 2. Validate Channel Results Exists
IF ノードで `channel_results` が存在し、かつ 1 件以上あるかを判定する。

#### true
- `Split Channel Results` へ進む。

#### false
- `Stop Missing Channel Results` へ進む。

---

### 3. Stop Missing Channel Results
Stop And Error ノードで停止する。

#### メッセージ例
```text
sub-channel-result-updater: channel_results is missing or empty.
```

#### 意図
- 親 workflow または upstream workflow の異常を即時に検知する。
- error workflow へ流せるようにする。

---

### 4. Split Channel Results
`channel_results` 配列を1件ずつ展開する。

#### 出力イメージ
```json
{
  "execution_id": "ex_001",
  "target_id": "t_001",
  "channel": "email",
  "provider": "gmail",
  "send_status": "sent",
  "failure_reason": null
}
```

---

### 5. Validate Result Item
IF ノードで最低限の必須項目を確認する。

#### 条件
- `target_id` が存在する。
- `channel` が存在する。
- `send_status` が存在する。

#### true
- `Build Sheet Upsert Payload` へ進む。

#### false
- `Build Invalid Result Item` へ進む。

---

### 6. Build Invalid Result Item
Set ノードで、不正 item をそのまま結果として返すためのオブジェクトを作る。

#### 出力例
```json
{
  "target_id": "={{$json.target_id || null}}",
  "channel": "={{$json.channel || null}}",
  "send_status": "invalid",
  "failure_reason": "invalid_channel_result_payload",
  "update_status": "skipped"
}
```

#### ポイント
- invalid item も落とさず結果に残す。
- 後で upstream の品質確認に使える。

---

### 7. Build Sheet Upsert Payload
Google Sheets 更新用の項目に整形する Set ノード。

#### 出力例
```json
{
  "target_id": "={{$json.target_id}}",
  "last_execution_id": "={{$json.execution_id}}",
  "last_channel": "={{$json.channel}}",
  "last_provider": "={{$json.provider}}",
  "last_send_status": "={{$json.send_status}}",
  "last_failure_reason": "={{$json.failure_reason}}",
  "updated_at": "={{$now}}"
}
```

#### 理由
- Sheets 側の列名に合わせてここで整形する。
- 後続の lookup / update / append ノードを単純化できる。

---

### 8. Lookup Existing Target Row
Google Sheets ノードで `Get Row(s)` または Lookup 相当の方法を使い、`target_id` をキーに既存行を探す。

#### 設定イメージ
- Operation: `Get Row(s)` または lookup 相当
- Filter column: `target_id`
- Filter value: `={{$json.target_id}}`

#### 注意
- `Always Output Data` 相当の考え方で、未検出でも後続分岐に進める形が扱いやすい。
- 一意キー前提なので複数行ヒットは原則起きない設計にする。

---

### 9. Existing Row Found?
IF ノードで既存行の有無を判定する。

#### 条件
- lookup 結果に `target_id` がある

#### true
- `Update Existing Target Row` へ進む。

#### false
- `Append New Target Row` へ進む。

---

### 10. Update Existing Target Row
Google Sheets ノードで既存行を更新する。

#### 設定方針
- Operation: `Update Row`
- Match or row selection: lookup で見つけた行を対象にする
- 更新列:
  - `last_execution_id`
  - `last_channel`
  - `last_provider`
  - `last_send_status`
  - `last_failure_reason`
  - `updated_at`

#### 注意
- row 番号ベース更新か key ベース更新かは実装時の node 設定に合わせて選ぶ。
- 一意性が曖昧な列を match key にしない。

---

### 11. Append New Target Row
Google Sheets ノードで新規行を追加する。

#### 設定方針
- Operation: `Append Row`
- 追加列:
  - `target_id`
  - `last_execution_id`
  - `last_channel`
  - `last_provider`
  - `last_send_status`
  - `last_failure_reason`
  - `updated_at`

#### 補足
- 実装を簡略化したい場合は `Append or Update Row` に寄せてもよい。
- ただし初期は lookup と分岐を明示した方がデバッグしやすい。

---

### 12. Build Updated Result Item
Sheets 更新後に返す `channel_results` 用の item を整形する。

#### 出力例
```json
{
  "target_id": "={{$json.target_id}}",
  "channel": "={{$json.last_channel || $json.channel}}",
  "provider": "={{$json.last_provider || $json.provider}}",
  "send_status": "={{$json.last_send_status || $json.send_status}}",
  "failure_reason": "={{$json.last_failure_reason || $json.failure_reason}}",
  "update_status": "updated"
}
```

#### 補足
- append の場合も update_status は `updated` ではなく `inserted` にしてもよい。
- MVP v1 では `updated` に寄せてもよいが、厳密に分けるなら `inserted` / `updated` を使い分ける。

---

### 13. Aggregate Updated Channel Results
個別 item を `channel_results` 配列へ再集約する。

#### 目的
- parent workflow がそのまま次段に渡せる形式に戻す。
- invalid / updated / inserted をまとめて返す。

---

### 14. Return Updated Channel Results Payload
最終返却形式を固定する。

#### 返却例
```json
{
  "channel_results": [
    {
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null,
      "update_status": "updated"
    },
    {
      "target_id": "t_002",
      "channel": "form",
      "provider": "http_request",
      "send_status": "failed",
      "failure_reason": "form_submit_failed",
      "update_status": "inserted"
    }
  ]
}
```

---

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Channel Results Exists
    -> true  -> Split Channel Results
              -> Validate Result Item
                   -> false -> Build Invalid Result Item
                   -> true  -> Build Sheet Upsert Payload
                             -> Lookup Existing Target Row
                             -> Existing Row Found?
                                -> true  -> Update Existing Target Row
                                -> false -> Append New Target Row
                             -> Build Updated Result Item
              -> Aggregate Updated Channel Results
              -> Return Updated Channel Results Payload
    -> false -> Stop Missing Channel Results
```

---

## 実装上の注意

### 1. 初期は lookup + IF + update/append でよい
`Append or Update Row` は便利だが、初期設計の確認フェーズでは lookup と分岐を明示した方が挙動を追いやすい。

### 2. target_id を一意キーにする
Google Sheets はDBではないため、曖昧なキーを使うと更新事故が起きやすい。
MVP v1 では `target_id` を固定キーにした方が安全。

### 3. activity_logs はここで触らない
この workflow の責務を latest result update に限定すると保守しやすい。
ログ出力は別 workflow に寄せた方が後で差し替えやすい。

### 4. inserted / updated を分けると運用が見やすい
最初は単純化してもよいが、本番では新規登録と既存更新の区別がある方が監視しやすい。

---

## 初期テストケース

### テスト1: 既存 target 更新
```json
{
  "execution_id": "ex_001",
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null
    }
  ]
}
```

### 期待結果
- lookup で既存行を検出する。
- `Update Row` が動く。
- `update_status = updated`

---

### テスト2: 新規 target 追加
```json
{
  "execution_id": "ex_002",
  "channel_results": [
    {
      "execution_id": "ex_002",
      "target_id": "t_999",
      "channel": "form",
      "provider": "http_request",
      "send_status": "failed",
      "failure_reason": "form_submit_failed"
    }
  ]
}
```

### 期待結果
- lookup で未検出。
- `Append Row` が動く。
- `update_status = inserted` または `updated`

---

### テスト3: 不正 item
```json
{
  "execution_id": "ex_003",
  "channel_results": [
    {
      "execution_id": "ex_003",
      "target_id": null,
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null
    }
  ]
}
```

### 期待結果
- `update_status = skipped`
- `failure_reason = invalid_channel_result_payload`

---

## MVP v1 時点の割り切り

- customer 単位の分離は行わない。
- campaign 単位の履歴管理は行わない。
- 同一 target の複数送信履歴はここでは持たない。
- Sheets の排他制御は行わない。
- 複数行ヒット時の高度解決は行わない。
- activity_logs の更新はここで扱わない。

---

## 次の sub-workflow への受け渡し前提

この workflow の出力は、親 workflow または `sub-kpi-aggregator` がそのまま受け取れる形にする。

### 前提出力
- `channel_results[]`
- 各 result に以下を含む
  - `target_id`
  - `channel`
  - `provider`
  - `send_status`
  - `failure_reason`
  - `update_status`

これにより、次段では「送信結果の最新状態が保存済み」という前提で KPI 集計に集中できる。
