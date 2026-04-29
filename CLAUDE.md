# CLAUDE.md

このリポジトリでの作業ガイド（Claude Code 向け）。

## 文脈の入手先

- **[README.md](README.md)** — ツール概要・モードと用語・技術ノート
- **[HANDOVER.md](HANDOVER.md)** — 関数表・既知の罠・現在の状態
- **[team-divider/CHANGELOG.md](team-divider/CHANGELOG.md)** — バージョン毎の変更履歴

## 編集 → 反映フロー

1. `team-divider.html` を編集
2. 機能変更時は `CHANGELOG.md` に新バージョンを追記し、HTML の `vX.Y` 表記を更新
3. `git add → commit → push origin main`
4. **本番反映**: `scp team-divider/team-divider.html sakura:www/noeyxy/tools/team-divider.html`
   - SSH エイリアス `sakura` は `~/.ssh/config` に設定済み
   - ツール一覧ページ更新時: `scp index.html sakura:www/noeyxy/tools/index.html`
   - ドキュメントだけの変更（README/HANDOVER/CHANGELOG）は scp 不要

## 必ず使う共通ヘルパー

ラウンドを変更したら **`commitRoundChange(round, opts)`** を呼ぶ。  
`invalidateStatsCache → saveData → updateGenerateButtonState → render系 → 重複検出` を一括実行。  
**手動で `saveData → render...` を並べてはいけない**（漏れる）。

プレイヤー入替は **`swapInRound(round, srcLoc, srcId, dstLoc, dstId)`** で行う。  
ID 入替・selectedIds 再計算・勝者クリア・レーン位置スワップを内包。

チーム分けスワップ最適化のスコアは **`optimizationScore(team1, team2)`** に統一。

統計取得は `getPlayCount/getSpecCount/getWinCount/getLossCount/getLastPlayedIndex`。  
内部キャッシュ越しなので、**`players`/`rounds` を変更したら `invalidateStatsCache()` を呼ぶ**（commitRoundChange 内では自動）。

## やってはいけないこと

- **`SAVE_KEY = 'lol_td_players'` を変える** → 既存ユーザーのデータが消える
- **`schemaVersion` なしでスキーマ変更** → マイグレーションが効かなくなる。`migrateData()` に分岐を追加すること
- **`commitRoundChange` をバイパス** → save / 再描画 / キャッシュ無効化が漏れる
- **`player.rank` の値域 0,1,2 を破る** → 配列インデックスが狂う（統一は別タスクで全体改修）
- **scp パスの変更** → 本番は `sakura:www/noeyxy/tools/team-divider.html` 固定

詳細な罠（バランス選出のレシオ偶数制約、Pass2 の targetDiff 維持、ロックグループ分断防止など）は HANDOVER.md 参照。

## ユーザーの傾向

- 直接的なフィードバックを好む。冗長な説明より要点を簡潔に
- 不正確な情報には鋭く指摘するので、推測より検証ベースで動く
- バグ報告はスクリーンショットで来ることが多い
- 「全部やって良いよ」など大胆な指示も出るが、影響範囲が広い変更は事前に方針を提示してから進める
