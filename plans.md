# WAAM 自律協調エージェント工場 POC — 計画書

- 作成日: 2026-04-26
- ターゲット日: 2026-07-13(動くデモ)
- ステータス: ドラフト(ブレインストーミング合意済み、実装計画はこれから)

---

## 1. 概要 / ゴール

重工・船舶向け複雑形状金属部品の高付加価値領域である **WAAM(Wire Arc Additive Manufacturing)** 工場を対象に、製造現場の主要 7 職種を AI エージェントで代替し、それらが相互的に連携して工場を自律的に運用する様子を示す **POC** を構築する。

7/13 のデモ時点で、仮想工場(WAAM セル 2 台 + 既存 TimescaleDB スキーマ)上で異常イベントの連鎖シナリオ(ワイヤ残量低下 → 装置故障)を発火させ、7 エージェントがハイブリッド協調(指揮者 + ピア)で議論しながら動的に再計画し、シミュレータの状態を実際に更新する一連の流れを見せる。

---

## 2. スコープ

### In-Scope
- 仮想工場上での POC(実機接続なし)
- WAAM 多品種少量、重工 / 船舶向け複雑形状金属部品
- 7 エージェント:Operator / Planner / Welding Engineer / Material Handler / Maintenance / Supervisor / QA
- ハイブリッド協調(共有チャネル + ピア DM)
- 連鎖シナリオ S-3:B-1(ワイヤ残量)→ B-3(装置故障)
- 既存 TimescaleDB の実スキーマに沿った合成イベント注入(D 方式)
- 自律レベル L2 主軸 + 重要決定 L3(人間承認モーダル)
- チャット中心 UI + 簡易ステータスパネル

### Out-of-Scope(YAGNI)
- 実 WAAM 装置の制御・OPC UA / MQTT 等の実機連携
- 実顧客データの利用(合成のみ)
- 認証 / 船級審査ワークフロー
- フロントオフィス領域(受注・DfAM・見積)
- 多並列スケジューリング(2 セル超)
- ファインチューニング・学習(プロンプト + 小規模 RAG のみ)
- 認可 / RBAC、本番品質のセキュリティ
- 多言語化(日本語のみ)
- 工場フロアの 3D 俯瞰図

---

## 3. 全体アーキテクチャ

```
┌──────────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)                         │
│  ┌─────────────────────────┐   ┌─────────────────────────────┐   │
│  │  チャットパネル          │   │  ステータスパネル              │   │
│  │  - 共有チャネル(全員)   │   │  - Cell-A / Cell-B 状態       │   │
│  │  - DM(ピア間直接対話)  │   │  - ワイヤ残量・ジョブキュー   │   │
│  │  - 承認モーダル(L3用) │   │  - 意思決定タイムライン        │   │
│  └────────────▲────────────┘   └─────────────▲───────────────┘   │
└───────────────│────────────WebSocket──────────│──────────────────┘
                │                                │
┌───────────────▼────────────FastAPI─────────────▼──────────────────┐
│                    Backend (Python / LangGraph)                    │
│                                                                    │
│   ┌─────────────────  Agent Graph (LangGraph) ─────────────────┐  │
│   │                                                              │  │
│   │   ┌─Supervisor─┐   ┌─Operator─┐   ┌──Welding Eng.──┐       │  │
│   │   │ 調停・最終 │   │ 検知・   │   │ パラメータ・    │       │  │
│   │   │ 承認        │   │ 一次対応 │   │ 品質影響        │       │  │
│   │   └─────┬──────┘   └────┬─────┘   └────────┬───────┘       │  │
│   │         │               │                  │               │  │
│   │   ┌─────▼──────┐  ┌─────▼─────┐    ┌──────▼───────┐       │  │
│   │   │  Planner   │  │  Material │    │ Maintenance  │       │  │
│   │   │  計画・    │  │  ワイヤ・   │    │  故障・復旧  │       │  │
│   │   │  再計画    │  │  在庫     │    │              │       │  │
│   │   └─────┬──────┘  └─────┬─────┘    └──────┬───────┘       │  │
│   │         │               │                  │               │  │
│   │         └───────────────┼──────────────────┘               │  │
│   │                         │                                  │  │
│   │                  ┌──────▼─────┐                            │  │
│   │                  │     QA     │                            │  │
│   │                  │  品質可否  │                            │  │
│   │                  └────────────┘                            │  │
│   │                                                              │  │
│   │  共有 State: 現在のインシデント / 議論履歴 / 提案集約          │  │
│   └──────────────────────────────────────────────────────────────┘  │
│         ▲                            ▲                             │
│         │ events / tool calls        │ state mutations             │
│   ┌─────┴──────┐              ┌──────▼───────┐                     │
│   │ Event Bus  │              │ Sim State    │                     │
│   │ in-mem     │              │ Manager      │                     │
│   │ pub/sub    │              │ cells/jobs/  │                     │
│   └─────▲──────┘              │ wire/sched   │                     │
│         │                     └──────▲───────┘                     │
│   ┌─────┴──────┐                     │                             │
│   │ Scenario   │              ┌──────▼───────┐                     │
│   │ Injector   │──────────────▶ TimescaleDB  │                     │
│   │ S-3 脚本   │              │ 実スキーマ + │                     │
│   └────────────┘              │ sim tables   │                     │
│                                └──────────────┘                     │
└────────────────────────────────────────────────────────────────────┘
```

