# sub-form-outreach 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の sub-workflow `sub-form-outreach` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、`sub-channel-router` が返した `form_targets` を受け取り、フォーム送信前整形、送信 payload 生成、フォーム送信実行、送信結果ログ生成までを一貫して処理することである。

MVP v1 では、フォーム送信の実行方式を HTTP Request または Browser Use / ブラウザ自動化の差し替え可能構成として設計し、まずは標準的な問い合わせフォームへの POST 実行またはダミー送信で end-to-end を通すことを優先する。

---

## 役割

- `form_targets` を受け取る。
- 送信前の最低限の入力確認を行う。
- フォーム送信用 payload を生成する。
- HTTP Request またはブラウザ自動化で送信を試行する。
- 成否に応じて `channel_results` と `activity_logs` を返す。
- 親 workflow が email 側と同じ形で扱える返却形式にそろえる。

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger` を使う。
- 初期は `Accept all data` で受ける。
- item 単位で送信処理する。
- 最初は「固定項目を持つシンプルフォーム」だけを対象にしてよい。
- 1件失敗で workflow 全体を止めず、失敗を結果として返す設計を優先する。
- CAPTCHA、複雑な multi-step form、JS依存フォームは MVP v1 では対象外にする。
- この workflow では KPI 集計を行わず、送信結果の生成までに責務を限定する。

---

## 入力仕様

### 想定入力
```json
{
  "execution_id": "ex_001",
  "form_targets": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "form_url": "https://example.com/contact",
      "channel": "form",
      "route_status": "routed"
    }
  ]
}
```

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Form Targets Exists | IF | `form_targets` の存在確認 |
| 3 | Stop Missing Form Targets | Stop And Error | 必須入力不足時に停止 |
| 4 | Split Form Targets | Split Out / Item Lists | 送信対象を1件ずつ処理 |
| 5 | Validate Sendable Form Item | IF | `form_url` と `target_id` の確認 |
| 6 | Build Invalid Form Result | Set | 不正 item の失敗結果生成 |
| 7 | Build Form Draft Payload | Set | フォーム送信用 payload の素案生成 |
| 8 | Build Form Request Body | Set | HTTP送信用 body 整形 |
| 9 | Submit Form | HTTP Request / Browser Use | フォーム送信実行 |
| 10 | Check Submit Result | IF | 送信成功判定 |
| 11 | Build Success Channel Result | Set | 成功時の `channel_results` 生成 |
| 12 | Build Success Activity Log | Set | 成功時の `activity_logs` 生成 |
| 13 | Build Failed Channel Result | Set | 失敗時の `channel_results` 生成 |
| 14 | Build Failed Activity Log | Set | 失敗時の `activity_logs` 生成 |
| 15 | Aggregate Channel Results | Aggregate / Item Lists | `channel_results` 配列へ集約 |
| 16 | Aggregate Activity Logs | Aggregate / Item Lists | `activity_logs` 配列へ集約 |
| 17 | Return Form Outreach Payload | Set | 最終返却形式へ整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノードとする。

#### Input data mode
- 初期は `Accept all data`

#### 理由
- parent 側からの入力項目を柔軟に受ける。
- 実データを流しながら設計を固めやすい。
- 後で `Define using JSON example` に厳格化できる。

---

### 2. Validate Form Targets Exists
IF ノードで `form_targets` が存在し、かつ 1 件以上あるかを判定する。

#### true
- `Split Form Targets` へ進む。

#### false
- `Stop Missing Form Targets` へ進む。

---

### 3. Stop Missing Form Targets
Stop And Error ノードで停止する。

#### メッセージ例
```text
sub-form-outreach: form_targets is missing or empty.
```

#### 意図
- parent または channel-router 側の異常を早期検知する。
- error workflow に流せるようにする。

---

### 4. Split Form Targets
`form_targets` 配列を1件ずつ展開する。

#### 出力イメージ
```json
{
  "execution_id": "ex_001",
  "target_id": "t_001",
  "company_name": "Example Inc.",
  "form_url": "https://example.com/contact",
  "channel": "form",
  "route_status": "routed"
}
```

---

### 5. Validate Sendable Form Item
IF ノードで最低限の送信条件を確認する。

#### 条件
- `target_id` が存在する。
- `form_url` が存在する。
- `channel == form` である。

#### true
- `Build Form Draft Payload` へ進む。

#### false
- `Build Invalid Form Result` へ進む。

---

### 6. Build Invalid Form Result
Set ノードで不正 item 用の失敗結果を生成する。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id || null}}",
  "channel": "form",
  "send_status": "failed",
  "failure_reason": "invalid_form_target_payload",
  "provider": "http_request"
}
```

