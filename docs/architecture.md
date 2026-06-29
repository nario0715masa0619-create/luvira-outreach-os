# Architecture

本ドキュメントでは、Luvira Outreach OSの全体システム構成、ワークフローの階層構造、および各コンポーネントの責務について定義します。「共通OS + 顧客設定レイヤー」の思想に基づき、保守性と拡張性を担保した設計を採用しています。

## Architecture Overview
Luvira Outreach OSは、顧客のn8n環境上にデプロイされるワークフロー群で構成されます。
単一の巨大なワークフローではなく、責務ごとに分割された「Sub-workflows（共通OS機能）」を、「Parent Workflow（顧客ごとの業務プロセス）」から呼び出すモジュール化されたアーキテクチャを採用しています。

これにより、コア機能のアップデートを容易にしつつ、顧客固有の要件を柔軟に吸収します。

## Workflow Layers

システムは大きく2つのレイヤーに分かれます。

1. **Customer Context Layer (親レイヤー):**
   - 顧客ごとにカスタマイズされるレイヤー。
   - トリガー（定期実行、Webhook、CSVアップロード等）の受け口。
   - 顧客固有の変数（使用する文面テンプレート、除外条件など）の定義。
2. **Core OS Layer (共通レイヤー):**
   - Luviraが標準モジュールとして提供するサブワークフロー群。
   - 顧客環境間で共通のロジックを持ち、処理の実行とログの返却に専念する。

## Parent Workflow
`Main Outreach Router` などと呼ばれる、プロセスの起点となるワークフローです。

- **責務:** トリガーの受付、初期データの取得、顧客固有ルールの適用、各サブワークフローの順次呼び出し。
- **特徴:** 顧客ごとに一部カスタマイズが入る前提のため、複雑な処理は持たせず、データの受け渡し（オーケストレーション）に徹します。

## Sub-workflows
Core OS Layerを構成する主要なモジュール群です。以下は標準的に提供されるコンポーネントです。

1. **csv-normalizer:**
   - 顧客がアップロードした多様なフォーマットのリストを読み込み、内部の標準データモデル（Standard Data Model）に変換・マッピングします。
2. **channel-router:**
   - 正規化された企業データと、顧客が設定したルールに基づき、対象企業へのアプローチを「フォーム」「メール」「両方」「除外」のいずれかに振り分けます。
3. **email-sender:**
   - 指定されたメールアドレスに対し、テンプレートと変数を組み合わせてメールを送信します。
   - 送信結果（成功、バウンス等）を正規化されたフォーマットで返却します。
4. **form-runner:**
   - 対象企業のURLから問い合わせフォームを探索し、指定された情報をPOST送信します。
   - 発見不可、CAPTCHAブロック、送信成功などの詳細なステータスを返却します。
5. **activity-logger:**
   - すべてのアクションの結果を受け取り、スプレッドシートやデータベース等、指定のログストレージに書き込みます。
   - KPI集計の元データとなる厳密なスキーマを保証します。
6. **kpi-aggregator:**
   - 定期的（日次・月次等）にログデータを集計し、レポート用のデータマートを生成します。

## Data Model
システム全体を流れる主要なペイロードは、正規化された一貫したスキーマを持ちます。

```json
{
  "executionId": "uuid",
  "target": {
    "companyName": "株式会社XXX",
    "domain": "example.com",
    "formUrl": "https://example.com/contact",
    "email": "info@example.com"
  },
  "context": {
    "templateId": "tmpl_001",
    "campaignId": "camp_202410"
  },
  "result": {
    "channel": "form",
    "status": "success",
    "errorReason": null
  }
}
```

## Logging and KPI Aggregation
- 各サブワークフロー（email-sender, form-runner等）は、必ず実行結果を統一された `result` オブジェクトとして返します。
- Parent Workflowはそれらを集約し、非同期で `activity-logger` に渡すことで、処理の遅延を防ぎます。
- 分析基盤（Looker Studio等）はこのログデータを直接、あるいは `kpi-aggregator` を経由して参照します。

## Future Extensions
このモジュールアーキテクチャにより、将来的な拡張が容易になっています。

- **AI連携:** `message-generator` サブワークフローを追加し、送信前に各チャネルに挟み込むことで、文面のパーソナライズが可能になります。
- **Browser Use連携:** `form-runner` の代替またはフォールバックとして、ブラウザ操作を行う新しいサブワークフローを追加するだけで、既存のプロセスを壊さずに高度なフォーム送信を導入できます。
