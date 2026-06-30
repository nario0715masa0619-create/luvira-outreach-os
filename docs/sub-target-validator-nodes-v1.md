# sub-target-validator 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の sub-workflow `sub-target-validator` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、`sub-csv-normalizer` が返した `targets` 配列を受け取り、
営業対象として利用可能かどうかを判定し、後続 workflow が使えるように eligibility と status を付与することである。

MVP v1 では、高度なデータクレンジングではなく、営業実行可否に必要な最低限の判定に集中する。

---

## 役割

- normalized targets を受け取る
- company_name の有無を確認する
- email / form の利用可否を判定する
- eligibility フラグを付ける
- normalized_status を更新する
- exclusion_reason を付ける
- 後続の channel-router が使いやすい形で返す

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger`
- 初期は `Accept all data` で受ける
- まずは `targets` 配列の存在だけを厳格に見る
- 会社名がないものは原則 `invalid`
- email, form は「存在するか」を MVP の主判定にする
- ここでは送信実行しない
- ここでは配信到達性やフォーム送信成功可否までは見ない
- raw data には戻らず標準 target schema のみ扱う

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Targets Exists | IF | targets 配列の存在確認 |
| 3 | Stop Missing Targets | Stop And Error | targets 欠落時に停止 |
| 4 | Split Targets | Split Out / Item Lists | target を1件ずつ処理 |
| 5 | Validate Company Name | IF | company_name 有無判定 |
| 6 | Mark Invalid No Company | Set | company_name 欠落の invalid 化 |
| 7 | Check Email Exists | IF | contact_email の有無判定 |
| 8 | Set Email Eligibility True | Set | email 可フラグ付与 |
| 9 | Set Email Eligibility False | Set | email 不可フラグ付与 |
| 10 | Check Form Exists | IF | form_url の有無判定 |
| 11 | Set Form Eligibility True | Set | form 可フラグ付与 |
| 12 | Set Form Eligibility False | Set | form 不可フラグ付与 |
| 13 | Set Final Target Status | Set | status / exclusion_reason 整理 |
| 14 | Aggregate Validated Targets | Aggregate / Item Lists | 配列へ再集約 |
| 15 | Return Validated Targets Payload | Set | 最終返却形式へ整形 |

---

## 入力仕様

### 想定入力
```json
{
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_sl_001_0",
      "source_list_id": "sl_001",
      "source_row_id": "row_0",
      "company_name": "Example Inc.",
      "website_url": "https://example.com",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact",
      "phone": null,
      "address": null,
      "industry": null,
      "normalized_status": "active"
    },
    {
      "target_id": "t_sl_001_1",
      "source_list_id": "sl_001",
      "source_row_id": "row_1",
      "company_name": null,
      "website_url": "https://demo.jp",
      "contact_email": null,
      "form_url": "https://demo.jp/contact",
      "phone": null,
      "address": null,
      "industry": null,
      "normalized_status": "active"
    }
  ]
}
```

---

## 判定ルール

### 1. company_name
- null または空なら `invalid`
- 値があれば次へ進む

### 2. contact_email
- 値があれば `channel_eligibility_email = true`
- 値がなければ `channel_eligibility_email = false`

### 3. form_url
- 値があれば `channel_eligibility_form = true`
- 値がなければ `channel_eligibility_form = false`

### 4. normalized_status
- company_name がない場合は `invalid`
- company_name があり、email / form の両方不可なら `excluded`
- company_name があり、どちらか利用可能なら `active`

### 5. exclusion_reason
- company_name 不足 → `missing_company_name`
- email / form 両方なし → `no_available_channel`
- active の場合 → null

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノード。

#### Input data mode
- 初期実装では `Accept all data`

#### 理由
- parent 側の input を柔軟に受ける
- 初期は入力スキーマ変更に強くしておく
- 実装が安定したら JSON example へ切り替え可能

---

### 2. Validate Targets Exists
IF ノード。

#### 条件
- `targets` が存在する
- `targets.length > 0`

#### true
- 次へ進む

#### false
- `Stop Missing Targets` へ進む

---

### 3. Stop Missing Targets
Stop And Error ノード。

#### エラーメッセージ例
```text
sub-target-validator: targets is missing or empty.
```

#### 意図
- parent または前段 sub-workflow の異常を早期検知する
- error-handler workflow で収集しやすくする

---

### 4. Split Targets
`targets` 配列を1件ずつ展開する。

#### 出力イメージ
```json
[
  {
    "target_id": "t_sl_001_0",
    "source_list_id": "sl_001",
    "source_row_id": "row_0",
    "company_name": "Example Inc.",
    "website_url": "https://example.com",
    "contact_email": "info@example.com",
    "form_url": "https://example.com/contact",
    "normalized_status": "active"
  },
  {
    "target_id": "t_sl_001_1",
    "source_list_id": "sl_001",
    "source_row_id": "row_1",
    "company_name": null,
    "website_url": "https://demo.jp",
    "contact_email": null,
    "form_url": "https://demo.jp/contact",
    "normalized_status": "active"
  }
]
```

---

### 5. Validate Company Name
IF ノード。

#### 条件
- `company_name` が存在する
- `company_name` が空文字でない

#### true
- email / form 判定へ進む

#### false
- `Mark Invalid No Company` へ進む

---

### 6. Mark Invalid No Company
Set ノード。

#### 設定内容
```json
{
  "target_id": "={{$json.target_id}}",
  "source_list_id": "={{$json.source_list_id}}",
  "source_row_id": "={{$json.source_row_id}}",
  "company_name": "={{$json.company_name}}",
  "website_url": "={{$json.website_url}}",
  "contact_email": "={{$json.contact_email}}",
  "form_url": "={{$json.form_url}}",
  "phone": "={{$json.phone}}",
  "address": "={{$json.address}}",
  "industry": "={{$json.industry}}",
  "channel_eligibility_email": false,
  "channel_eligibility_form": false,
  "normalized_status": "invalid",
  "exclusion_reason": "missing_company_name"
}
```

#### ポイント
- この分岐では以降の email / form 判定は不要
- ただし target 形式はそろえて返す

---

### 7. Check Email Exists
IF ノード。

#### 条件
- `contact_email` が存在する
- `contact_email` が空でない

#### true
- `Set Email Eligibility True`

#### false
- `Set Email Eligibility False`

---

### 8. Set Email Eligibility True
Set ノード。

#### 設定内容
```json
{
  "channel_eligibility_email": true
}
```

---

### 9. Set Email Eligibility False
Set ノード。

#### 設定内容
```json
{
  "channel_eligibility_email": false
}
```

---

### 10. Check Form Exists
IF ノード。

#### 条件
- `form_url` が存在する
- `form_url` が空でない

#### true
- `Set Form Eligibility True`

#### false
- `Set Form Eligibility False`

---

### 11. Set Form Eligibility True
Set ノード。

#### 設定内容
```json
{
  "channel_eligibility_form": true
}
```

---

### 12. Set Form Eligibility False
Set ノード。

#### 設定内容
```json
{
  "channel_eligibility_form": false
}
```

---

### 13. Set Final Target Status
email / form eligibility の結果を受けて最終状態を付与する Set ノード。

#### ロジック
- company_name なし → `invalid`
- company_name あり、email / form の両方 false → `excluded`
- company_name あり、どちらか true → `active`

#### JSON Output 例
```json
{
  "target_id": "={{$json.target_id}}",
  "source_list_id": "={{$json.source_list_id}}",
  "source_row_id": "={{$json.source_row_id}}",
  "company_name": "={{$json.company_name}}",
  "website_url": "={{$json.website_url}}",
  "contact_email": "={{$json.contact_email}}",
  "form_url": "={{$json.form_url}}",
  "phone": "={{$json.phone}}",
  "address": "={{$json.address}}",
  "industry": "={{$json.industry}}",
  "channel_eligibility_email": "={{$json.channel_eligibility_email}}",
  "channel_eligibility_form": "={{$json.channel_eligibility_form}}",
  "normalized_status": "={{(!$json.company_name) ? 'invalid' : (($json.channel_eligibility_email || $json.channel_eligibility_form) ? 'active' : 'excluded')}}",
  "exclusion_reason": "={{(!$json.company_name) ? 'missing_company_name' : ((!$json.channel_eligibility_email && !$json.channel_eligibility_form) ? 'no_available_channel' : null)}}"
}
```

---

### 14. Aggregate Validated Targets
個別 item を再度 `targets` 配列へまとめる。

#### 目的
- parent / 次の sub-workflow が配列として受け取れるようにする
- channel-router にそのまま渡せる構造にする

---

### 15. Return Validated Targets Payload
最終返却用 Set ノード。

#### 返却例
```json
{
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_sl_001_0",
      "source_list_id": "sl_001",
      "source_row_id": "row_0",
      "company_name": "Example Inc.",
      "website_url": "https://example.com",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact",
      "phone": null,
      "address": null,
      "industry": null,
      "channel_eligibility_email": true,
      "channel_eligibility_form": true,
      "normalized_status": "active",
      "exclusion_reason": null
    },
    {
      "target_id": "t_sl_001_1",
      "source_list_id": "sl_001",
      "source_row_id": "row_1",
      "company_name": null,
      "website_url": "https://demo.jp",
      "contact_email": null,
      "form_url": "https://demo.jp/contact",
      "phone": null,
      "address": null,
      "industry": null,
      "channel_eligibility_email": false,
      "channel_eligibility_form": false,
      "normalized_status": "invalid",
      "exclusion_reason": "missing_company_name"
    }
  ]
}
```

---

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Targets Exists
    -> true  -> Split Targets
              -> Validate Company Name
                   -> false -> Mark Invalid No Company
                   -> true  -> Check Email Exists
                                -> Set Email Eligibility True / False
                              -> Check Form Exists
                                -> Set Form Eligibility True / False
                              -> Set Final Target Status
              -> Aggregate Validated Targets
              -> Return Validated Targets Payload
    -> false -> Stop Missing Targets
```