### 設計上のポイント

1. **2 系統のチャネル**:共有チャネル(議題提起・合意形成)+ DM(ドメイン特化のピア相談)。ハイブリッド協調を物理的に実装。
2. **Event Bus と Sim State の分離**:エージェントは Bus からイベントを受け取り、Tool 経由で State Manager / TimescaleDB を読み書き。「議論中のチャットは流れる / 状態は議論結果が確定したときだけ更新」を分離。
3. **Scenario Injector は別プロセス**:S-3 脚本を時刻ベースで発火。エージェントは脚本の存在を知らず、純粋にイベントに反応する。

---

## 4. エージェント定義(7 体)

各エージェントは以下の 3 要素で構成:
- **Role Prompt**:役割・観点・口調を規定する Claude 用システムプロンプト
- **Tools**:LangGraph のツール呼び出しで参照可能な関数群
- **Knowledge**:静的プロンプト埋め込み + 必要に応じ小規模 RAG

| Agent | 主な責務 | 主な Tools | Knowledge 源 |
|---|---|---|---|
| **Operator** | 装置監視 / 一次検知 / 議題提起 / 安全停止判断 | `get_cell_status`, `get_recent_alerts`, `acknowledge_alert` | 装置操作 SOP |
| **Planner** | ジョブ割付 / 納期管理 / 再計画 | `get_schedule`, `simulate_reschedule`, `commit_schedule` | ジョブ優先度ルール、納期影響評価 |
| **Welding Engineer** | 溶接パラメータ / 品質影響評価 / 装置間差吸収 | `get_weld_params`, `predict_quality_impact`, `propose_param_adjustment` | WAAM 欠陥モード、パラメータ標準 |
| **Material Handler** | ワイヤ在庫 / 消費予測 / 補充手配 | `get_wire_inventory`, `predict_wire_consumption`, `place_replenishment_order` | ワイヤ規格、補充リードタイム |
| **Maintenance** | 故障診断 / 復旧時間見積 / 修理手配 | `get_alarm_codes`, `lookup_repair_history`, `estimate_recovery_time` | 装置保全マニュアル、過去故障事例 |
| **QA** | 中断 / 再開時の品質可否 / 検査計画 | `get_quality_criteria`, `assess_resume_feasibility`, `flag_for_inspection` | 船級規格、検査基準 |
| **Supervisor** | 議論調停 / 最終承認 / 優先度決定 | `request_human_approval`(L3 用), `commit_decision` | 経営目標、KPI |

LLM は全エージェント Claude(初期は claude-opus-4-7 統一)。コストとレイテンシの観点で、Supervisor 以外は claude-sonnet-4-6 への切替を後段で検討。

---

## 5. 協調プロトコル

### 基本フロー

1. Event Bus がイベントを受信(例:`wire_low(cell=A, current_kg=2.1, predicted_empty_at=T+5h)`)
2. Operator が最初に反応:イベントを共有チャネルに投稿(議題提起)
3. Supervisor がトピックを承認し、関係エージェントを召喚
4. 関係エージェントは:
   - 共有チャネルで意見を述べる(全員可視)
   - 必要なら DM でピア相談(例:Operator ↔ Welding Eng. でパラメータ確認)
5. Supervisor が一定ターン後または合意検出時に集約
6. 決定をコミット(L2:状態更新 / L3:承認モーダル → 承認後コミット)
7. UI へイベント・チャット・状態変化を WebSocket で配信

### メッセージスキーマ(共有 State の一部)

```typescript
type Message = {
  id: string
  channel: "shared" | `dm:${agentA}-${agentB}`
  from: AgentRole
  to?: AgentRole              // DM の場合のみ
  type: "propose" | "question" | "answer" | "agree" | "disagree" | "decision"
  content: string
  timestamp: ISO8601
  citations?: ToolCallRef[]   // 根拠の透明性
}
```

### 終端条件
- Supervisor が `commit_decision` を呼んだ時
- ターン上限(初期値:30 ターン)を超えた時:Supervisor が現状最良案を選択

---

## 6. データ層(TimescaleDB)

