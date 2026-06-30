# sub-target-validator 実ノード構成案 v1

## 概要

この文書は、luvira-outreach-os の sub-workflow `sub-target-validator` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、正規化済み target を受け取り、
実行対象として有効かどうかを判定し、`active`、`excluded`、`invalid` などの標準 status を付与して返すことである。

MVP v1 では、複雑なスコアリングや高度な重複排除ではなく、
最低限の実行可否判定に責務を限定する。

---

## 役割

- 正規化済み target を受け取る
- 必須項目の存在を確認する
- email / form の実行可否を判定する
- target に `normalized_status` を付与する
- 必要に応じて `failure_reason` や `exclusion_reason` を付与する
- 親 workflow や次段 sub-workflow に渡しやすい shape で返す

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger` を使う
- 初期は `Accept all data` で受ける
- 入力は `targets[]` を基本とする
- 1 target ごとに独立判定する
- 判定結果は item に直接書き戻す
- 判定ロジックは IF ノードまたは Code ノードで構成する
- MVP v1 では結果を返すだけにして、保存は行わない
- 複雑な dedupe や blacklist 照合は将来拡張に回す

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
      "company_name": "Example Inc.",
      "contact_name": "Yamada",
      "email": "test@example.com",
      "submit_url": null,
      "channel_hint": "email",
      "body_text": "本文"
    },
    {
      "target_id": "t_002",
      "company_name": "Form Co.",
      "contact_name": "Suzuki",
      "email": null,
      "submit_url": "https://example.com/contact/submit",
      "channel_hint": "form",
      "form_payload": {
        "name": "Suzuki",
        "message": "お問い合わせ本文"
      }
    }
  ]
}
```

### 前提
- `targets[]` は `sub-csv-normalizer` を通過した後の標準 shape を想定する
- `target_id` は原則付与済み
- `channel_hint` は任意だが、あれば判定に利用してよい

---

## 出力仕様

### 返却するデータ
- `targets[]`

### 返却イメージ
```json
{
  "targets": [
    {
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "email": "test@example.com",
      "submit_url": null,
      "channel_hint": "email",
      "normalized_status": "active",
      "validated_channel": "email",
      "failure_reason": null,
      "exclusion_reason": null
    },
    {
      "target_id": "t_002",
      "company_name": "Form Co.",
      "email": null,
      "submit_url": "https://example.com/contact/submit",
      "channel_hint": "form",
      "normalized_status": "active",
      "validated_channel": "form",
      "failure_reason": null,
      "exclusion_reason": null
    }
  ]
}
```

---

## 判定ルール

### active
実行対象として有効なもの。

#### 例
- email があり、email 送信可能
- submit_url と form_payload があり、form 送信可能

### excluded
入力としては正しいが、MVP v1 では送信対象外にするもの。

#### 例
- channel_hint が `exclude`
- CAPTCHA 必須フォーム
- JS 実行前提フォーム
- 送信先情報はあるが運用上除外したい

### invalid
必須情報不足や shape 不正で、実行対象にできないもの。

#### 例
- email も submit_url もない
- target_id がない
- form 用なのに form_payload がない

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Targets Exists | IF | `targets` の存在確認 |
| 3 | Stop Missing Targets | Stop And Error | 入力不足時に停止 |
| 4 | Split Targets | Split Out / Item Lists | target を1件ずつ展開 |
| 5 | Validate Base Fields | IF | `target_id` など最低限確認 |
| 6 | Build Invalid Base Result | Set | 基本不正 target の結果化 |
| 7 | Evaluate Email Eligibility | IF | email 実行可否判定 |
| 8 | Evaluate Form Eligibility | IF | form 実行可否判定 |
| 9 | Evaluate Explicit Exclusion | IF | 明示除外判定 |
| 10 | Build Active Email Target | Set | active/email 付与 |
| 11 | Build Active Form Target | Set | active/form 付与 |
| 12 | Build Excluded Target | Set | excluded 付与 |
| 13 | Build Invalid Target | Set | invalid 付与 |
| 14 | Aggregate Validated Targets | Aggregate / Item Lists | `targets[]` に再集約 |
| 15 | Return Validated Targets Payload | Set | 最終返却形式に整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノードとする。

#### Input data mode
- 初期は `Accept all data`

---

### 2. Validate Targets Exists
IF ノードで `targets` が存在し、かつ 1 件以上あるかを判定する。

#### 条件
- `targets` が存在する
- `targets.length > 0`

#### true
- `Split Targets` へ進む

#### false
- `Stop Missing Targets` へ進む

---

### 3. Stop Missing Targets
Stop And Error ノードで停止する。

#### メッセージ例
```text
sub-target-validator: targets is missing or empty.
```