---

## 実装上の注意

### 1. IF ノードを増やしすぎない
MVP v1 では、contact_email の厳密な RFC バリデーションや URL 到達確認まではやらない。
まずは「ある / ない」で十分。

### 2. invalid と excluded を分ける
- invalid：入力データとして壊れている
- excluded：入力はあるが営業対象として使えない

この区別は後で KPI やログの意味を明確にする。

### 3. ここで送信可否を確定しすぎない
たとえば form_url があっても実際に送れるかは別問題。
この sub-workflow では「候補として使えるか」までに留める。

### 4. error workflow 連携
この sub-workflow 自体に異常があった場合は、親 workflow の error workflow で拾う前提にする。
sub-workflow 単独運用する場合は、必要に応じて個別に error workflow を設定してもよい。

---

## 初期テストケース

### テスト1: email も form もある
```json
{
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_001",
      "source_list_id": "sl_001",
      "source_row_id": "row_0",
      "company_name": "Example Inc.",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact"
    }
  ]
}
```

### 期待結果
- `channel_eligibility_email = true`
- `channel_eligibility_form = true`
- `normalized_status = active`

---

### テスト2: form だけある
```json
{
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_002",
      "source_list_id": "sl_001",
      "source_row_id": "row_1",
      "company_name": "Demo LLC",
      "contact_email": null,
      "form_url": "https://demo.jp/contact"
    }
  ]
}
```