### 既存スキーマ(再利用)
- 溶接データ時系列(電流・電圧・ワイヤ送給速度・ガス流量、装置単位、ms オーダ)
- ロボットデータ時系列(関節角・TCP 位置、ms オーダ)
- アラーム履歴

### POC で追加する sim tables
- `cells(cell_id, name, status, current_job_id, ramp_rate, ...)`
- `jobs(job_id, part_no, customer, deadline, est_hours, status, current_layer, total_layers, alloy, ...)`
- `wire_spools(spool_id, alloy, total_kg, remaining_kg, mounted_cell_id, ...)`
- `schedule(slot_id, cell_id, job_id, planned_start, planned_end, ...)`
- `decisions(decision_id, incident_id, agents_involved, chosen_action, rationale, approval_status, committed_at, ...)`

注入イベントは時系列テーブルにレコードを書き込みつつ、Event Bus にも publish する(=既存スキーマに「実機が出すであろう値」と同じ形で流す)。

---

## 7. UI

### 画面構成(1 画面、2 カラム)

- **左カラム(70%):チャット**
  - 上部:タブで「Shared」+ 発生中の DM チャネル
  - メッセージは発言者アバター・役職名・タイプ(propose/question/decision)バッジ付き
  - ツール呼び出しは折りたたみで表示(根拠の透明性)
- **右カラム(30%):ステータス**
  - Cell-A / Cell-B カード(ジョブ名、進捗 %、ステータス、ワイヤ残量プログレスバー)
  - ジョブキュー(次の 3 件)
  - 意思決定タイムライン(過去の承認・コミット履歴)
- **モーダル:承認(L3 用)**
  - 提案内容・根拠・影響範囲を表示
  - 「承認 / 却下 / 詳細を質問」ボタン

### スタイル方針
- ダーク基調(現場感)、状態色のみ強調(green / amber / red)
- 工場フロアの 3D 俯瞰図は POC では省略

---

## 8. デモシナリオ S-3 タイムライン

### 初期状態(T+0 直前)

| Cell | Job | 部材 | 進捗 | 残時間 |
|---|---|---|---|---|
| Cell-A | J-101 船舶用ブラケット | Inconel 625 | 240/720 層 | 約 36 h |
| Cell-B | J-102 プロペラ補修部品 | SUS316L | 180/200 層 | 約 8 h |

- ワイヤ:Cell-A に Inconel 625 残 2.1 kg、補充スプール在庫あり
- スケジュール待機:J-103(納期厳しい)、J-104(余裕あり)

### イベント 1(T+0):ワイヤ残量低下

- Injector 発火:`wire_low(cell=A, current_kg=2.1, predicted_empty_at=T+5h)`
- 期待される議論:
  1. **Operator** が議題提起
  2. **Material** が在庫・補充リードタイム回答(別スプール準備可、ただし途中交換)
  3. **Welding Eng.** が途中交換時のビード接合品質を評価(条件付き OK)
  4. **Planner** が代替案比較:
     - 案 a:Cell-A 低速モードで継続 + 5h 後にスプール交換
     - 案 b:J-101 を一時停止、J-103 を Cell-B 完了後に Cell-A で投入
  5. **QA** が途中交換時の追加検査要件を提示
  6. **Supervisor** が **L3 承認**(案 a を提案)
- L2 状態更新:Cell-A の `ramp_rate` 変更、5h 後の交換予定が `schedule` にコミット

### イベント 2(T+15min):装置故障

- Injector 発火:`robot_fault(cell=B, error_code=E-407, est_diag_min=20)`
- 期待される議論(再協議):
  1. **Operator** が異常検知、安全停止
  2. **Maintenance** がアラームコード照合 → ロボットアーム軸エンコーダ異常、過去事例から復旧見込み 3-6 時間
  3. **Planner** が Cell-B のジョブ J-102 をどうするか + 先ほどの Cell-A 低速計画も再検討
  4. **Welding Eng.** が J-102 を Cell-A に移管する場合のパラメータ差吸収可否を評価 → Cell-A は Inconel 用、SUS316L 移管はワイヤ・シールドガス変更が必要 → **不可** と判定
  5. **Material** がワイヤ・ガス変更不可を確認
  6. **QA** が J-102 中断・再開時の検査要件を提示
  7. **Supervisor** が **L3 承認**(J-102 は Cell-B 復旧待ち、Cell-A は計画通り低速継続、J-103 は Cell-B 復旧後に投入)
- L2 状態更新:Cell-B `status=down`、`schedule` 再生成

### デモ尺
- 全体 8〜12 分
- 注入は 2 点のみ、議論ターンは初期 5〜8 ターン × 2 シーン

---

## 9. 技術スタック

