# Team Divider - Claude Code 引き継ぎ資料

最終更新: 2026-04-29（v1.12 D&D入替機能追加）

## プロジェクト概要

参加者をバランスよく 2 チームに分けるための Web ツール。  
LoL のカスタム対戦用に作り始めたが、汎用化済み。

- **リポジトリ**: Tool-collection
- **現在のバージョン**: v1.12（2026-04-29、D&D入替追加）
- **構成**: 単一HTMLファイル（外部CDNのみ依存）
- **ブランチ**: main
- **本番URL**: https://noeyxy.com/tools/team-divider.html

## ファイル構成

```
Tool-collection/
├── README.md                       # リポジトリ全体の説明
└── team-divider/
    ├── team-divider.html           # 本体（単一ファイル構成）
    └── CHANGELOG.md                # 更新ログ
```

**注意**: README内の参照パスは `./lol-team-divider/team-divider.html` のままなので、`./team-divider/team-divider.html` に修正が必要。

## 技術スタック

- HTML / CSS / JavaScript（単一ファイル、ビルド不要）
- Bootstrap 5.3.3（CDN）
- Bootstrap Icons 1.11.3（CDN）
- Google Fonts: Cinzel / Noto Sans JP

ビルドプロセス・パッケージ管理なし。ブラウザで開けば動く。

## 主要機能の全体像

### 参加者管理
- 最大20人、レシオ3段階（表示は 1/2/3）
- 各プレイヤーに ★優先 / 👁待機 フラグ
- localStorage で自動保存
- レシオバッジクリックでレシオ変更（1→2→3→1 サイクル）

### チーム分けモード
- **均等レシオ分け**（標準）: 各レシオの人数が両チーム均等
- **レシオ合計均等**（スネークドラフト）: 合計値が均等

### 選出方式（誰をプレイヤーに、誰を待機者にするか）
- **バランス優先**（デフォルト）: レシオの均衡を重視、連戦許可
- **ローテーション優先**: 全員が均等に参加することを重視

### 選出傾向（バランス優先時のみ）
- **低レシオ優先**（デフォルト）: 低レシオの人を多く選出
- **高レシオ優先**: 高レシオの人を多く選出

### 最低1名制約
- 各レシオから最低1名は選出する（デフォルトON）
- バランス・ローテーション両方で有効

### リロール
- 現在のチームをシャッフルし直す
- チェックON: 待機者も含めて完全に再抽選（generateRound と同じロジック）
- チェックOFF: 現在のチームメンバーだけで再シャッフル
- 👁待機チェック者は常に除外、★優先者は必ず参加

### ロックグループ（v1.11〜）
- 各プレイヤーに `lockGroup` フィールド（0=未指定, 1=A, 2=B, 3=C）
- 同じグループは同チーム強制
- チーム分け後に `consolidateLockGroups` で寄せる（多数派側に集約）
- スワップ最適化（`optimizeSameRankSwaps` / `balanceByRankSum` Pass 2）には
  `wouldSplitLockGroup` ガードあり
- 寄せきれない場合（グループ人数 > 5、複数グループ衝突）はトースト警告
- **既知の制限**: 選出（プレイヤー/待機者の振り分け）はロックグループを考慮していない
  - 同グループのうち一部だけ選出される可能性あり（必要なら ★優先 で全員確保）

### ドラッグ＆ドロップ入替（v1.12〜）
- ラウンド生成後、BLUE / RED / 待機者 のプレイヤーをD&Dで入替可能
- HTML5 Drag and Drop API を使用（外部ライブラリなし）
- ID スワップ方式: drop 先のプレイヤーと src の ID を入替（チーム人数自動維持）
- `onDragStart / onDragOver / onDrop` 等は inline ハンドラ、状態は `dragState` 変数
- 入替後に `findDuplicateRound` を再実行
- **既知の制限**: 入替でロックグループが分断される可能性あり（手動操作のため意図と判断、警告なし）

### その他
- 過去ラウンドと同じ組み合わせ（Blue/Red 逆転含む）を検出して警告
- 同チームペア重複の最小化最適化

## 重要な技術的注意点

### ⚠ 内部レシオ値と表示値のズレ

**最重要**: コード内では `rank` を `0, 1, 2` で管理している（初期実装時の「初級/中級/上級」の名残）。
表示は `RANK_LABELS = ['1', '2', '3']` で 1, 2, 3 に変換している。

合計値の計算では `rank + 1` を使う必要がある（`calcRankSum` 参照）。

**影響箇所**:
- `players[].rank` は 0, 1, 2
- `RANK_LABELS[rank]` で表示
- `RANK_BADGE_CLASSES[rank]` でCSS適用（rb-0, rb-1, rb-2）
- `groups[player.rank]` のように配列インデックスとして使用される箇所多数
- `calcRankDiff` は内部値で計算（5v5なら相殺されるので問題なし）
- `calcRankSum`（表示用）のみ `rank + 1` で計算

根本対応する場合、配列インデックス・グルーピング・スワップロジック全てに影響するため慎重に。

### localStorage キー

`SAVE_KEY = 'lol_td_players'` のまま（変更すると既存ユーザーのデータが消える）。