### 期待結果
- `channel_eligibility_email = false`
- `channel_eligibility_form = true`
- `normalized_status = active`

---

### テスト3: company_name がない
```json
{
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_003",
      "source_list_id": "sl_001",
      "source_row_id": "row_2",
      "company_name": null,
      "contact_email": "info@unknown.jp",
      "form_url": "https://unknown.jp/contact"
    }
  ]
}
```

### 期待結果
- `normalized_status = invalid`
- `exclusion_reason = missing_company_name`

---

### テスト4: company はあるが channel がない
```json
{
  "source_list_id": "sl_001",
  "targets": [
    {
      "target_id": "t_004",
      "source_list_id": "sl_001",
      "source_row_id": "row_3",
      "company_name": "No Contact KK",
      "contact_email": null,
      "form_url": null
    }
  ]
}
```

### 期待結果
- `channel_eligibility_email = false`
- `channel_eligibility_form = false`
- `normalized_status = excluded`
- `exclusion_reason = no_available_channel`

---

## MVP v1 時点の割り切り

- email 形式の厳密チェックは行わない
- URL の正常性確認は行わない
- MX確認や到達確認は行わない
- company 名寄せは行わない
- ブラックリスト判定は行わない
- NG業種除外などの高度ルールは customer settings 側へ将来分離する

---

## 次の sub-workflow への受け渡し前提
この workflow の出力は、次の `sub-channel-router` がそのまま受け取れる形にする。

### 前提出力
- `source_list_id`
- `targets[]`
- 各 target に以下を含む
  - `channel_eligibility_email`
  - `channel_eligibility_form`
  - `normalized_status`
  - `exclusion_reason`

これにより、channel-router 側では eligibility と run_mode だけを見れば分岐できる。
