# luvira-outreach-os Workflow Architecture v1

## 概要
この文書は、luvira-outreach-os のMVP v1を実装するためのワークフロー構成を定義する。
目的は、n8n 上で営業自動化OSを modular に構築し、実装・保守・顧客別拡張をしやすい責務分離を先に固定することである。

本設計では、親ワークフローが全体実行を制御し、CSV正規化、対象判定、チャネル分岐、営業実行、ログ記録、出力変換、KPI集計、エラー処理を sub-workflow に分割する。

---

## 設計方針
ワークフロー設計の方針は以下の通り。

- 1本の巨大 workflow に集約しない
- 親 workflow は制御とオーケストレーションに専念する
- 業務責務ごとに sub-workflow を分離する
- 内部データは標準スキーマで流す
- 出力先の差分は output adapter で吸収する
- KPI は出力先依存ではなく内部ログから計算する
- エラー処理は共通 error workflow に集約する

---

## 全体構成
MVP v1 の全体構成は以下の3層で考える。

1. 入口・制御層
2. 実行・記録層
3. 出力・集計層

### 1. 入口・制御層
- CSVアップロード
- Webhook入力
- 手動実行
- 将来のSchedule実行

### 2. 実行・記録層
- CSV正規化
- 対象判定
- チャネル分岐
- メール営業実行
- フォーム営業実行
- 標準ログスキーマ記録
- 最新状態サマリ更新

### 3. 出力・集計層
- Google Sheets標準出力
- 将来のCSV出力
- 将来のWebhook出力
- 将来のDB向け変換
- KPI集計
- エラー通知

---

## 親ワークフロー一覧

### 1. main-ingest-and-run
MVP v1 の主親ワークフロー。
CSVまたはWebhookを入口として受け取り、以降の sub-workflow を順に呼び出す。

#### 責務
- 実行開始
- execution_id の採番または引継ぎ
- 入力形式の判定
- sub-workflow 呼び出し順の制御
- 実行全体の status 更新

#### 想定入口
- Manual Trigger
- Webhook
- 将来は Schedule Trigger

---

### 2. error-handler
共通エラー処理ワークフロー。
Error Trigger を先頭に持ち、各 workflow で発生した production エラーを受けて記録・通知する。

#### 責務
- エラー情報の受信
- 標準エラーログへの変換
- activity_logs への記録
- 通知チャネルへの送信
- 必要に応じた再処理判断

---

## sub-workflow 一覧

### 1. sub-csv-normalizer
CSV入力を標準スキーマへ変換する。

#### 入力
- source_list metadata
- raw rows

#### 出力
- targets 形式の配列

#### 責務
- 列名揺れの吸収
- 空値の標準化
- 基本整形
- source_row_id の付与

---

### 2. sub-target-validator
営業対象として利用可能かを判定する。

#### 入力
- targets 配列

#### 出力
- 判定済み targets 配列

#### 責務
- company_name の有無確認
- contact_email / form_url の基本妥当性確認
- 除外対象判定
- channel_eligibility_email の設定
- channel_eligibility_form の設定

---

### 3. sub-channel-router
実行モードと対象条件からチャネルを決定する。

#### 入力
- targets 配列
- run_mode
- execution context

#### 出力
- routing 済み targets 配列

#### 責務
- `email` / `form` / `both` の分岐
- メール対象のみ
- フォーム対象のみ
- 両方対象
- どちらも不可の除外

---

### 4. sub-email-outreach
メール営業を実行する。

#### 入力
- email対象 targets
- execution context

#### 出力
- email channel_results
- email activity logs

#### 責務
- 送信前ログ
- メール送信
- 成否結果の生成
- latest status 更新用データ返却

---

### 5. sub-form-outreach
フォーム営業を実行する。

#### 入力
- form対象 targets
- execution context

#### 出力
- form channel_results
- form activity logs

#### 責務
- 送信前ログ
- フォーム送信実行
- 成否結果の生成
- latest status 更新用データ返却

---

### 6. sub-activity-logger
標準ログスキーマへの記録を担当する。

#### 入力
- execution context
- event payload

#### 出力
- activity_logs 形式

#### 責務
- log_id 採番
- event_type の統一
- status / reason_code / reason_detail の整形
- occurred_at の付与
- 内部ログスキーマ化

---

### 7. sub-log-output-router
標準ログスキーマから出力先ごとの形式へ振り分ける。

#### 入力
- activity_logs
- output settings

#### 出力
- adapter 向け整形済み payload

#### 責務
- 出力先設定の判定
- adapter の呼び出し
- 標準ログと外部出力の分離

---

### 8. log-output-sheets
Google Sheets 向け標準出力アダプタ。

#### 入力
- activity_logs
- channel_results
- kpi_snapshots

#### 出力
- Google Sheets 書き込み結果

#### 責務
- activity_logs の Append
- channel_results の Update / Append or Update
- kpi_snapshots の Update / Append or Update

---

### 9. log-output-csv
CSVエクスポート向け出力アダプタ。

#### 入力
- 標準ログまたは集計データ

#### 出力
- CSV形式データ

#### 責務
- 顧客提出用CSVの生成
- 内部スキーマから外部列順への並び替え

---

### 10. log-output-webhook
外部API / Webhook向け出力アダプタ。