### バランス選出のアルゴリズム

`sortForBalance` の核心部分:

1. ★優先者を先に確保
2. 残り枠の構成（c0=レシオ1の人数, c1=レシオ2, c2=レシオ3）を全列挙
3. ハード制約でフィルタ:
   - レシオ合計が偶数（0差分割の必要条件）
   - 均等レシオ分け時: 各レシオの人数が偶数
   - 最低1名制約（ON時）
4. バイアスで重み付けランダム選択:
   - 低: `weight = base^(c0*3 + c1)`
   - 高: `weight = base^(c2*3 + c1)`
   - 均等レシオ時 base=2（強い偏り）、スネークドラフト時 base=1.05（バリエーション重視）
5. 制約満たす候補が0なら制約緩和でフォールバック

### スネークドラフトの最適化

`balanceByRankSum` は2パス構成:

- **Pass 1**: レシオ差の最小化スワップのみ。差0達成で即終了
- **Pass 2**: Pass1 で達成した差を上限に、同チームペア最小化

旧実装は差を +1 まで悪化許容していたため差が蓄積していた問題を修正済み。

## コード構造

### State / Constants（先頭部分）
```javascript
let players, rounds, selectedRank, nextPlayerId
const TEAM_SIZE = 5
const RANK_LABELS, RANK_BADGE_CLASSES, MODE_LABELS, SELECTION_LABELS, BIAS_LABELS
```

### 主要関数

| 関数名 | 役割 |
|---|---|
| `addPlayer / deletePlayer / togglePriority / toggleBreak / cycleRank / cycleLockGroup` | プレイヤーCRUD |
| `getPlayCount / getSpecCount / getLastPlayedIndex` | 統計 |
| `sortForRotation / sortForBalance` | 選出ロジック |
| `generateRound / rerollRound` | ラウンド生成・リロール |
| `balanceByRankCount / balanceByRankSum` | チーム分けアルゴリズム |
| `optimizeSameRankSwaps` | 同チームペア最適化 |
| `consolidateLockGroups / wouldSplitLockGroup` | ロックグループ整合化 |
| `calcSameTeamScore / calcRankDiff / calcRankSum` | スコアリング |
| `findDuplicateRound` | 重複ラウンド検出 |
| `renderList / renderCurrent / renderHistory` | 描画 |
| `saveData / loadData / clearSave` | localStorage |

セクションコメント `═══` で区切られているので追いやすいはず。

## 次回作業候補

優先度順：

1. **READMEのパス修正**（最優先・即対応可）
   - [README.md:10](README.md#L10) のリンク `./lol-team-divider/team-divider.html` → `./team-divider/team-divider.html`
   - リポジトリ名のリネーム後、参照のみ取り残されている

2. **内部レシオ値の根本対応**（任意）
   - `rank` を 0,1,2 から 1,2,3 へ変更
   - 影響範囲広いので全関数の見直し必要
   - 既存の localStorage データ（`SAVE_KEY = 'lol_td_players'`）は互換性維持が必要

3. **チーム人数の可変化**（ゆくゆく）
   - 現在 `TEAM_SIZE = 5` 固定（[team-divider.html:318](team-divider/team-divider.html#L318)）
   - 4v4, 3v3 等への対応

4. **レシオ段階数の可変化**（ゆくゆく）
   - 現在3段階固定
   - 5段階等への対応
   - `RANK_LABELS`, `RANK_BADGE_CLASSES`, CSS全般、構成列挙ロジックに影響

## 過去にハマった罠

開発を引き継ぐ上で知っておくと良い、過去にバグが頻発した箇所：

### リロール周りは要注意
- 過去に同じバグを繰り返し直していた経緯あり
- specs（待機者）の算出を間違えると、待機者が消失して二度と戻らないバグになる
- specs は「プールからのスライス」ではなく「チームに入らなかった全員」から算出する原則
- ★優先者がシャッフルで後ろに回ると枠不足で待機に落ちるので、優先者を先に確保する必要あり

### バランス優先のレシオ差
- レシオ合計が奇数の構成を選ぶと、どうやっても5v5でレシオ差0は不可能（数学的制約）
- 構成選出の段階で「合計が偶数」を保証する必要がある
- 最初これに気付かず、チーム分け側で頑張ろうとして失敗していた

### スネークドラフトの差蓄積
- 同チームペア最適化のスワップでレシオ差の悪化を許容すると、毎回 +1 ずつ蓄積する
- 必ず2パス構成にして、差を先に確定させてからペア最適化する

### HTMLのdivタグ
- セクションを移動する時は開閉タグの数が合っているか確認すること
- `python3 -c "c=open('team-divider.html').read(); print(c.count('<div'), c.count('</div>'))"` で確認できる

## 開発履歴の確認方法

- `CHANGELOG.md`: 全バージョンの変更履歴
- v1.00 以前の経緯も「v1.00 以前の開発履歴」セクションに残してある

## ユーザーの傾向

- 直接的なフィードバックを好む
- 不正確な情報には鋭く指摘するので、推測より検証ベースで動く
- バグ報告は具体的な再現手順付きで来ることが多い
- リファクタリングや表記統一の依頼を時々挟む
