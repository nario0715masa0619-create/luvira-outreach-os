# sub-email-outreach 実ノード構成案 v1

## 概要

この文書は、luvira-outreach-os の sub-workflow `sub-email-outreach` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、親 workflow または `sub-channel-router` から渡された email 対象 target を受け取り、
Gmail を使って送信を実行し、標準化された `channel_results` と `activity_logs` を返すことである。

MVP v1 では、複雑なパーソナライズやステップメールは扱わず、
単発送信と送信結果返却に責務を限定する。

---

## 役割

- email 対象 target を受け取る
- 必須項目を検証する
- Gmail 送信 payload を組み立てる
- Gmail で送信する
- 成功 / 失敗を `channel_results` に変換する
- 送信ログを `activity_logs` に変換する
- 親 workflow に返す

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger` を使う
- 初期は `Accept all data` で受ける
- 入力は `targets[]` を基本とする
- 1 target = 1 email を原則とする
- 送信は Gmail node を標準とする
- 本文は plain text でもよいが、将来的な拡張を考えて HTML 対応しやすい構造で持つ
- 失敗時は workflow を即停止させず、item 単位で結果を返す方針を優先する
- credential 破損など workflow 全体が継続不能な障害は error-handler に委ねる

---

## 入力仕様

### 想定入力
```json
{
  "execution_id": "ex_001",
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_001",
      "channel": "email",
      "email": "test@example.com",
      "company_name": "Example Inc.",
      "contact_name": "Yamada",
      "subject_template": "ご提案のご連絡",
      "body_text": "はじめまして。ご提案のご連絡です。"
    }
  ]
}
```

### 前提
- `targets[]` は email 対象のみが入っている想定
- channel routing は upstream 側で完了している想定
- `body_text` は upstream で完成済みでもよい
- `subject_template` がない場合は default subject を使う

---

## 出力仕様

### 返却するデータ
- `channel_results[]`
- `activity_logs[]`

### 返却イメージ
```json
{
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null
    }
  ],
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "activity_type": "email_sent",
      "activity_status": "success",
      "detail": "gmail send completed"
    }
  ]
}
```

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Email Targets Exists | IF | `targets` の存在確認 |
| 3 | Stop Missing Targets | Stop And Error | 入力不足時に停止 |
| 4 | Split Email Targets | Split Out / Item Lists | target を1件ずつ展開 |
| 5 | Validate Email Target Fields | IF | 必須項目確認 |
| 6 | Build Invalid Email Result | Set | 不正 target の結果化 |
| 7 | Build Email Payload | Set | Gmail 送信用 payload 整形 |
| 8 | Send Gmail | Gmail | メール送信 |
| 9 | Build Email Success Result | Set | success 用 `channel_results` / `activity_logs` 生成 |
| 10 | Build Email Failure Result | Set / Error Branch | failure 用 `channel_results` / `activity_logs` 生成 |
| 11 | Aggregate Email Results | Aggregate / Item Lists | `channel_results[]` に再集約 |
| 12 | Aggregate Activity Logs | Aggregate / Item Lists | `activity_logs[]` に再集約 |
| 13 | Return Email Outreach Payload | Set | 最終返却形式に整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノードとする。

#### Input data mode
- 初期は `Accept all data`

---

### 2. Validate Email Targets Exists
IF ノードで `targets` が存在し、かつ 1 件以上あるかを判定する。

#### 条件
- `targets` が存在する
- `targets.length > 0`

#### true
- `Split Email Targets` へ進む

#### false
- `Stop Missing Targets` へ進む

---

### 3. Stop Missing Targets
Stop And Error ノードで停止する。

#### メッセージ例
```text
sub-email-outreach: targets is missing or empty.
```

#### 意図
- upstream 異常を即時に検知する
- error-handler に伝わるメッセージを明確にする

---

### 4. Split Email Targets
`targets[]` を1件ずつ展開する。

#### 出力イメージ
```json
{
  "execution_id": "ex_001",
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "target_id": "t_001",
  "channel": "email",
  "email": "test@example.com",
  "company_name": "Example Inc.",
  "contact_name": "Yamada",
  "subject_template": "ご提案のご連絡",
  "body_text": "はじめまして。ご提案のご連絡です。"
}
```

---

### 5. Validate Email Target Fields
IF ノードで必須項目を確認する。

#### 条件
- `target_id` が存在する
- `email` が存在する
- `body_text` が存在する

#### true
- `Build Email Payload` へ進む

#### false
- `Build Invalid Email Result` へ進む

#### 補足
- `subject_template` は必須にしない
- contact_name や company_name も MVP v1 では任意

---

### 6. Build Invalid Email Result
Set ノードで、不正 target に対する `channel_results` と `activity_logs` を返す。

#### 出力例
```json
{
  "channel_result": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id || null}}",
    "channel": "email",
    "provider": "gmail",
    "send_status": "invalid",
    "failure_reason": "invalid_email_target_payload"
  },
  "activity_log": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id || null}}",
    "activity_type": "email_validation_failed",
    "activity_status": "failed",
    "detail": "invalid email target payload"
  }
}
```

---

### 7. Build Email Payload
Gmail node に渡すための送信 payload を Set ノードで整形する。

#### 出力項目例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "to_email": "={{$json.email}}",
  "email_subject": "={{$json.subject_template || 'ご連絡'}}",
  "email_body_text": "={{$json.body_text}}",
  "email_body_html": "={{$json.body_html || $json.body_text}}"
}
```

#### 補足
- HTML 本文がなければ text を流用してもよい
- subject は default fallback を持たせる
- 将来 personalization が増えても、このノードで吸収しやすい

---

### 8. Send Gmail
Gmail node で送信する。

