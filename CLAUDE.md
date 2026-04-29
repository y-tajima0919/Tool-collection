# CLAUDE.md

このリポジトリで作業する Claude Code 向けの作業ガイド。

## このプロジェクト

League of Legends カスタム戦用のチーム分けツール（HTML 単体）。  
本体: [team-divider/team-divider.html](team-divider/team-divider.html) — 単一HTMLでブラウザだけで動く。  
最新の状態と未解決事項は [HANDOVER.md](HANDOVER.md) を参照。  
バージョン履歴は [team-divider/CHANGELOG.md](team-divider/CHANGELOG.md) に記録。

## 技術スタック

- HTML / CSS / JavaScript（**単一ファイル構成**、ビルドプロセスなし）
- Bootstrap 5.3.3 / Bootstrap Icons 1.11.3 / Google Fonts (Cinzel, Noto Sans JP) — すべて CDN
- 永続化: localStorage（key: `lol_td_players`、`SCHEMA_VERSION` で互換管理）

## 基本ルール

### 編集して反映する流れ

1. `team-divider.html` を編集
2. 機能追加・仕様変更なら `CHANGELOG.md` に v X.Y エントリを追記し、HTML の `vX.Y` バージョン表記を 3箇所更新（`<title>` / ヘッダ / class名なし）
3. 必要なら `HANDOVER.md` を更新（現在の状態・既知の罠など）
4. git commit + push origin main
5. **本番反映**: `scp team-divider/team-divider.html sakura:www/noeyxy/tools/team-divider.html`
   - SSH エイリアス `sakura` は `~/.ssh/config` に設定済み
   - 本番URL: https://noeyxy.com/tools/team-divider.html
6. ドキュメントだけの変更（CHANGELOG/HANDOVER/README）は scp 不要

### 単一ファイル構成を維持する

- 外部 JS/CSS ファイルを増やさない
- 必要なら CDN から追加するが、慎重に
- ヘッダの `<style>` と末尾の `<script>` セクションに集約

## コードベースの全体像

`team-divider.html` は次の順序で構成。`═══` セクションコメントが目印。

| セクション | 主な内容 |
|---|---|
| State / Constants | グローバル状態、`LANES`, `LOCK_LABELS`, `MODE_LABELS`, `SCHEMA_VERSION` 等 |
| Utilities | `escapeHtml`, `shuffle`, `findPlayer`, getter (`getSelectedMode` 等), `toast` |
| Lane Selector | レーン選択UI・編集モーダル・**`assignLanes` / `assignLanesPaired`** |
| Player CRUD | `addPlayer` / `deletePlayer` / `cycleRank` / `cycleLockGroup` / トグル系 |
| Player Stats | **キャッシュ越し**の `getPlayCount` / `getWinCount` 等。`invalidateStatsCache()` で破棄 |
| Round Mutation Helpers | **`commitRoundChange()` / `swapInRound()` / `optimizationScore()`** |
| Round Generation | `sortForBalance` / `sortForRotation` / `generateRound` / `setWinner` / `rerollRound` |
| Team Balancing | `balanceByRankCount` / `balanceByRankSum` / `consolidateLockGroups` / `optimizeSameRankSwaps` / `wouldSplitLockGroup` |
| Scoring | `calcSameTeamScore` / `calcRankDiff` / `calcRankSum` / `getWinRate` / `calcWinRateScore` |
| D&D / Tap-to-Swap | `onDragStart` / `onDrop` / `onSwapTap` |
| Rendering | `renderList` / `renderCurrent` / `renderPlayerRow` / `renderHistory` / `renderWinnerContainer` |
| Storage | `saveData` / `loadData` / `migrateData` / `clearSave` |

## 重要な共通ヘルパー（必ずこれを使う）

ラウンドを変更したら **`commitRoundChange(round, opts)`** を呼ぶ。これは:
1. `invalidateStatsCache()`
2. `saveData()`
3. `updateGenerateButtonState()`
4. `renderCurrent` / `renderHistory` / `renderList`
5. 重複ラウンド検出 → トースト

を一括で行う。手動で `saveData → render...` を並べてはいけない（漏れる）。

プレイヤーのスワップは **`swapInRound(round, srcLoc, srcId, dstLoc, dstId)`** で行う。
ID 入替・`selectedIds` 再計算・勝者クリア・レーン位置スワップを内包。

チーム分けのスワップ最適化スコアは **`optimizationScore(team1, team2)`** に統一。
（過去の同チームペア + 勝率差 × 2、勝敗判定モード時のみ後者が効く）

