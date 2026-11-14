# サーバー構成資料

個人で運用している VPS（yu-syumilog.com / sauna-rally.com）の構成と運用手順をまとめたもの。
ページは作業・理解の順に番号を振っている。

> 公開リポジトリのため、IPアドレス・パスワード・鍵などの秘密情報は書かない。
> 秘密情報は `.env`（Git管理外）とパスワードマネージャーで管理する。

## 目次

| # | ページ | 内容 |
| --- | --- | --- |
| 00 | [概要](00_top.md) | 全体構成図、運用サイト |
| 02 | [ホスト初期設定](02_host.md) | OS・Docker・自動更新 |
| 03 | [ネットワーク・アクセス制限](03_network.md) | Cloudflare、パケットフィルタ、待ち受けポート |
| 04 | [リバースプロキシ / 証明書](04_nginx.md) | saunas-nginx、証明書更新 |
| 05 | [WordPress](05_wordpress.md) | コンテナ構成、compose、PHP設定 |
| 06 | [sauna-rally](06_sauna-rally.md) | サービス終了後の状態 |
| 07 | [デプロイ](07_deploy.md) | deploy.sh の処理内容と手順 |
| 08 | [OSアップデート](08_os-update.md) | パッケージ更新・再起動の手順 |
| 99 | [決定ログ](99_decision-log.md) | 設計・運用上の判断とその理由 |

## 環境

| 項目 | 内容 |
| --- | --- |
| ホスティング | さくらのVPS |
| OS | Ubuntu 24.04.5 LTS |
| 実行基盤 | Docker / Docker Compose |
| 前段 | Cloudflare |

## 変更履歴

| 日付 | 内容 |
| --- | --- |
| 2026-10-23 | `session.upload_progress.enabled=0` を設定、WordPressイメージのバージョンを固定、deploy.sh を作成 |
| 2026-11-03 | Web（80/443）への接続を Cloudflare 経由に限定 |
| 2026-11-14 | `expose_php = Off` を設定、deploy.sh に再起動処理を追加 |
| 2026-11-14 | OSアップデート（24.04.4 → 24.04.5、カーネル 6.8.0-149） |
| 2026-11-14 | saunas-php / saunas-mariadb の自動起動を無効化 |
| 2026-11-14 | 資料を作成 |
