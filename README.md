# luvira-outreach-os

Luvira Outreach OS は、n8n を中核にして営業実行、送信結果管理、KPI 集計を標準化するためのアウトリーチ自動化基盤です。

このリポジトリでは、MVP v1 の設計ドキュメント、workflow 分割方針、入出力スキーマ、運用前提を管理します。

---

## 目的

このプロジェクトの目的は以下の3つです。

- 営業自動化の標準MVPを定義する
- n8n workflow を保守しやすい単位で分割する
- Google Sheets を標準出力先とした運用可能な基盤を整える

---

## スコープ

MVP v1 では以下を標準範囲とします。

- CSV または Webhook 起点の target 取込
- target 正規化
- target 妥当性判定
- email / form のチャネル分岐
- email 送信
- form 送信
- channel result 更新
- KPI 集計
- Google Sheets への標準出力

MVP v1 では以下は原則スコープ外です。

- 高度なパーソナライズ生成
- 開封率、返信率、商談化率の追跡
- multi-step form 完全対応
- CAPTCHA 対応
- 顧客別の高度な routing ルール
- DB、BigQuery、Notion など複数出力先対応

---

## アーキテクチャ

workflow は parent workflow 1 本と複数の sub-workflow に分割します。

### parent workflow
- `main-ingest-and-run`

### sub-workflows
- `sub-csv-normalizer`
- `sub-target-validator`
- `sub-channel-router`
- `sub-email-outreach`
- `sub-form-outreach`
- `sub-channel-result-updater`
- `sub-kpi-aggregator`
- `sub-log-output-router`

---

## workflow 一覧

| Workflow | 役割 |
|---|---|
| `main-ingest-and-run` | 全体のオーケストレーション |
| `sub-csv-normalizer` | CSV / row の標準 schema 化 |
| `sub-target-validator` | 実行対象可否判定 |
| `sub-channel-router` | email / form / excluded への振り分け |
| `sub-email-outreach` | email 送信処理 |
| `sub-form-outreach` | form 送信処理 |
| `sub-channel-result-updater` | latest channel result 更新 |
| `sub-kpi-aggregator` | execution 単位 KPI 集計 |
| `sub-log-output-router` | Google Sheets への標準出力 |

---

## データフロー

```text
main-ingest-and-run
  -> sub-csv-normalizer
  -> sub-target-validator
  -> sub-channel-router
     -> sub-email-outreach
     -> sub-form-outreach
  -> sub-channel-result-updater
  -> sub-kpi-aggregator
  -> sub-log-output-router
```

---

## 標準出力先

MVP v1 の標準出力先は Google Sheets とします。

### 想定シート
- `activity_logs`
- `channel_results_latest`
- `kpi_snapshots`

---

## ディレクトリ構成

```text
luvira-outreach-os/
├── README.md
└── docs/
    ├── index.md
    ├── mvp-v1.md
    ├── workflow-json-mock-v1.md
    ├── main-ingest-and-run.md
    ├── sub-csv-normalizer.md
    ├── sub-target-validator.md
    ├── sub-channel-router.md
    ├── sub-email-outreach.md
    ├── sub-form-outreach.md
    ├── sub-channel-result-updater.md
    ├── sub-kpi-aggregator.md
    └── sub-log-output-router.md
```

---

## 今後の実装順

実装は以下の順で進めます。

1. `main-ingest-and-run`
2. `sub-csv-normalizer`
3. `sub-target-validator`
4. `sub-channel-router`
5. `sub-email-outreach`
6. `sub-form-outreach`
7. `sub-channel-result-updater`
8. `sub-kpi-aggregator`
9. `sub-log-output-router`

---

## 運用ルール

- 各 workflow は 1 つの責務に絞る
- 入出力 schema は docs に先に書く
- Google Sheets の列は docs に固定してから実装する
- parent workflow は orchestration に徹する
- 送信実行と集計、保存を混在させない
