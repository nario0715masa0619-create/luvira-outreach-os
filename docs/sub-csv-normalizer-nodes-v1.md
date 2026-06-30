# sub-csv-normalizer 実ノード構成案 v1

## 概要
この文書は、luvira-outreach-os の sub-workflow `sub-csv-normalizer` を n8n 上で最初に実装するための実ノード構成案である。

この sub-workflow の目的は、親 workflow から受け取った raw CSV rows を、標準 `targets` スキーマへ変換することである。
MVP v1 では、高度なCSVパーサーではなく、列名揺れの最低限吸収と空値正規化に集中する。

---

## 役割

- parent workflow から raw rows を受け取る
- row 単位で標準 target 形式へマッピングする
- CSV列名の揺れを最低限吸収する
- source_row_id を付ける
- 後続 workflow が使いやすい `targets` 配列を返す

---

## 実装方針

- trigger は `Execute Sub-workflow Trigger`
- 初期は `Accept all data` で受ける
- row 単位の正規化を優先する
- 必須列不足でも即落とさず、可能な範囲で整形して返す
- validation は `sub-target-validator` 側に寄せる
- ここでは「CSVを読む」のではなく「受け取った rows を整形する」に責務を限定する

---

## ノード一覧

| No | Node Name | Node Type | 役割 |
|---|---|---|---|
| 1 | Execute Sub-workflow Trigger | Execute Sub-workflow Trigger | parent からの入力受信 |
| 2 | Validate Raw Rows Exists | IF | raw_rows の存在確認 |
| 3 | Stop Missing Raw Rows | Stop And Error | raw_rows 欠落時に停止 |
| 4 | Set Source Context | Set | source_list_id などの共通値整理 |
| 5 | Split Raw Rows | Split Out / Item Lists | raw_rows を1行ずつ処理できる形にする |
| 6 | Normalize Row Fields | Set | 列名揺れ吸収と標準項目へのマッピング |
| 7 | Normalize Empty Values | Set | 空文字や未定義値を null 化 |
| 8 | Add Row Metadata | Set | target_id, source_row_id, normalized_status を付与 |
| 9 | Return Normalized Target | Set | 後続で使う target 形へ固定 |
| 10 | Aggregate Targets | Aggregate / Item Lists | target 配列へ再集約 |
| 11 | Return Targets Payload | Set | 最終返却形式へ整形 |

---

## 入力仕様

### parent から受け取る想定入力
```json
{
  "source_list_id": "sl_001",
  "raw_rows": [
    {
      "company": "Example Inc.",
      "email": "info@example.com",
      "website": "https://example.com",
      "form": "https://example.com/contact"
    },
    {
      "company_name": "Demo LLC",
      "mail": "sales@demo.jp",
      "url": "https://demo.jp",
      "contact_form_url": "https://demo.jp/contact"
    }
  ]
}
```

### 想定する列名揺れ
MVP v1 では以下程度の揺れを吸収対象とする。

| 標準項目 | 候補列名 |
|---|---|
| company_name | `company`, `company_name`, `name` |
| contact_email | `email`, `mail`, `contact_email` |
| website_url | `website`, `url`, `site_url` |
| form_url | `form`, `form_url`, `contact_form_url` |
| phone | `phone`, `tel`, `telephone` |
| address | `address`, `addr` |
| industry | `industry`, `category` |

---

## 各ノード詳細

### 1. Execute Sub-workflow Trigger
sub-workflow の先頭ノード。

#### Input data mode
- 初期実装では `Accept all data`

#### 理由
- parent 側から柔軟にデータを流せる
- 初期開発で field 定義を固めすぎない
- 後で JSON example に寄せやすい

---

### 2. Validate Raw Rows Exists
IF ノード。

#### 条件
- `raw_rows` が存在する
- `raw_rows` が空配列ではない

#### true
- 次へ進む

#### false
- `Stop Missing Raw Rows` へ進む

#### 判定イメージ
- `raw_rows exists`
- `raw_rows length greater than 0`

---

### 3. Stop Missing Raw Rows
Stop And Error ノード。

#### メッセージ例
```text
sub-csv-normalizer: raw_rows is missing or empty.
```

#### 意図
- parent からの不正入力を早期検知する
- error-handler で拾えるようにする

---

### 4. Set Source Context
共通値を明示的に残す Set ノード。

#### 保持する項目
```json
{
  "source_list_id": "={{$json.source_list_id}}",
  "raw_rows": "={{$json.raw_rows}}"
}
```

#### ポイント
- 以降の row 展開で source_list_id を失わないようにする
- 必要なら uploaded context もここで保持する

---

### 5. Split Raw Rows
`raw_rows` 配列を1アイテムずつ展開する。

#### 目的
- 各 row を individual item として後続 Set ノードで処理する
- row ごとの正規化をシンプルにする

#### 出力イメージ
```json
[
  {
    "source_list_id": "sl_001",
    "row_index": 0,
    "row": {
      "company": "Example Inc.",
      "email": "info@example.com",
      "website": "https://example.com",
      "form": "https://example.com/contact"
    }
  },
  {
    "source_list_id": "sl_001",
    "row_index": 1,
    "row": {
      "company_name": "Demo LLC",
      "mail": "sales@demo.jp",
      "url": "https://demo.jp",
      "contact_form_url": "https://demo.jp/contact"
    }
  }
]
```

