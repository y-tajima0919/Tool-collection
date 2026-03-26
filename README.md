# Tool-collection

個人利用向けの Web ツール集です。  
各ツールは HTML 単体で動作し、サーバー不要でブラウザだけで使えます。

## ツール一覧

| ツール | 説明 | バージョン |
|--------|------|-----------|
| [LoL Team Divider](./lol-team-divider/lol-team-divider.html) | League of Legends カスタム対戦向けのチーム分けツール | v1.00 |

## LoL Team Divider

League of Legends の 5v5 カスタム対戦で、参加者をバランスよく 2 チームに分けるためのツールです。

### 主な機能

- 参加者登録（最大 20 人、ランク 3 段階：初級・中級・上級）
- 2 つのチーム分けモード（均等ランク分け / ランク合計均等）
- 過去ラウンドの同チームペア重複を最小化する最適化
- 観戦者の自動ローテーション（次ラウンド優先参加）
- ★優先参加 / 👁観戦フラグ
- リロール機能（モード変更・観戦者含めるオプション付き）
- 参加者データの自動保存（localStorage）

### 使い方

`lol-team-divider.html` をブラウザで開くだけで動作します。

## 技術スタック

- HTML / CSS / JavaScript（単一ファイル構成）
- Bootstrap 5.3.3
- Bootstrap Icons 1.11.3
- Google Fonts（Cinzel / Noto Sans JP）
