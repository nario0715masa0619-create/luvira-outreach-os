# sub-channel-router 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の sub-workflow `sub-channel-router` を n8n 上で実装するための実ノード構成案である。

この sub-workflow の目的は、`sub-target-validator` が返した validated targets と `run_mode` を受け取り、
各 target を email / form / both / excluded のどこへ流すかを決定することである。

MVP v1 では、複雑な重みづけやスコアリングではなく、`run_mode` と eligibility に基づく単純で説明可能なルーティングを採用する。

---

## 役割

- validated targets を受け取る
- run_mode を受け取る
- target ごとに送信チャネルを決定する
- email_targets / form_targets / excluded_targets を返す
- 後続 workflow が迷わず分岐できる形を作る

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger`
- 初期は `Accept all data` で受ける
- ルーティング判定は target 単位で行う
- `normalized_status != active` は routing 対象外
- `run_mode` は `email` / `form` / `both` の3択
- `both` は「両チャネルへ流してよい」ことを意味する
- ここでは送信順制御やレート制御は行わない
- ここでは送信実行せず、配列振り分けだけを責務とする

---

## 入力仕様

### 想定入力
```json
{
  "execution_id": "ex_001",
  "run_mode": "both",
  "targets": [
    {
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact",
      "channel_eligibility_email": true,
      "channel_eligibility_form": true,
      "normalized_status": "active",
      "exclusion_reason": null
    },
    {
      "target_id": "t_002",
      "company_name": "Mail Only KK",
      "contact_email": "sales@mailonly.jp",
      "form_url": null,
      "channel_eligibility_email": true,
      "channel_eligibility_form": false,
      "normalized_status": "active",
      "exclusion_reason": null
    },
    {
      "target_id": "t_003",
      "company_name": "No Contact KK",
      "contact_email": null,
      "form_url": null,
      "channel_eligibility_email": false,
      "channel_eligibility_form": false,
      "normalized_status": "excluded",
      "exclusion_reason": "no_available_channel"
    }
  ]
}
```

---

## ルーティングルール

### 前提
`normalized_status` が `active` のものだけ routing 候補とする。

### run_mode = email
- email eligibility true → email_targets
- email eligibility false → excluded_targets

### run_mode = form
- form eligibility true → form_targets
- form eligibility false → excluded_targets

### run_mode = both
- email eligibility true なら email_targets へ追加
- form eligibility true なら form_targets へ追加
- 両方 false なら excluded_targets
- 両方 true の target は email_targets と form_targets の両方へ入る

### normalized_status != active
- 常に excluded_targets

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Routing Payload | IF | run_mode と targets の存在確認 |
| 3 | Stop Invalid Routing Input | Stop And Error | 必須入力不足時に停止 |
| 4 | Split Targets | Split Out / Item Lists | target を1件ずつ処理 |
| 5 | Exclude Non-Active Targets | IF | active 以外を除外 |
| 6 | Route By Run Mode | Switch | run_mode ごとに分岐 |
| 7 | Route Email Mode | IF | email モードの可否判定 |
| 8 | Build Email Target | Set | email_targets 用 item 生成 |
| 9 | Build Excluded From Email Mode | Set | email モード除外 item 生成 |
| 10 | Route Form Mode | IF | form モードの可否判定 |
| 11 | Build Form Target | Set | form_targets 用 item 生成 |
| 12 | Build Excluded From Form Mode | Set | form モード除外 item 生成 |
| 13 | Route Both Mode - Email | IF | both時の email 可否判定 |
| 14 | Build Both Email Target | Set | bothの email 側 item 生成 |
| 15 | Route Both Mode - Form | IF | both時の form 可否判定 |
| 16 | Build Both Form Target | Set | bothの form 側 item 生成 |
| 17 | Build Excluded From Both Mode | IF / Set | both でも両方不可なら除外 |
| 18 | Aggregate Email Targets | Aggregate / Item Lists | email 配列へ集約 |
| 19 | Aggregate Form Targets | Aggregate / Item Lists | form 配列へ集約 |
| 20 | Aggregate Excluded Targets | Aggregate / Item Lists | excluded 配列へ集約 |
| 21 | Return Routing Payload | Set | 最終返却形式へ整形 |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノード。

#### Input data mode
- 初期は `Accept all data`

#### 理由
- parent 側入力項目を固定しきる前でも開発しやすい
- 後で fields 定義や JSON example に移行できる

---

### 2. Validate Routing Payload
IF ノード。

#### 条件
- `run_mode` が存在する
- `run_mode` が `email` / `form` / `both` のいずれか
- `targets` が存在する
- `targets.length > 0`

#### true
- `Split Targets` へ

#### false
- `Stop Invalid Routing Input` へ

---

### 3. Stop Invalid Routing Input
Stop And Error ノード。

#### メッセージ例
```text
sub-channel-router: run_mode or targets is missing or invalid.
```

#### 意図
- 前段 workflow の異常を即時に検知する
- parent 側で不完全な routing を防ぐ

---

### 4. Split Targets
targets 配列を1件ずつ展開する。

#### 出力イメージ
```json
[
  {
    "execution_id": "ex_001",
    "run_mode": "both",
    "target_id": "t_001",
    "company_name": "Example Inc.",
    "contact_email": "info@example.com",
    "form_url": "https://example.com/contact",
    "channel_eligibility_email": true,
    "channel_eligibility_form": true,
    "normalized_status": "active",
    "exclusion_reason": null
  }
]
```

---

### 5. Exclude Non-Active Targets
IF ノード。

#### 条件
- `normalized_status == active`

#### true
- `Route By Run Mode` へ

#### false
- excluded item を生成して excluded 集約へ流す

#### 非active対象の扱い
- `invalid`
- `excluded`
- その他 active 以外

#### excluded item 例
```json
{
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "route_status": "excluded",
  "route_reason": "={{$json.exclusion_reason || 'non_active_status'}}",
  "execution_id": "={{$json.execution_id}}"
}
```

---

### 6. Route By Run Mode
Switch ノード。

#### case
- `email`
- `form`
- `both`

#### 理由
- 2分岐以上なので IF 連打より Switch の方が見通しがよい
- mode ごとの責務を分けやすい

---

## email モード分岐

### 7. Route Email Mode
IF ノード。

#### 条件
- `channel_eligibility_email == true`

#### true
- `Build Email Target`

#### false
- `Build Excluded From Email Mode`

---

### 8. Build Email Target
Set ノード。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "contact_email": "={{$json.contact_email}}",
  "channel": "email",
  "route_status": "routed"
}
```

