# Workflow List

## Purpose

この文書は、Luvira Outreach OS で管理する n8n workflow の一覧、役割、責務境界、実装優先度を整理するための基準資料である。

## Management Policy

- 原則は `1 workflow = 1責務`
- parent workflow は全体オーケストレーションを担当する
- sub-workflow は単機能部品として分ける
- error workflow は共通エラー処理を担当する
- workflow 名は `main-...`、`sub-...`、`error-...` で統一する
- n8n 上では tag で絞り込みやすいように管理する

## Recommended Tags

- Project: `luvira-outreach-os`
- Type: `parent`, `sub`, `error-handler`
- Function: `ingest`, `validation`, `routing`, `email`, `form`, `logging`, `kpi`

## Workflow Inventory

| Workflow Name | Type | Responsibility | Priority | Notes |
|---|---|---|---|---|
| `main-ingest-and-run` | Parent | 全体オーケストレーション | High | parent workflow |
| `sub-csv-normalizer` | Sub | CSV/JSON入力の標準化 | High | 正規化担当 |
| `sub-target-validator` | Sub | target妥当性判定 | High | `active` / `excluded` / `invalid` を付与 |
| `sub-channel-router` | Sub | email/form/excluded振り分け | High | 分岐担当 |
| `sub-email-outreach` | Sub | email送信 | Medium | 初期はダミーでも可 |
| `sub-form-outreach` | Sub | フォーム送信 | Medium | 後で Browser Use 連携余地あり |
| `sub-channel-result-updater` | Sub | 実行結果整形 | Medium | 集計前の整形 |
| `sub-kpi-aggregator` | Sub | KPI集計 | Medium | execution 単位集計 |
| `sub-log-output-router` | Sub | Sheets等への保存 | High | 出力担当 |
| `error-handler` | Error | 共通エラー処理 | High | 共通通知と記録 |

## Build Order

以下の順番で実装する。

1. `sub-csv-normalizer`
2. `sub-target-validator`
3. `sub-channel-router`
4. `main-ingest-and-run`
5. `sub-log-output-router`
6. `error-handler`
7. `sub-email-outreach`
8. `sub-form-outreach`
9. `sub-channel-result-updater`
10. `sub-kpi-aggregator`

## Minimum MVP Set

まずは次の3本を通す。

- `main-ingest-and-run`
- `sub-target-validator`
- `sub-channel-router`

## Rule

- 新しい workflow を作る前にこの一覧へ追加する
- Claude に workflow JSON を作らせるときは、この一覧の workflow name を source of truth にする
- `WORKFLOW_NAME` と n8n 上の workflow 名は原則そろえる