---

### 6. Normalize Row Fields
Set ノードで列名揺れを標準項目へ寄せる。

#### 推奨モード
- JSON Output
- Include in Output: 必要に応じて All Input Fields または No Other Fields
- 初期は `Keep Only Set Fields` を使って標準化後の形を明確にする

#### JSON Output 例
```json
{
  "source_list_id": "={{$json.source_list_id}}",
  "row_index": "={{$json.row_index}}",
  "company_name": "={{$json.row.company_name || $json.row.company || $json.row.name || ''}}",
  "contact_email": "={{$json.row.contact_email || $json.row.email || $json.row.mail || ''}}",
  "website_url": "={{$json.row.website_url || $json.row.website || $json.row.url || $json.row.site_url || ''}}",
  "form_url": "={{$json.row.form_url || $json.row.form || $json.row.contact_form_url || ''}}",
  "phone": "={{$json.row.phone || $json.row.tel || $json.row.telephone || ''}}",
  "address": "={{$json.row.address || $json.row.addr || ''}}",
  "industry": "={{$json.row.industry || $json.row.category || ''}}"
}
```

#### 意図
- このノードで「何列名だったか」を忘れてよい状態にする
- 以降は標準名だけを見る

---

### 7. Normalize Empty Values
空文字を null に寄せる Set ノード。

#### JSON Output 例
```json
{
  "source_list_id": "={{$json.source_list_id}}",
  "row_index": "={{$json.row_index}}",
  "company_name": "={{$json.company_name || null}}",
  "contact_email": "={{$json.contact_email || null}}",
  "website_url": "={{$json.website_url || null}}",
  "form_url": "={{$json.form_url || null}}",
  "phone": "={{$json.phone || null}}",
  "address": "={{$json.address || null}}",
  "industry": "={{$json.industry || null}}"
}
```

#### 理由
- 後続 IF 判定を簡単にする
- 空文字と未設定を分けて悩まなくてよい

---

### 8. Add Row Metadata
target 用の補助項目を付与する。

#### 追加項目
```json
{
  "target_id": "={{'t_' + $json.source_list_id + '_' + $json.row_index}}",
  "source_row_id": "={{'row_' + $json.row_index}}",
  "normalized_status": "active"
}
```

#### 注意
- 初期は簡易IDでよい
- 将来は UUID 生成や親 workflow 側採番へ切り替えてもよい

---

### 9. Return Normalized Target
最終的な1件分の target 形式に整える。

#### 出力例
```json
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
}
```

#### ポイント
- validator 以降がそのまま使える形にする
- 不要な raw data をここで落としてもよい

---

### 10. Aggregate Targets
個別 item を再度配列へまとめる。

#### 目的
- parent workflow が `targets` 配列として受け取れるようにする
- child output を標準化する

#### 出力イメージ
```json
{
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
      "company_name": "Demo LLC",
      "website_url": "https://demo.jp",
      "contact_email": "sales@demo.jp",
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

### 11. Return Targets Payload
最後の返却形式を固定する Set ノード。

#### 最終返却例
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
      "company_name": "Demo LLC",
      "website_url": "https://demo.jp",
      "contact_email": "sales@demo.jp",
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

## 接続順序

```text
Execute Sub-workflow Trigger
  -> Validate Raw Rows Exists
    -> true  -> Set Source Context
              -> Split Raw Rows
              -> Normalize Row Fields
              -> Normalize Empty Values
              -> Add Row Metadata
              -> Return Normalized Target
              -> Aggregate Targets
              -> Return Targets Payload
    -> false -> Stop Missing Raw Rows
```

---

## 初期テストケース

### テスト1
```json
{
  "source_list_id": "sl_001",
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

### 期待結果
- target が1件返る
- `company_name`, `contact_email`, `website_url`, `form_url` に正規化される
- `normalized_status` は `active`

---

### テスト2
```json
{
  "source_list_id": "sl_001",
  "raw_rows": [
    {
      "company_name": "Demo LLC",
      "mail": "sales@demo.jp",
      "url": "https://demo.jp",
      "contact_form_url": "https://demo.jp/contact"
    }
  ]
}
```

### 期待結果
- 列名揺れを吸収できる
- `contact_email = sales@demo.jp`
- `website_url = https://demo.jp`
- `form_url = https://demo.jp/contact`

---

### テスト3
```json
{
  "source_list_id": "sl_001",
  "raw_rows": []
}
```

### 期待結果
- Stop Missing Raw Rows で停止
- error-handler で記録可能

---

## MVP v1 時点の割り切り

- CSVの文字コード問題までは扱わない
- ヘッダ判定の高度化は行わない
- 会社名の表記ゆれ補正は行わない
- URL正規化は最低限に留める
- email の厳密妥当性判定は validator 側へ回す
- industry 分類の自動補完は行わない

---

## 次の sub-workflow への受け渡し前提
この workflow の出力は、次の `sub-target-validator` がそのまま受け取れる形にする。

### 前提出力
- `source_list_id`
- `targets[]`
- 各 target は標準項目のみ保持

これにより、validation, routing, KPI, output の全段が同じ内部スキーマで進められる。
