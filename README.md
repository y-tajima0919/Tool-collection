# Tool-collection

個人利用向けの Web ツール集です。  
各ツールは HTML 単体で動作し、サーバー不要でブラウザだけで使えます。

## ツール一覧

| ツール | 説明 | バージョン |
|--------|------|-----------|
| [LoL Team Divider](./team-divider/team-divider.html) | League of Legends 用チーム分けツール | v1.17 |

---

## LoL Team Divider

参加者をバランスよく 2 チームに分けるためのツール。  
本番URL: https://noeyxy.com/tools/team-divider.html

### 主な機能

- 参加者登録（最大 20 人、レシオ 3 段階：1 / 2 / 3）
- レシオはクリックで変更可能
- 担当レーン (TOP / JG / MID / ADC / SUP) を複数選択可能、ラウンド生成時に最適割当
- ★優先参加 / 👁待機 フラグ
- 🔒ロックグループ（A/B/C）：同じグループのプレイヤーを必ず同じチームに配属
- ドラッグ＆ドロップ または タップ2回 でチーム⇔待機者を手動入替
- 勝敗判定モード：勝者を入力して勝率を均等化（ON/OFF切替可）
- 過去ラウンドと同じチーム構成（Blue/Red 逆転含む）を検出し警告
- 待機者の自動ローテーション（次ラウンドで優先参加）
- リロール機能（モード変更・待機者を含めて再抽選オプション付き）
- 参加者データ・ラウンド履歴の自動保存（localStorage）

### 使い方

`team-divider.html` をブラウザで開くだけで動作します。

### モードと用語

| 用語 | 意味 |
|---|---|
| 人数バランス (`rank_count`) | 各レシオの人数を両チーム均等に分配。同レシオが対面レーンに来るよう最適化 |
| 戦力バランス (`rank_sum`) | スネークドラフトでレシオ合計値を均等に分配 |
| バランス（選出方式） | レシオ均衡優先、連戦許可 |
| ローテーション（選出方式） | 全員均等参加優先 |
| 選出傾向 | バランス時の構成偏り（均等／低レシオ優先／高レシオ優先） |
| ★優先 | 次ラウンド必ず参加 |
| 👁待機 | 今ラウンド欠席 |
| 🔒ロックグループ | 同グループは必ず同チーム（A/B/C 最大3グループ） |
| 担当レーン | TOP/JG/MID/ADC/SUP の希望レーン（複数可、空＝全レーンOK） |
| 勝敗判定モード | 勝者入力で勝率均等化（OFF時は機能無効） |

### 技術ノート

- 内部ではレシオを `0, 1, 2` で管理（初級/中級/上級時代の名残）
- 表示上は `1, 2, 3` に変換し、合計値の計算には `rank + 1` を使用
- 永続化キー: `lol_td_players`（変更不可、`SCHEMA_VERSION` でマイグレーション管理）

## 技術スタック

- HTML / CSS / JavaScript（単一ファイル構成、ビルドプロセスなし）
- Bootstrap 5.3.3 / Bootstrap Icons 1.11.3 / Google Fonts (Cinzel, Noto Sans JP) — すべて CDN

## ファイル構成

```
Tool-collection/
├── README.md                       # このファイル
├── CLAUDE.md                       # Claude Code 用作業ガイド（軽量）
├── HANDOVER.md                     # プロジェクト引き継ぎ資料（関数表・既知の罠）
├── .gitignore
└── team-divider/
    ├── team-divider.html           # 本体
    └── CHANGELOG.md                # バージョン履歴
```

詳細な開発情報は [HANDOVER.md](HANDOVER.md)、変更履歴は [team-divider/CHANGELOG.md](team-divider/CHANGELOG.md) を参照。