#### 入力
- 標準ログまたは集計データ

#### 出力
- Webhook送信用payload

#### 責務
- 顧客外部システム向け payload 変換
- POST送信

---

### 11. log-output-db-map
顧客独自DB向けの列名変換アダプタ。

#### 入力
- 標準ログまたは集計データ
- 顧客別mapping設定

#### 出力
- 顧客DB取り込み向け構造

#### 責務
- 標準スキーマから顧客列へ変換
- 顧客別 ingest 形式への整形

---

### 12. sub-channel-result-updater
各対象の最新チャネル結果を更新する。

#### 入力
- channel_results

#### 出力
- 更新済み channel_results

#### 責務
- latest status 管理
- delivered / replied / positive_reply などの反映

---

### 13. sub-kpi-aggregator
内部ログと結果データからKPIを計算する。

#### 入力
- source_list
- executions
- activity_logs
- channel_results

#### 出力
- kpi_snapshots

#### 責務
- フォーム保有率
- フォーム送信成功率
- メール保有率
- メール送信成功率
- リスト適合性指標

---

## 実行順序
main-ingest-and-run は概ね以下の順に sub-workflow を呼ぶ。

1. 入口受信
2. execution 作成
3. sub-csv-normalizer
4. sub-target-validator
5. sub-channel-router
6. sub-email-outreach / sub-form-outreach
7. sub-activity-logger
8. sub-channel-result-updater
9. sub-log-output-router
10. sub-kpi-aggregator
11. 実行完了更新

---

## 入口設計

### CSV入口
MVP v1 の標準入口。
人手運用でも説明しやすく、顧客導入初期に向いている。

#### 想定フロー
- CSVを受け取る
- source_lists を作成
- raw rows を targets に正規化
- 実行開始

---

### Webhook入口
拡張入口。
外部システムから直接データを流し込む場合に使う。

#### 設計ルール
- 開発中は Test URL を使用する
- 本番では Production URL を使用する
- Production URL は workflow を保存・公開・有効化した状態で利用する
- Test URL と Production URL は用途を混同しない

---

## 入出力ルール

### 基本ルール
- すべての sub-workflow は標準JSON構造を返す
- 子 workflow の最後の node 出力を親 workflow が受け取る
- execution_id, source_list_id, target_id を必ず引き回す
- output adapter 層では内部スキーマを書き換えない
- 顧客向け出力はコピー変換で扱う

### Input Data Mode の方針
sub-workflow では以下のいずれかで入力を定義する。

- Define using fields below
- Define using JSON example
- Accept all data

MVP v1 では、早期実装を優先する sub-workflow は Accept all data でもよいが、責務が安定してきた段階で fields または JSON example に寄せて厳格化する。

---

## ログ設計

### 標準ログの考え方
ログは workflow 末端のベタ書きではなく、独立責務として扱う。
activity-logger は内部標準スキーマへの記録を担当し、output-router は出力先への変換と振り分けを担当する。

### 標準ログ項目
- execution_id
- target_id
- channel
- event_type
- status
- reason_code
- reason_detail
- occurred_at
- operator
- payload_summary

### 設計意図
- KPIは内部ログ基準で計算する
- 顧客ごとの出力形式差分に内部設計を引きずられない
- 実行証跡と顧客提出形式を分離する

---

## エラー処理設計

### error-handler の基本構成
- Error Trigger
- エラー内容整形
- activity_logs 用の標準エラーログ生成
- 通知送信
- 必要に応じた dead-letter 的保存

### 運用ルール
- 一時エラーは node 単位の retry を使う
- 業務的に止めるべきエラーは明示的に失敗させる
- production 実行で発生したエラーを error-handler に集約する
- 通知だけでなく内部ログにも残す

---

## Google Sheets 標準出力方針
MVP v1 の標準出力先は Google Sheets とする。

### シート分割例
- `source_lists`
- `targets`
- `activity_logs`
- `channel_results`
- `kpi_snapshots`

### 保存方針
- activity_logs：Append Row
- channel_results：Update Row または Append or Update Row
- kpi_snapshots：Append or Update Row

---

## 設定レイヤー
共通OS本体と顧客別設定を分離する。

### 共通OS本体
- ワークフロー責務
- 標準ログスキーマ
- KPI定義
- 基本分岐ロジック

### 顧客設定レイヤー
- run_mode
- 出力先設定
- 列マッピング
- 送信条件
- 除外条件
- 顧客別 adapter 設定

### 出力アダプタ層
- Sheets
- CSV
- Webhook
- DB map

---

## 将来拡張ポイント
MVP v1 以降で追加しやすい拡張は以下の通り。

- Schedule実行
- CRM / SFA 連携
- AI文面生成
- Browser Use 連携
- 高度な承認フロー
- 顧客別DB同期
- 複数チャネル同時制御
- より厳格な sub-workflow input schema 定義

---

## 位置づけ
この workflow architecture v1 は、luvira-outreach-os の中核実装ルールを定義する文書である。
目的は、共通OSの構造を固定し、実装担当者が n8n 上で迷わず workflow を分割・接続できるようにすることにある。

この設計により、共通OSの中核は維持したまま、顧客別差分は設定レイヤーと出力アダプタ層で吸収する。