#### 補足
- 初期 provider 名は `http_request` としてよい。
- Browser Use に切り替える場合は provider を `browser_use` に変える。

---

### 7. Build Form Draft Payload
Set ノードで送信内容の素案を整える。

#### 目的
- 後段の送信ノードが受け取りやすい形にする。
- 将来的に AI による本文生成へ差し替えやすくする。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "form_url": "={{$json.form_url}}",
  "sender_name": "Luvira",
  "sender_email": "contact@luvira.example",
  "subject": "={{'営業・業務自動化のご相談'}}",
  "message": "={{$json.company_name + '様\\n\\n営業・業務自動化のご支援可能性についてご相談したくご連絡しました。短くご紹介の機会をいただけますと幸いです。'}}"
}
```

---

### 8. Build Form Request Body
Set ノードで HTTP Request 用の request body を生成する。

#### 例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "form_url": "={{$json.form_url}}",
  "request_body": {
    "company": "Luvira",
    "name": "Luvira",
    "email": "contact@luvira.example",
    "subject": "={{$json.subject}}",
    "message": "={{$json.message}}"
  }
}
```

#### 注意
- 実際のフォーム項目名はサイトごとに違う。
- MVP v1 では「固定の問い合わせ項目名を持つフォームだけ成功すればよい」という割り切りで始める。
- 本番では site-specific mapper が別に必要になる可能性が高い。

---

### 9. Submit Form
HTTP Request ノードまたは Browser Use 連携ノードでフォーム送信する。

#### パターンA: HTTP Request
- Method: `POST`
- URL: `={{$json.form_url}}`
- Body: `={{$json.request_body}}`

#### パターンB: Browser Use
- GUI 操作が必要な場合は Browser Use 側 workflow へ委譲する。
- Browser Use 利用時も返却形式は同じにそろえる。

#### エラー方針
- 1件失敗で全体停止しないように node 側 error handling を使う。
- `Continue On Fail` または error output を検討する。

---

### 10. Check Submit Result
IF ノードで送信成功かどうかを判定する。

#### 判定例
- HTTP status code が 200〜299
- または Browser Use の戻り値が `success = true`

#### true
- `Build Success Channel Result` へ進む。

#### false
- `Build Failed Channel Result` へ進む。

#### 注意
- 「HTTP 200 = 本当に送信成功」とは限らない。
- ただし MVP v1 では暫定的に success 判定としてよい。

---

### 11. Build Success Channel Result
送信成功時の `channel_results` を作る。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "channel": "form",
  "provider": "http_request",
  "send_status": "sent",
  "failure_reason": null
}
```

---

### 12. Build Success Activity Log
送信成功時の `activity_logs` を作る。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "activity_type": "form_sent",
  "activity_status": "success",
  "detail": "form submit completed"
}
```

---

### 13. Build Failed Channel Result
送信失敗時の `channel_results` を作る。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "channel": "form",
  "provider": "http_request",
  "send_status": "failed",
  "failure_reason": "form_submit_failed"
}
```

#### 補足
- CAPTCHA や入力項目不一致、JSフォーム非対応なども最初はこの failure_reason にまとめてよい。
- 後で reason taxonomy を分ける。

---

### 14. Build Failed Activity Log
送信失敗時の `activity_logs` を作る。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "activity_type": "form_sent",
  "activity_status": "failed",
  "detail": "form submit failed"
}
```

