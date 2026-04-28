# Prompt Template — WAAM POC 業務用 5 要素テンプレ

毎回 **Task** だけ書き換えて使う。Context / Constraints / Output / Guardrail は WAAM POC 業務の足場として固定。

---

## Context
- プロジェクト: WAAM(Wire Arc Additive Manufacturing)自律協調エージェント工場 POC
- ターゲット: 2026-07-13 デモ(全 11 週マイルストーン)
- 技術スタック: Python 3.12 / FastAPI / LangGraph / Anthropic Claude(claude-opus-4-7 主軸)/ Next.js 14 / TimescaleDB / Docker Compose
- 既存資産:
  - TimescaleDB 実スキーマ: `C:\dev\promethean\IPS\app\db\schema.sql`(`process_data` / `anomaly_events` / `camera_snapshots`)
  - 接続層参考実装: `C:\dev\promethean\IPS\app\db\timescale.py`
  - 上位計画: `C:\Users\koich\app-projects\warp-bootcamp2\plans.md`(全 14 章)
- ドメイン: 重工・船舶向け複雑形状金属部品の多品種少量(Inconel 625 / SUS316L など)
- 7 エージェント: Operator / Planner / Welding Engineer / Material Handler / Maintenance / QA / Supervisor
- 協調モデル: 共有チャネル + DM のハイブリッド、Supervisor 主導
- 自律レベル: L1(議論のみ)/ L2(自動コミット)/ L3(人間承認モーダル)
- 作業ディレクトリ: `C:\Users\koich\app-projects\warp-bootcamp2`(Windows + bash)
- コミュニケーション言語: 日本語

## Task
<!-- 毎回ここを埋める。例:
- 「Week N の実装計画を作成」
- 「<章番号> の設計を <具体的検討点> の観点で深掘り」
- 「<ファイルパス> をレビューして <観点> を改善」
- 「<バグ症状> の根本原因を特定して fix を提案」
-->

## Constraints
- スコープ境界を明記:〔Week N のみ / 特定章 / 特定モジュール / 特定ファイル群〕
- 既存資産優先:plans.md の記述・IPS リポの実装・既存 schema を **再利用**(新規実装は最後の手段)
- YAGNI:当該フェーズで使わないものは作らない(plans.md §2 Out-of-Scope を厳守)
- 既存 TimescaleDB(public schema)は変更不可、新規は `sim` schema 配下
- 自律レベル L3 相当の判断(納期影響・装置間移管・安全関連)は人間承認前提で設計
- 重要な前提が不明な場合は **仮定で進めず** AskUserQuestion(最大 4 択)

## Output
- 形式:Markdown(日本語)
- 計画系タスクの必須セクション:
  1. **Context**:なぜやるか(動機・問題)
  2. **設計判断**:plans.md との突合結果と確定事項
  3. **タスク分解**:ID 付き(例 `WK<n>-T<nn>`)、各タスクに〔成果物 / 依存 / 完了基準 / 見積〕
  4. **並列実行ストリーム**:依存グラフを ASCII で
  5. **重要ファイル一覧**:着手順 + 絶対/相対パス
  6. **検証手順**:`docker compose` / `pytest` / `psql` などの実コマンドで E2E 確認
  7. **YAGNI 確認**:意図的にやらないこと
  8. **残未決事項**:次フェーズ持ち越し
  9. **リスクと緩和**
- タスク粒度:着手見積 **0.5〜3 h**(これを超えるなら分解不足)
- ファイル参照は `path:line` 形式

## Guardrail
- plans.md §2 の Out-of-Scope(実機制御 / 認証 / 多並列 / 3D 俯瞰図 / 多言語化 など)を超える提案は **禁止**。必要なら「Out-of-Scope 拡張提案」として別枠で明示
- 既存 `process_data` / `anomaly_events` / `camera_snapshots` テーブルへの破壊的変更は禁止
- 計画段階で **コード変更しない**(plan mode 中は plan ファイルのみ編集可)
- 「とりあえず動かす」「あとで直す」を許容しない:完了基準を満たすか、明示的に未完として残す
- ユーザの判断が必要な分岐(納期・スコープ・優先度・予算)は勝手に決めない
- `claude-opus-4-7` / `claude-sonnet-4-6` / `claude-haiku-4-5` 以外のモデル ID を提案しない
