# sub-form-outreach 実ノード構成案 v1

## 概要

この文書は、luvira-outreach-os の sub-workflow `sub-form-outreach` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、親 workflow または `sub-channel-router` から渡された form 対象 target を受け取り、
HTTP Request を使って問い合わせフォーム送信を実行し、標準化された `channel_results` と `activity_logs` を返すことである。

MVP v1 では、単純な POST 送信または事前定義済みフォーム送信に責務を限定し、
JavaScript 実行必須フォーム、CAPTCHA、multi-step form には対応しない。

---

## 役割

- form 対象 target を受け取る
- 必須項目を検証する
- HTTP Request 用 payload を組み立てる
- form 送信を実行する
- 成功 / 失敗を `channel_results` に変換する
- 実行ログを `activity_logs` に変換する
- 親 workflow に返す

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger` を使う
- 初期は `Accept all data` で受ける
- 入力は `targets[]` を基本とする
- 1 target = 1 form submit を原則とする
- 送信は HTTP Request node を標準とする
- 失敗時は item 単位で結果を返す方針を優先する
- node 設定の error output を使うか、明示的な分岐で failure を結果化する
- HTTP レスポンスだけで成功判定しづらい場合に備えて、簡易 success 条件を定義する
- CAPTCHA や JS 必須フォームは upstream で除外するか、MVP 外として失敗扱いに寄せる

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
      "target_id": "t_010",
      "channel": "form",
      "form_url": "https://example.com/contact",
      "submit_url": "https://example.com/contact/submit",
      "http_method": "POST",
      "form_payload": {
        "company": "Example Inc.",
        "name": "Yamada",
        "email": "test@example.com",
        "message": "お問い合わせ本文です。"
      },
      "success_keyword": "お問い合わせありがとうございました"
    }
  ]
}
```

### 前提
- `targets[]` は form 対象のみが入っている想定
- channel routing は upstream 側で完了している想定
- `form_payload` は upstream で完成済みでもよい
- `submit_url` がある target のみ MVP v1 で送信対象とする
- `success_keyword` は任意だが、可能なら持たせる

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
      "target_id": "t_010",
      "channel": "form",
      "provider": "http_request",
      "send_status": "sent",
      "failure_reason": null
    }
  ],
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_010",
      "activity_type": "form_submitted",
      "activity_status": "success",
      "detail": "form submit completed"
    }
  ]
}
```

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Form Targets Exists | IF | `targets` の存在確認 |
| 3 | Stop Missing Targets | Stop And Error | 入力不足時に停止 |
| 4 | Split Form Targets | Split Out / Item Lists | target を1件ずつ展開 |
| 5 | Validate Form Target Fields | IF | 必須項目確認 |
| 6 | Build Invalid Form Result | Set | 不正 target の結果化 |
| 7 | Build Form Request Payload | Set | HTTP Request 用 payload 整形 |
| 8 | Submit Form Request | HTTP Request | form 送信 |
| 9 | Evaluate Form Response | IF / Code | 成功判定 |
| 10 | Build Form Success Result | Set | success 用 `channel_results` / `activity_logs` 生成 |
| 11 | Build Form Failure Result | Set | failure 用 `channel_results` / `activity_logs` 生成 |
| 12 | Aggregate Form Results | Aggregate / Item Lists | `channel_results[]` に再集約 |
| 13 | Aggregate Activity Logs | Aggregate / Item Lists | `activity_logs[]` に再集約 |
| 14 | Return Form Outreach Payload | Set | 最終返却形式に整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノードとする。

#### Input data mode
- 初期は `Accept all data`

---

### 2. Validate Form Targets Exists
IF ノードで `targets` が存在し、かつ 1 件以上あるかを判定する。

#### 条件
- `targets` が存在する
- `targets.length > 0`

#### true
- `Split Form Targets` へ進む

#### false
- `Stop Missing Targets` へ進む

---

### 3. Stop Missing Targets
Stop And Error ノードで停止する。

#### メッセージ例
```text
sub-form-outreach: targets is missing or empty.
```

---

### 4. Split Form Targets
`targets[]` を1件ずつ展開する。

#### 出力イメージ
```json
{
  "execution_id": "ex_001",
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "target_id": "t_010",
  "channel": "form",
  "form_url": "https://example.com/contact",
  "submit_url": "https://example.com/contact/submit",
  "http_method": "POST",
  "form_payload": {
    "company": "Example Inc.",
    "name": "Yamada",
    "email": "test@example.com",
    "message": "お問い合わせ本文です。"
  },
  "success_keyword": "お問い合わせありがとうございました"
}
```

---

### 5. Validate Form Target Fields
IF ノードで必須項目を確認する。

#### 条件
- `target_id` が存在する
- `submit_url` が存在する
- `form_payload` が存在する

#### true
- `Build Form Request Payload` へ進む

#### false
- `Build Invalid Form Result` へ進む

#### 補足
- `http_method` は未指定なら `POST` を既定値にする
- `form_url` は任意
- `success_keyword` は任意

---

### 6. Build Invalid Form Result
Set ノードで、不正 target に対する `channel_results` と `activity_logs` を返す。

#### 出力例
```json
{
  "channel_result": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id || null}}",
    "channel": "form",
    "provider": "http_request",
    "send_status": "invalid",
    "failure_reason": "invalid_form_target_payload"
  },
  "activity_log": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id || null}}",
    "activity_type": "form_validation_failed",
    "activity_status": "failed",
    "detail": "invalid form target payload"
  }
}
```

---

### 7. Build Form Request Payload
HTTP Request node に渡す送信 payload を Set ノードで整形する。

#### 出力項目例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "request_url": "={{$json.submit_url}}",
  "request_method": "={{$json.http_method || 'POST'}}",
  "request_body": "={{$json.form_payload}}",
  "success_keyword": "={{$json.success_keyword || ''}}"
}
```