---

### 9. Build Excluded From Email Mode
Set ノード。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "route_status": "excluded",
  "route_reason": "email_not_available"
}
```

---

## form モード分岐

### 10. Route Form Mode
IF ノード。

#### 条件
- `channel_eligibility_form == true`

#### true
- `Build Form Target`

#### false
- `Build Excluded From Form Mode`

---

### 11. Build Form Target
Set ノード。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "form_url": "={{$json.form_url}}",
  "channel": "form",
  "route_status": "routed"
}
```

---

### 12. Build Excluded From Form Mode
Set ノード。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "route_status": "excluded",
  "route_reason": "form_not_available"
}
```

---

## both モード分岐

### 13. Route Both Mode - Email
IF ノード。

#### 条件
- `channel_eligibility_email == true`

#### true
- `Build Both Email Target`

#### false
- 何も作らず次へ

---

### 14. Build Both Email Target
Set ノード。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "contact_email": "={{$json.contact_email}}",
  "channel": "email",
  "route_status": "routed"
}
```

---

### 15. Route Both Mode - Form
IF ノード。

#### 条件
- `channel_eligibility_form == true`

#### true
- `Build Both Form Target`

#### false
- 何も作らず次へ

---

### 16. Build Both Form Target
Set ノード。

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "form_url": "={{$json.form_url}}",
  "channel": "form",
  "route_status": "routed"
}
```

---

### 17. Build Excluded From Both Mode
IF ノードまたは Set ノード。

#### 条件
- `channel_eligibility_email == false`
- AND `channel_eligibility_form == false`

#### 出力例
```json
{
  "execution_id": "={{$json.execution_id}}",
  "target_id": "={{$json.target_id}}",
  "company_name": "={{$json.company_name}}",
  "route_status": "excluded",
  "route_reason": "no_available_channel_for_both"
}
```

---

## 集約ノード

### 18. Aggregate Email Targets
email へ流す item を `email_targets` 配列にまとめる。

### 19. Aggregate Form Targets
form へ流す item を `form_targets` 配列にまとめる。

### 20. Aggregate Excluded Targets
除外対象を `excluded_targets` 配列にまとめる。

#### ポイント
- target が0件でも workflow 全体が壊れないようにする
- 初期実装では branch ごとに明示的に空配列を返す Set ノードを挟んでもよい

---

### 21. Return Routing Payload
最終返却形式を固定する Set ノード。

#### 返却例
```json
{
  "execution_id": "ex_001",
  "run_mode": "both",
  "email_targets": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "contact_email": "info@example.com",
      "channel": "email",
      "route_status": "routed"
    },
    {
      "execution_id": "ex_001",
      "target_id": "t_002",
      "company_name": "Mail Only KK",
      "contact_email": "sales@mailonly.jp",
      "channel": "email",
      "route_status": "routed"
    }
  ],
  "form_targets": [
    {
      "execution_id": "ex_001",
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "form_url": "https://example.com/contact",
      "channel": "form",
      "route_status": "routed"
    }
  ],
  "excluded_targets": [
    {
      "execution_id": "ex_001",
      "target_id": "t_003",
      "company_name": "No Contact KK",
      "route_status": "excluded",
      "route_reason": "no_available_channel"
    }
  ]
}
```

---

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Routing Payload
    -> true  -> Split Targets
              -> Exclude Non-Active Targets
                   -> false -> excluded branch
                   -> true  -> Route By Run Mode
                                -> email -> Route Email Mode
                                          -> Build Email Target / Build Excluded From Email Mode
                                -> form  -> Route Form Mode
                                          -> Build Form Target / Build Excluded From Form Mode
                                -> both  -> Route Both Mode - Email
                                          -> Build Both Email Target
                                          -> Route Both Mode - Form
                                          -> Build Both Form Target
                                          -> Build Excluded From Both Mode
              -> Aggregate Email Targets
              -> Aggregate Form Targets
              -> Aggregate Excluded Targets
              -> Return Routing Payload
    -> false -> Stop Invalid Routing Input
```