---

### 4. Split Targets
`targets[]` を1件ずつ展開する。

#### 出力イメージ
```json
{
  "execution_id": "ex_001",
  "customer_id": "cust_001",
  "source_list_id": "sl_001",
  "target_id": "t_001",
  "company_name": "Example Inc.",
  "contact_name": "Yamada",
  "email": "test@example.com",
  "submit_url": null,
  "channel_hint": "email",
  "body_text": "本文"
}
```

---

### 5. Validate Base Fields
IF ノードで最低限の field を確認する。

#### 条件
- `target_id` が存在する

#### true
- 次の判定へ進む

#### false
- `Build Invalid Base Result` へ進む

#### 補足
- `company_name` や `contact_name` は MVP v1 では必須にしない
- `target_id` は downstream の key になるため必須

---

### 6. Build Invalid Base Result
基本 shape が不正な target を invalid 化する。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": null,
  "company_name": "={{$json.company_name || null}}",
  "email": "={{$json.email || null}}",
  "submit_url": "={{$json.submit_url || null}}",
  "channel_hint": "={{$json.channel_hint || null}}",
  "normalized_status": "invalid",
  "validated_channel": null,
  "failure_reason": "missing_target_id",
  "exclusion_reason": null
}
```

---

### 7. Evaluate Email Eligibility
email 実行可否を判定する IF ノード。

#### 条件例
- `email` が存在する
- `email` が空でない
- `channel_hint != 'form'` でもよい、または hint を無視して email 優先でもよい

#### true
- `Build Active Email Target` へ進む

#### false
- 次の判定へ進む

#### 補足
- MVP v1 では email があるものを最優先に email 扱いしてもよい
- ただし channel_hint を優先する設計でもよい
- 運用で揺れないよう、どちらを優先するか文書に固定する

---

### 8. Evaluate Form Eligibility
form 実行可否を判定する IF ノード。

#### 条件例
- `submit_url` が存在する
- `form_payload` が存在する

#### true
- `Build Active Form Target` へ進む

#### false
- 次の判定へ進む

---

### 9. Evaluate Explicit Exclusion
明示除外や対応外フォームを excluded とする IF ノード。

#### 条件例
- `channel_hint == 'exclude'`
- `form_type == 'captcha'`
- `requires_browser == true`

#### true
- `Build Excluded Target` へ進む

#### false
- `Build Invalid Target` へ進む

#### 補足
- 明示除外より前に email / form 判定を置くか、逆にするかは設計次第
- 「運用上送らない」が優先なら exclusion 判定を先に置いてもよい

---

### 10. Build Active Email Target
email 実行対象として返す。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name || null}}",
  "contact_name": "={{$json.contact_name || null}}",
  "email": "={{$json.email}}",
  "submit_url": "={{$json.submit_url || null}}",
  "channel_hint": "={{$json.channel_hint || null}}",
  "body_text": "={{$json.body_text || null}}",
  "normalized_status": "active",
  "validated_channel": "email",
  "failure_reason": null,
  "exclusion_reason": null
}
```

---

### 11. Build Active Form Target
form 実行対象として返す。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name || null}}",
  "contact_name": "={{$json.contact_name || null}}",
  "email": "={{$json.email || null}}",
  "submit_url": "={{$json.submit_url}}",
  "channel_hint": "={{$json.channel_hint || null}}",
  "form_payload": "={{$json.form_payload || null}}",
  "normalized_status": "active",
  "validated_channel": "form",
  "failure_reason": null,
  "exclusion_reason": null
}
```

---

### 12. Build Excluded Target
明示的に除外対象として返す。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name || null}}",
  "email": "={{$json.email || null}}",
  "submit_url": "={{$json.submit_url || null}}",
  "channel_hint": "={{$json.channel_hint || null}}",
  "normalized_status": "excluded",
  "validated_channel": null,
  "failure_reason": null,
  "exclusion_reason": "explicitly_excluded"
}
```

---

