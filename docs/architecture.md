# Architecture

本ドキュメントでは、Luvira Outreach OSの全体システム構成、ワークフローの階層構造、および各コンポーネントの責務について定義します。「共通OS + 顧客設定レイヤー + 出力アダプタ層」の3層構造に基づき、保守性と拡張性を担保した設計を採用しています。

## Architecture Overview
Luvira Outreach OSは、顧客のn8n環境上にデプロイされるワークフロー群で構成されます。
単一の巨大なワークフローではなく、責務ごとに分割された「Sub-workflows（共通OS機能）」を、「Parent Workflow（顧客ごとの業務プロセス）」から呼び出すモジュール化されたアーキテクチャを採用しています。
さらに、ログの記録と出力を完全に分離することで、多様なログ連携要件に柔軟に対応します。

## Workflow Layers

システムは大きく3つのレイヤーに分かれます。

1. **Customer Context Layer (親レイヤー):**
   - 顧客ごとにカスタマイズされるレイヤー。
   - トリガー（定期実行、Webhook、CSVアップロード等）の受け口。
   - 顧客固有の変数（使用する文面テンプレート、除外条件など）の定義。
2. **Core OS Layer (共通レイヤー):**
   - Luviraが標準モジュールとして提供するサブワークフロー群。
   - 顧客環境間で共通のロジックを持ち、処理の実行と「標準ログスキーマ」でのイベント記録に専念する。
3. **Output Adapter Layer (出力アダプタ層):**
   - 共通化されたログを、顧客ごとの出力要件（Google Sheets、DB、Webhook等）に合わせて変換し、外部へ送信するレイヤー。

## Parent Workflow
`Main Outreach Router` などと呼ばれる、プロセスの起点となるワークフローです。

- **責務:** トリガーの受付、初期データの取得、顧客固有ルールの適用、各サブワークフローの順次呼び出し。
- **特徴:** 顧客ごとに一部カスタマイズが入る前提のため、複雑な処理は持たせず、データの受け渡し（オーケストレーション）に徹します。

## Sub-workflows
Core OS Layerを構成する主要なモジュール群です。以下は標準的に提供されるコンポーネントです。

1. **csv-normalizer:**
   - 顧客がアップロードした多様なフォーマットのリストを読み込み、内部の標準データモデルに変換・マッピングします。
2. **channel-router:**
   - 正規化された企業データと、顧客が設定したルールに基づき、対象企業へのアプローチを「フォーム」「メール」「両方」「除外」のいずれかに振り分けます。
3. **email-sender:**
   - 指定されたメールアドレスに対し、テンプレートと変数を組み合わせてメールを送信します。
   - 送信結果（成功、バウンス等）を正規化されたフォーマットで返却します。
4. **form-runner:**
   - 対象企業のURLから問い合わせフォームを探索し、指定された情報をPOST送信します。
   - 発見不可、CAPTCHAブロック、送信成功などの詳細なステータスを返却します。
5. **activity-logger:**
   - すべてのアクションの結果を受け取り、「標準ログスキーマ」に基づく内部イベントとして記録を担保します。出力先には依存しません。
6. **log-output-router:**
   - `activity-logger` を通過した標準ログを受け取り、設定値（出力先）に応じて適切なOutput Adapterへ振り分けます。
7. **kpi-aggregator:**
   - 顧客への出力先（Google SheetsかDBかなど）に関わらず、標準ログスキーマのデータを元に定期的（日次・月次等）なKPI集計を行い、レポート用のデータマートを生成します。

## Output Adapters
Output Adapter Layerを構成し、ログを指定の形式に変換して外部へ書き出します。顧客ごとに必要なモジュールを有効化します。

- **log-output-sheets:** Google Sheetsの指定フォーマットへ行追加するアダプタ。
- **log-output-csv:** 一定件数ごとにCSV化し、ストレージへエクスポートするアダプタ。
- **log-output-webhook:** 社内APIやチャットツール等に対してJSONペイロードを送信するアダプタ。
- **log-output-db-map:** 顧客独自のデータベースやCRM/SFAのテーブル構造に合わせて、カラムをマッピングしてINSERT/UPDATEするアダプタ。

## Data Model (標準ログスキーマ)
システム全体を流れ、KPIの計算元となる主要なログペイロードは、出力先に依存しない一貫したスキーマを持ちます。

```json
{
  "execution_id": "uuid",
  "company_id": "C_001",
  "company_name": "株式会社XXX",
  "source_list_id": "list_202410",
  "source_row_id": "row_001",
  "channel": "form",
  "status": "success",
  "reason_code": "submitted",
  "reason_detail": null,
  "timestamp": "2024-10-01T10:00:00Z",
  "operator": "auto",
  "payload_summary": "{\"formUrl\":\"https://example.com/contact\"}"
}
```

## Logging and KPI Aggregation
- 各サブワークフローは、実行結果を統一オブジェクトとして返し、Parent Workflowがそれを `activity-logger` に渡します。
- これにより、「KPI集計用の内部イベント」の完全性が担保されます。
- その後 `log-output-router` を経由して非同期に各Adapterで「顧客提出用の外部出力」が行われるため、出力処理の遅延やエラーがコア機能（OS本体）に影響を与えない設計となっています。

## Future Extensions
このモジュールアーキテクチャにより、将来的な拡張が容易になっています。

- **AI連携:** 送信前に各チャネルに挟み込むことで、文面のパーソナライズが可能になります。
- **出力先の柔軟な追加:** 既存のOS本体には一切手を加えず、新しい `log-output-*` アダプタを作成するだけで未知のシステムとの連携が実現できます。