---

## 実装上の注意

### 1. routing と scoring を混ぜない
この workflow は「どこへ流すか」だけに責務を絞る。
優先順位づけや営業難易度スコアは将来別 workflow で扱う方がよい。

### 2. both は “二者択一” ではない
`both` は email と form の両方へ流してよい、という意味にする。
ここを曖昧にすると KPI の意味がぶれる。

### 3. 非activeを先に落とす
`normalized_status != active` を最初に除外しておくことで、
後段の条件分岐がかなり単純になる。

### 4. excluded_targets を残す
送信しなかった対象も記録対象として重要。
後で KPI やリスト品質分析に使える。

### 5. empty branch に備える
たとえば run_mode=email で form_targets は空になる。
空配列でも後続 workflow が壊れない返却形式に固定する。

---

## 初期テストケース

### テスト1: run_mode=email
```json
{
  "execution_id": "ex_001",
  "run_mode": "email",
  "targets": [
    {
      "target_id": "t_001",
      "company_name": "Example Inc.",
      "contact_email": "info@example.com",
      "form_url": "https://example.com/contact",
      "channel_eligibility_email": true,
      "channel_eligibility_form": true,
      "normalized_status": "active"
    },
    {
      "target_id": "t_002",
      "company_name": "No Mail KK",
      "contact_email": null,
      "form_url": "https://nomail.jp/contact",
      "channel_eligibility_email": false,
      "channel_eligibility_form": true,
      "normalized_status": "active"
    }
  ]
}
```

### 期待結果
- `t_001` は email_targets
- `t_002` は excluded_targets

---

### テスト2: run_mode=form
```json
{
  "execution_id": "ex_002",
  "run_mode": "form",
  "targets": [
    {
      "target_id": "t_003",
      "company_name": "Form OK KK",
      "contact_email": "hello@formok.jp",
      "form_url": "https://formok.jp/contact",
      "channel_eligibility_email": true,
      "channel_eligibility_form": true,
      "normalized_status": "active"
    }
  ]
}
```

### 期待結果
- `t_003` は form_targets
- email_targets は空

---

### テスト3: run_mode=both
```json
{
  "execution_id": "ex_003",
  "run_mode": "both",
  "targets": [
    {
      "target_id": "t_004",
      "company_name": "Both OK KK",
      "contact_email": "sales@bothok.jp",
      "form_url": "https://bothok.jp/contact",
      "channel_eligibility_email": true,
      "channel_eligibility_form": true,
      "normalized_status": "active"
    }
  ]
}
```

### 期待結果
- `t_004` は email_targets と form_targets の両方に入る

---

### テスト4: 非active
```json
{
  "execution_id": "ex_004",
  "run_mode": "both",
  "targets": [
    {
      "target_id": "t_005",
      "company_name": "Bad Data KK",
      "contact_email": null,
      "form_url": null,
      "channel_eligibility_email": false,
      "channel_eligibility_form": false,
      "normalized_status": "excluded",
      "exclusion_reason": "no_available_channel"
    }
  ]
}
```

### 期待結果
- `t_005` は excluded_targets
- email_targets と form_targets は空

---

## MVP v1 時点の割り切り

- email 優先 / form 優先の重みづけはしない
- 同一 target のチャネル順序制御はしない
- 送信回数制限や冷却期間は見ない
- 過去送信履歴ベースの除外は見ない
- 顧客別特殊 routing は扱わない
- ブラックリストやドメイン除外は別 workflow に分ける

---

## 次の sub-workflow への受け渡し前提

### email 側へ渡すもの
- `execution_id`
- `email_targets[]`
- 各 item に `target_id`, `company_name`, `contact_email`

### form 側へ渡すもの
- `execution_id`
- `form_targets[]`
- 各 item に `target_id`, `company_name`, `form_url`

### 保持すべきもの
- `excluded_targets[]`

これにより、`sub-email-outreach` と `sub-form-outreach` は routing 判定を再度考えずに送信処理へ集中できる。