### 13. Build Invalid Target
実行先が確定できない target を invalid として返す。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name || null}}",
  "email": "={{$json.email || null}}",
  "submit_url": "={{$json.submit_url || null}}",
  "channel_hint": "={{$json.channel_hint || null}}",
  "normalized_status": "invalid",
  "validated_channel": null,
  "failure_reason": "no_supported_channel",
  "exclusion_reason": null
}
```

---

### 14. Aggregate Validated Targets
各 item を `targets[]` に再集約する。

#### 出力イメージ
```json
{
  "targets": [
    {
      "target_id": "t_001",
      "normalized_status": "active",
      "validated_channel": "email",
      "failure_reason": null,
      "exclusion_reason": null
    },
    {
      "target_id": "t_002",
      "normalized_status": "active",
      "validated_channel": "form",
      "failure_reason": null,
      "exclusion_reason": null
    }
  ]
}
```

---

### 15. Return Validated Targets Payload
最終返却形式を固定する。

#### 返却例
```json
{
  "targets": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "email": "test@example.com",
      "submit_url": null,
      "channel_hint": "email",
      "normalized_status": "active",
      "validated_channel": "email",
      "failure_reason": null,
      "exclusion_reason": null
    },
    {
      "execution_id": "ex_001",
      "target_id": "t_002",
      "company_name": "Form Co.",
      "email": null,
      "submit_url": "https://example.com/contact/submit",
      "channel_hint": "form",
      "normalized_status": "active",
      "validated_channel": "form",
      "failure_reason": null,
      "exclusion_reason": null
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
              -> Validate Base Fields
                   -> false -> Build Invalid Base Result
                   -> true  -> Evaluate Email Eligibility
                             -> true  -> Build Active Email Target
                             -> false -> Evaluate Form Eligibility
                                        -> true  -> Build Active Form Target
                                        -> false -> Evaluate Explicit Exclusion
                                                   -> true  -> Build Excluded Target
                                                   -> false -> Build Invalid Target
              -> Aggregate Validated Targets
              -> Return Validated Targets Payload
    -> false -> Stop Missing Targets
```

---

## 実装上の注意

### 1. validation と routing を混ぜすぎない
ここでは「実行可能かどうか」の判定までに寄せる。
email / form の配列分割は `sub-channel-router` に任せる。

### 2. email 優先か hint 優先かを固定する
email と submit_url の両方がある target がありうる。
どちらを優先するかを決めて文書化しておく。

### 3. excluded と invalid を分ける
- excluded: 意図的に送らない
- invalid: そもそも送れない

この違いを残すと、KPI や運用判断がしやすい。

### 4. failure_reason と exclusion_reason を分ける
理由を 1 つの field に混ぜると downstream 集計が見づらくなる。

### 5. 保存はここでやらない
この workflow は判定結果を返すだけにし、ログ保存は後段に任せる。

---

## 想定 normalized_status

| normalized_status | 意味 |
|---|---|
| `active` | 実行対象 |
| `excluded` | 運用上の除外対象 |
| `invalid` | 入力不正または実行不能 |

---

## 想定 failure_reason

| failure_reason | 意味 |
|---|---|
| `missing_target_id` | target_id 欠損 |
| `no_supported_channel` | email / form どちらでも送れない |
| `missing_form_payload` | form payload 欠損 |
| `missing_email` | email 欠損 |

---

## 想定 exclusion_reason

| exclusion_reason | 意味 |
|---|---|
| `explicitly_excluded` | 明示的除外 |
| `unsupported_form_type` | 対応外フォーム |
| `requires_browser_automation` | Browser Use など別設計が必要 |

---

## 初期テストケース

### テスト1: email target
```json
{
  "execution_id": "ex_001",
  "targets": [
    {
      "target_id": "t_001",
      "email": "test@example.com",
      "channel_hint": "email",
      "body_text": "本文"
    }
  ]
}
```

### 期待結果
- `normalized_status = active`
- `validated_channel = email`

---

### テスト2: form target
```json
{
  "execution_id": "ex_002",
  "targets": [
    {
      "target_id": "t_002",
      "submit_url": "https://example.com/contact/submit",
      "form_payload": {
        "name": "Yamada",
        "message": "本文"
      },
      "channel_hint": "form"
    }
  ]
}
```

### 期待結果
- `normalized_status = active`
- `validated_channel = form`

---

### テスト3: 明示除外
```json
{
  "execution_id": "ex_003",
  "targets": [
    {
      "target_id": "t_003",
      "channel_hint": "exclude"
    }
  ]
}
```

### 期待結果
- `normalized_status = excluded`
- `exclusion_reason = explicitly_excluded`

---

### テスト4: 無効 target
```json
{
  "execution_id": "ex_004",
  "targets": [
    {
      "target_id": "t_004",
      "email": null,
      "submit_url": null
    }
  ]
}
```

### 期待結果
- `normalized_status = invalid`
- `failure_reason = no_supported_channel`

---

## MVP v1 時点の割り切り

- MX チェックは行わない
- email deliverability 判定は行わない
- form 到達性の事前確認は行わない
- blacklist / denylist 照合は行わない
- 顧客別例外ルールは行わない
- 重複排除は高度には行わない
- score ベースの優先順位付けは行わない

---

## 親 workflow への返却前提

この workflow の出力は、後続の `sub-channel-router` がそのまま受け取れる形にする。

### 前提出力
- `targets[]`

これにより、判定と routing を疎結合に保てる。