---

### 15. Aggregate Channel Results
結果を `channel_results` 配列へ集約する。

#### ポイント
- 成功・失敗・invalid を同じ配列に載せる。
- 後続 updater が channel 共通の形で扱えるようにする。

---

### 16. Aggregate Activity Logs
ログを `activity_logs` 配列へ集約する。

#### ポイント
- email 側と同一スキーマを意識する。
- 後で Sheets や DB へ出力しやすくなる。

---

### 17. Return Form Outreach Payload
最終返却形式を固定する。

#### 返却例
```json
{
  "execution_id": "ex_001",
  "channel_results": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "channel": "form",
      "provider": "http_request",
      "send_status": "sent",
      "failure_reason": null
    }
  ],
  "activity_logs": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "activity_type": "form_sent",
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
              -> Validate Sendable Form Item
                   -> false -> Build Invalid Form Result
                   -> true  -> Build Form Draft Payload
                             -> Build Form Request Body
                             -> Submit Form
                             -> Check Submit Result
                                -> true  -> Build Success Channel Result
                                          -> Build Success Activity Log
                                -> false -> Build Failed Channel Result
                                          -> Build Failed Activity Log
              -> Aggregate Channel Results
              -> Aggregate Activity Logs
              -> Return Form Outreach Payload
    -> false -> Stop Missing Form Targets
```

---

## 実装上の注意

### 1. 最初から万能フォーム対応を目指さない
フォームはサイトごとの差が大きいので、MVP v1 では「通るものだけ通す」思想でよい。

### 2. HTTP送信とブラウザ送信を抽象化する
provider を明示しておくと、後で Browser Use に切り替えても downstream を変えずに済む。

### 3. success 判定は暫定でよい
本来は confirmation text や redirect 先確認まで見たいが、MVP v1 では HTTP 成否や automation 成否で十分。

### 4. 不正 item は送信前に落とす
`target_id` や `form_url` 欠落は送信失敗ではなく入力不正として扱う方が分析しやすい。

---

## 初期テストケース

### テスト1: 正常送信
```json
{
  "execution_id": "ex_001",
  "form_targets": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "form_url": "https://example.com/contact",
      "channel": "form",
      "route_status": "routed"
    }
  ]
}
```

### 期待結果
- `channel_results[0].send_status = sent`
- `activity_logs[0].activity_status = success`

---

### テスト2: form_url 欠落
```json
{
  "execution_id": "ex_002",
  "form_targets": [
    {
      "execution_id": "ex_002",
      "target_id": "t_002",
      "company_name": "Broken Form KK",
      "form_url": null,
      "channel": "form",
      "route_status": "routed"
    }
  ]
}
```

### 期待結果
- invalid item として `send_status = failed`
- `failure_reason = invalid_form_target_payload`

---

### テスト3: HTTP送信失敗
```json
{
  "execution_id": "ex_003",
  "form_targets": [
    {
      "execution_id": "ex_003",
      "target_id": "t_003",
      "company_name": "Reject Form KK",
      "form_url": "https://reject.example/contact",
      "channel": "form",
      "route_status": "routed"
    }
  ]
}
```

### 期待結果
- `send_status = failed`
- `failure_reason = form_submit_failed`

---

## MVP v1 時点の割り切り

- CAPTCHA 対応は行わない。
- multi-step form は扱わない。
- hidden field / CSRF の高度対応は行わない。
- サイト別フォームマッピングは行わない。
- 送信後の自動返信メール確認は行わない。
- Browser Use は必要時のみ差し替える。

---

## 次の sub-workflow への受け渡し前提

この workflow の出力は、親 workflow の `Merge Channel Results` と `sub-channel-result-updater` がそのまま受け取れる形にする。

### 前提出力
- `execution_id`
- `channel_results[]`
- `activity_logs[]`

この形に固定することで、email 側と form 側を同じ merge / update ロジックで扱える。