統計取得は `getPlayCount/getSpecCount/getWinCount/getLossCount/getLastPlayedIndex` を使う。
内部で `_statsCache` (`Map<id, stats>`) からの返却。
**`players` / `rounds` を変更したら `invalidateStatsCache()` を呼ぶ**（commitRoundChange 内では自動）。

## 既知の罠（必ず読む）

### 内部レシオ値と表示値のズレ

- `player.rank` は **0, 1, 2** で管理（初級/中級/上級時代の名残）
- 表示は `RANK_LABELS = ['1','2','3']` で 1, 2, 3 に変換
- 合計値の計算では `rank + 1` を使う（`calcRankSum` 参照）
- 配列インデックスとして使われる箇所多数（`groups[player.rank]` 等）
- 統一は長期 TODO（HANDOVER.md 参照）。**触るときは慎重に**

### localStorage 保存

- **`SAVE_KEY = 'lol_td_players'` を変えてはいけない**（既存ユーザーのデータが消える）
- スキーマ拡張時は `SCHEMA_VERSION` をインクリメントし `migrateData()` にマイグレーションを追加
- 後方互換: 旧データには既定値を補完（lockGroup, lanes, winner, rounds 等）

### バランス選出の数学的制約

- 5v5 でレシオ差0にするには **構成の合計が偶数** である必要（`sortForBalance` 内のハード制約）
- 均等レシオ分け時はさらに **各レシオの人数が偶数** が必要
- 違反する構成を選ぶと差分が必ず出る

### スネークドラフトの差蓄積

- `balanceByRankSum` は **2パス構成**: Pass1 で差最小化 → Pass2 で同チームペア最小化
- Pass2 は Pass1 の差を**悪化させない**範囲でのみスワップする（`targetDiff` 参照）

### スワップ最適化はロックグループを跨がない

- `optimizeSameRankSwaps` / Pass2 では `wouldSplitLockGroup` ガードあり
- 新しい最適化を追加する場合も同ガードを通すこと

## モードと用語

| 用語 | 意味 |
|---|---|
| 人数バランス (`rank_count`) | 各レシオの人数を両チーム均等に分配 |
| 戦力バランス (`rank_sum`) | スネークドラフトでレシオ合計値を均等に |
| バランス（選出方式） | レシオ均衡優先、連戦許可 |
| ローテーション（選出方式） | 全員均等参加優先 |
| 選出傾向 | バランス時の構成偏り（均等/低レシオ優先/高レシオ優先） |
| ★優先 | 次ラウンド必ず参加 |
| 👁待機 | 今ラウンド欠席 |
| 🔒ロックグループ | 同グループは必ず同チーム |
| 担当レーン | TOP/JG/MID/ADC/SUP の希望レーン（複数可） |
| 勝敗判定モード | 勝者入力で勝率均等化 |

## やってはいけないこと

- **SAVE_KEY の変更**: 既存ユーザーの参加者データ・履歴が消える
- **schemaVersion なしでスキーマ変更**: マイグレーションが効かなくなる
- **既存ラウンドの破壊**: `rounds` 内の round オブジェクトのキー削除（renderHistory が落ちる）
- **`player.rank` の値域 0,1,2 を破る**: 配列インデックスが狂う（統一する場合は別タスクで全体改修）
- **`commitRoundChange` をバイパス**: saveData / 再描画 / キャッシュ無効化が漏れる
- **`scp` のパスを変える**: 本番は `sakura:www/noeyxy/tools/team-divider.html` 固定

## ドキュメント運用

- **CHANGELOG.md**: バージョン毎の変更履歴。機能追加・仕様変更時は必ず追記
- **HANDOVER.md**: 現在の状態・既知の罠・関数表。大きな変更時に同時更新
- **CLAUDE.md** (本ファイル): プロジェクト規約・作業手順。慣習が変わったら更新
- **README.md**: 利用者向けの機能紹介。バージョン番号と機能リストを最新に

## ユーザーの傾向（観察ベース）

- 直接的なフィードバックを好む。冗長な説明より要点を簡潔に
- 不正確な情報には鋭く指摘するので、推測より検証ベースで動く
- バグ報告は具体的な再現手順または **スクリーンショット** で来ることが多い
- リファクタリングや表記統一の依頼を時々挟むので、コードの整然さを保つ
- 「全部やって良いよ」など大胆な指示も出るが、影響範囲が広い変更は事前に方針を提示してから進める