#### 補足
- `Content-Type` は target ごとに固定でもよい
- MVP v1 では `application/x-www-form-urlencoded` または JSON のどちらかに寄せる
- 最初はフォーム送信対象を限定した方が保守しやすい

---

### 8. Submit Form Request
HTTP Request node で form 送信する。

#### 基本設定例
- Method: `={{$json.request_method}}`
- URL: `={{$json.request_url}}`
- Send Body: true
- Body: `={{$json.request_body}}`
- Response Format: text または JSON
- On Error: `Continue (using error output)` を検討する

#### 補足
- レスポンス body を success 判定に使う場合は text で保持する
- API 的 endpoint なら status code 中心で判定してよい
- フォームごとの差異が大きい場合は template 化を別途検討する

---

### 9. Evaluate Form Response
IF ノードまたは Code ノードで成功判定を行う。

#### 成功判定の候補
- HTTP status code が 200 または 201
- `success_keyword` が response body に含まれる
- レスポンス JSON に success フラグがある

#### MVP v1 推奨
以下のどれかを満たせば success とする。

- status code が 200〜299
- かつ `success_keyword` 未設定
- または `success_keyword` が response body に含まれる

#### 注意
- HTTP 200 でも実際には失敗しているフォームがある
- success_keyword がある target の方が判定しやすい
- CAPTCHA や確認画面付きフォームはここで false になりやすい

---

### 10. Build Form Success Result
送信成功時の `channel_results` と `activity_logs` を生成する。

#### 出力例
```json
{
  "channel_result": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "channel": "form",
    "provider": "http_request",
    "send_status": "sent",
    "failure_reason": null
  },
  "activity_log": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "activity_type": "form_submitted",
    "activity_status": "success",
    "detail": "form submit completed"
  }
}
```

---

### 11. Build Form Failure Result
送信失敗時の `channel_results` と `activity_logs` を生成する。

#### 出力例
```json
{
  "channel_result": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "channel": "form",
    "provider": "http_request",
    "send_status": "failed",
    "failure_reason": "form_submit_failed"
  },
  "activity_log": {
    "execution_id": "={{$json.execution_id}}",
    "target_id": "={{$json.target_id}}",
    "activity_type": "form_submitted",
    "activity_status": "failed",
    "detail": "form submit failed"
  }
}
```

#### 補足
- `detail` に status code や error message を含めてもよい
- node error output を使う場合は `$json.error` の存在を明示判定する
- Continue on Fail は downstream で成功扱いに見えやすいので、明示的に failed 結果へ変換する

---

### 12. Aggregate Form Results
各 item の `channel_result` を `channel_results[]` に再集約する。

#### 出力イメージ
```json
{
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_010",
      "channel": "form",
      "provider": "http_request",
      "send_status": "sent",
      "failure_reason": null
    }
  ]
}
```

---

### 13. Aggregate Activity Logs
各 item の `activity_log` を `activity_logs[]` に再集約する。

#### 出力イメージ
```json
{
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_010",
      "activity_type": "form_submitted",
      "activity_status": "success",
      "detail": "form submit completed"
    }
  ]
}
```

---

### 14. Return Form Outreach Payload
最終返却形式を固定する。

#### 返却例
```json
{
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_010",
      "channel": "form",
      "provider": "http_request",
      "send_status": "sent",
      "failure_reason": null
    }
  ],
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_010",
      "activity_type": "form_submitted",
      "activity_status": "success",
      "detail": "form submit completed"
    }
  ]
}
```