#### 基本設定
- Resource: Message
- Operation: Send
- To: `={{$json.to_email}}`
- Subject: `={{$json.email_subject}}`
- Email Type: HTML または Text
- Message: `={{$json.email_body_html}}` または `={{$json.email_body_text}}`

#### 補足
- 初期は HTML より text 重視でもよい
- 本番では送信元 credential を固定する
- Gmail credential failure は item failure ではなく workflow error になりうる

---

### 9. Build Email Success Result
送信成功時の `channel_results` と `activity_logs` を Set ノードで生成する。

#### 出力例
```json
{
  "channel_result": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "channel": "email",
    "provider": "gmail",
    "send_status": "sent",
    "failure_reason": null
  },
  "activity_log": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "activity_type": "email_sent",
    "activity_status": "success",
    "detail": "gmail send completed"
  }
}
```

---

### 10. Build Email Failure Result
送信失敗時の `channel_results` と `activity_logs` を生成する。

#### 出力例
```json
{
  "channel_result": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "channel": "email",
    "provider": "gmail",
    "send_status": "failed",
    "failure_reason": "gmail_send_failed"
  },
  "activity_log": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "activity_type": "email_sent",
    "activity_status": "failed",
    "detail": "gmail send failed"
  }
}
```

#### 補足
- n8n 側の continue-on-fail 方針を使う場合、このノードで結果化しやすい
- エラーメッセージ詳細は必要に応じて `detail` に含めてもよい
- credential failure のような全体停止級エラーは error-handler 側で拾う

---

### 11. Aggregate Email Results
各 item の `channel_result` を `channel_results[]` に再集約する。

#### 出力イメージ
```json
{
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

---

### 12. Aggregate Activity Logs
各 item の `activity_log` を `activity_logs[]` に再集約する。

#### 出力イメージ
```json
{
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "activity_type": "email_sent",
      "activity_status": "success",
      "detail": "gmail send completed"
    }
  ]
}
```

---

### 13. Return Email Outreach Payload
最終返却形式を固定する。

#### 返却例
```json
{
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "email",
      "provider": "gmail",
      "send_status": "sent",
      "failure_reason": null
    }
  ],
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "activity_type": "email_sent",
      "activity_status": "success",
      "detail": "gmail send completed"
    }
  ]
}
```

---

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Email Targets Exists
    -> true  -> Split Email Targets
              -> Validate Email Target Fields
                   -> false -> Build Invalid Email Result
                   -> true  -> Build Email Payload
                             -> Send Gmail
                                -> success -> Build Email Success Result
                                -> failure -> Build Email Failure Result
              -> Aggregate Email Results
              -> Aggregate Activity Logs
              -> Return Email Outreach Payload
    -> false -> Stop Missing Targets
```

---

## 実装上の注意

### 1. 入力は email 対象だけに絞る
channel 分岐済みで渡す前提にすると、この workflow の責務が明確になる。

### 2. subject は fallback を持つ
subject 欠損で送信不能にならないよう、default 値を持たせる。

### 3. 送信失敗を item 結果へ変換する
すべてを workflow 停止にすると downstream 集計が難しくなる。
送信先単位で failed を返せる設計が扱いやすい。

### 4. credential failure は error-handler に任せる
個別 target の失敗と、workflow 全体障害は分けて考える。

### 5. HTML 本文は後方互換を持たせる
MVP v1 では text でもよいが、将来は HTML 前提に寄せやすい構造にしておく。

---

## 想定 activity_type

| activity_type | 意味 |
|---|---|
| `email_sent` | 送信実行 |
| `email_validation_failed` | 入力検証失敗 |

---

## 想定 failure_reason

| failure_reason | 意味 |
|---|---|
| `invalid_email_target_payload` | 必須項目不足 |
| `gmail_send_failed` | Gmail 送信失敗 |
| `missing_targets` | upstream から対象未受領 |

---

## 初期テストケース

### テスト1: 正常送信
```json
{
  "execution_id": "ex_001",
  "targets": [
    {
      "target_id": "t_001",
      "channel": "email",
      "email": "test@example.com",
      "subject_template": "ご提案のご連絡",
      "body_text": "はじめまして。ご提案のご連絡です。"
    }
  ]
}
```

### 期待結果
- Gmail 送信成功
- `channel_results[0].send_status = sent`
- `activity_logs[0].activity_status = success`

---

### テスト2: email 欠損
```json
{
  "execution_id": "ex_002",
  "targets": [
    {
      "target_id": "t_002",
      "channel": "email",
      "subject_template": "件名",
      "body_text": "本文"
    }
  ]
}
```

### 期待結果
- `send_status = invalid`
- `failure_reason = invalid_email_target_payload`

---

### テスト3: Gmail 送信失敗
```json
{
  "execution_id": "ex_003",
  "targets": [
    {
      "target_id": "t_003",
      "channel": "email",
      "email": "invalid@example.com",
      "subject_template": "件名",
      "body_text": "本文"
    }
  ]
}
```

### 期待結果
- `send_status = failed`
- `failure_reason = gmail_send_failed`

---

## MVP v1 時点の割り切り

- 添付ファイル送信は行わない
- CC / BCC は扱わない
- threading は扱わない
- 開封計測は行わない
- 返信検知は行わない
- ステップメールは行わない
- 送信速度制御は高度には行わない
- A/B subject test は行わない

---

## 親 workflow への返却前提

この workflow の出力は、親 workflow または後続の
`sub-channel-result-updater` と `sub-log-output-router` がそのまま受け取れる形にする。

### 前提出力
- `channel_results[]`
- `activity_logs[]`

これにより、送信処理、結果更新、ログ保存を疎結合で接続できる。