| 領域 | 採用 |
|---|---|
| LLM | Anthropic Claude(claude-opus-4-7 主軸、必要に応じ claude-sonnet-4-6 でコスト最適化) |
| Agent Framework | LangGraph |
| Backend Language | Python 3.12 |
| Backend Framework | FastAPI(REST + WebSocket) |
| Frontend | Next.js 14(App Router)+ React + TypeScript + Tailwind CSS |
| DB | TimescaleDB(既存スキーマ + sim tables) |
| Event Bus | Python asyncio Queue ベース(POC 内蔵) |
| Container | Docker Compose(local dev) |
| Test | pytest(backend)、Playwright(e2e demo replay) |

---

## 10. 自律性と意思決定境界

| 決定種別 | レベル |
|---|---|
| 議論内のターン継続・話題転換 | L1(議論のみ、状態不変) |
| パラメータ微調整、診断クエリ | L2(自動コミット) |
| スケジュール変更、ワイヤ交換指示 | L2(全エージェント合意要) |
| ジョブ中止、装置間移管、納期影響あり | **L3**(人間承認モーダル) |
| 安全関連(エラーコード重大度高) | **L3**(必須) |

L3 のとき、Supervisor は具体提案 + 根拠 + 代替案を提示する。人間は「承認 / 却下 / 詳細を質問」を選択。質問選択時はチャットに追加質問が流れ、議論が再開する。

---

## 11. マイルストーン(〜 2026/7/13、約 11 週)

| 期間 | フェーズ | 完了基準 |
|---|---|---|
| Week 1(〜 5/3) | 環境構築・基礎設計 | リポ・モノレポ・Docker Compose・CI 雛形、TimescaleDB スキーマ確認、sim tables 設計、LangGraph PoC(2 エージェント往復) |
| Week 2-3(〜 5/17) | エージェント基盤 | Agent base class、Tool 抽象、Event Bus、Sim State Manager、7 エージェントの role prompt 初版、tool 配線、単体テスト |
| Week 4-5(〜 5/31) | 協調プロトコル | 共有チャネル + DM、Supervisor 主導フロー、B-1 単独シナリオがバックエンド単独で完走 |
| Week 6-7(〜 6/14) | UI・統合 | Next.js + WebSocket、チャット / ステータス / L3 モーダル、B-1 シナリオ UI 込みで完走 |
| Week 8(〜 6/21) | B-3 実装 | 装置故障イベント、Maintenance/Planner/Welding Eng の追加ロジック、B-3 単独完走 |
| Week 9(〜 6/28) | S-3 連鎖 | B-1 → B-3 連鎖、メタ的再協議、議論品質チューニング |
| Week 10(〜 7/5) | デモ磨き込み | ステークホルダー試写、ストーリーテリング、フォールバック設計 |
| Week 11(〜 7/12) | リハ・予備 | 7/13 当日リハ、バックアップ録画 |

---

## 12. リスクと対策

| リスク | 対策 |
|---|---|
| 議論が収束しない / 無限往復 | ターン上限、Supervisor 強制集約、フォールバック決定の用意 |
| ドメイン知識(WAAM 欠陥モード等)の薄さ | 短い RAG ドキュメントを Week 2 までに作成、現場 SME レビュー |
| Claude API レイテンシで議論がもたつく | 並列ツール呼び出し、提案は構造化出力で短縮、Supervisor 以外を claude-sonnet-4-6 へ |
| Next.js + WebSocket のリアルタイム描画不具合 | Week 6 前半で骨組みだけ通す疎通テスト |
| TimescaleDB 既存スキーマと sim 設計の不整合 | Week 1 でスキーマ突合、不整合は sim tables 側で吸収 |
| デモ当日にライブで失敗 | 録画バックアップ + シナリオ再現性テスト(Playwright) |

---

## 13. 主要な未決事項(実装計画段階で詰める)

- TimescaleDB の既存スキーマ詳細(カラム名・型・粒度)との突合
- 各エージェントの Knowledge に入れる具体的な WAAM 欠陥モード・船級規格の量と粒度
- DM チャネルの開設トリガー(自動 vs 明示宣言)
- ステークホルダーレビュー日程(Week 9〜10 のいつか)
- デモ環境(クラウド vs ローカル、誰が何を見せるか)

---

## 14. 用語

| 用語 | 意味 |
|---|---|
| WAAM | Wire Arc Additive Manufacturing。金属ワイヤを用いたアーク式付加製造 |
| B-1 | サブシナリオ:造形中のワイヤ残量低下 |
| B-3 | サブシナリオ:装置故障(ロボット異常停止など) |
| S-3 | デモシナリオ:B-1 → B-3 の連鎖 |
| L1 / L2 / L3 | 自律レベル(議論のみ / 自動コミット / 人間承認) |
| DfAM | Design for Additive Manufacturing(造形性を考慮した設計) |