---

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Form Targets Exists
    -> true  -> Split Form Targets
              -> Validate Form Target Fields
                   -> false -> Build Invalid Form Result
                   -> true  -> Build Form Request Payload
                             -> Submit Form Request
                             -> Evaluate Form Response
                                -> success -> Build Form Success Result
                                -> failure -> Build Form Failure Result
              -> Aggregate Form Results
              -> Aggregate Activity Logs
              -> Return Form Outreach Payload
    -> false -> Stop Missing Targets
```

---

## 実装上の注意

### 1. 対応フォームを最初に限定する
すべてのフォームに汎用対応しようとすると急激に複雑化する。
MVP v1 では「submit_url が明確で単純 POST できるもの」に絞る。

### 2. success 判定を曖昧にしない
HTTP 200 のみで sent 扱いにすると誤判定が起きることがある。
可能なら success keyword を持たせる。

### 3. Continue on Fail を使う場合は明示分岐を入れる
エラー出力をそのまま流すだけでは、あとで正常系と混ざって見えやすい。
failed 結果へ必ず変換する。

### 4. CAPTCHA と JS 必須フォームは MVP 外
これらを無理に通すと Browser Use や Playwright 系の別設計が必要になる。
MVP v1 では failed もしくは excluded 扱いに寄せる。

### 5. provider 名を固定する
`provider = http_request` に統一しておくと KPI 集計やログ分析がしやすい。

---

## 想定 activity_type

| activity_type | 意味 |
|---|---|
| `form_submitted` | form 送信実行 |
| `form_validation_failed` | 入力検証失敗 |

---

## 想定 failure_reason

| failure_reason | 意味 |
|---|---|
| `invalid_form_target_payload` | 必須項目不足 |
| `form_submit_failed` | form 送信失敗 |
| `unsupported_form_type` | 対応外フォーム |
| `missing_targets` | upstream から対象未受領 |

---

## 初期テストケース

### テスト1: 正常送信
```json
{
  "execution_id": "ex_001",
  "targets": [
    {
      "target_id": "t_010",
      "channel": "form",
      "submit_url": "https://example.com/contact/submit",
      "http_method": "POST",
      "form_payload": {
        "name": "Yamada",
        "email": "test@example.com",
        "message": "お問い合わせ本文です。"
      },
      "success_keyword": "ありがとうございました"
    }
  ]
}
```

### 期待結果
- HTTP Request 実行成功
- `channel_results[0].send_status = sent`
- `activity_logs[0].activity_status = success`

---

### テスト2: submit_url 欠損
```json
{
  "execution_id": "ex_002",
  "targets": [
    {
      "target_id": "t_011",
      "channel": "form",
      "form_payload": {
        "name": "Yamada",
        "message": "本文"
      }
    }
  ]
}
```

### 期待結果
- `send_status = invalid`
- `failure_reason = invalid_form_target_payload`

---

### テスト3: HTTP 失敗
```json
{
  "execution_id": "ex_003",
  "targets": [
    {
      "target_id": "t_012",
      "channel": "form",
      "submit_url": "https://example.com/invalid",
      "http_method": "POST",
      "form_payload": {
        "name": "Yamada",
        "message": "本文"
      }
    }
  ]
}
```

### 期待結果
- `send_status = failed`
- `failure_reason = form_submit_failed`

---

### テスト4: CAPTCHA 付きフォーム
```json
{
  "execution_id": "ex_004",
  "targets": [
    {
      "target_id": "t_013",
      "channel": "form",
      "submit_url": "https://example.com/captcha",
      "http_method": "POST",
      "form_payload": {
        "name": "Yamada",
        "message": "本文"
      }
    }
  ]
}
```

### 期待結果
- `send_status = failed` または `excluded`
- `failure_reason = unsupported_form_type` を検討する

---

## MVP v1 時点の割り切り

- multi-step form は対応しない
- CAPTCHA は対応しない
- JavaScript 実行前提フォームは対応しない
- hidden token の動的取得は対応しない
- Cookie セッション維持は高度には扱わない
- 送信後確認画面の再遷移処理は行わない
- Browser Use 連携は別 workflow で扱う

---

## 親 workflow への返却前提

この workflow の出力は、親 workflow または後続の
`sub-channel-result-updater` と `sub-log-output-router` がそのまま受け取れる形にする。

### 前提出力
- `channel_results[]`
- `activity_logs[]`

これにより、form 送信処理、結果更新、ログ保存を疎結合で接続できる。
