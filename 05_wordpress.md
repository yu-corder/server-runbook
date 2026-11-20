# 05 WordPress

[目次へ戻る](README.md)

## コンテナ

| コンテナ名 | イメージ | 公開ポート | restart |
| --- | --- | --- | --- |
| wp-app-prod | wordpress:7.1.5-php8.3-apache | 0.0.0.0:8080→80 | always |
| wp-mariadb-prod | mariadb:10.6 | なし | always |

WordPress のイメージはバージョンを固定している（→ [99 決定ログ](99_decision-log.md)）。

## ファイル構成

```
/var/www/my-wp-blog/
├── .env                     # DB接続情報など（Git管理外、権限600）
├── .env.example
├── docker-compose.yml       # 共通定義
├── docker-compose.prod.yml  # 本番用の上書き
├── security.ini             # PHP設定
├── deploy.sh                # デプロイスクリプト（→ 07）
├── wp-content/
│   ├── themes/cocoon-child/ # 子テーマ（Git管理）
│   ├── themes/cocoon-master/# 親テーマ（Git管理外）
│   ├── plugins/             # プラグイン（Git管理外）
│   ├── uploads/             # アップロード画像（Git管理外）
│   └── languages/           # 翻訳ファイル（Git管理）
└── wp-db-data/              # MariaDB のデータ（Git管理外）
```

WordPress 本体（`wp-admin/`、`wp-includes/` など）は、リポジトリにもバインドマウントにも含まれない。コンテナの `/var/www/html` は匿名ボリュームで、空のときだけイメージから本体がコピーされる（→ [WordPress 本体の更新](#wordpress-本体の更新)）。

## compose の構成

- `docker-compose.yml` に共通の定義を書き、`docker-compose.prod.yml` が `extends` で取り込む
- 本番用ファイルでは次の項目を上書きする
  - `restart: always`
  - `container_name`（wp-app-prod / wp-mariadb-prod）
  - `WP_HOME` / `WP_SITEURL`（`https://yu-syumilog.com`）
  - `WP_AUTO_UPDATE_CORE`（`false`。本体の自動更新を止める）
- DB のパスワードは compose に直接書かず、`.env` から `${MYSQL_PASSWORD}` の形で読み込む

## PHP の設定（security.ini）

| 設定 | 値 | 目的 |
| --- | --- | --- |
| session.upload_progress.enabled | 0 | セッションアップロード進捗機能を悪用した RCE のリスクを下げる |
| expose_php | Off | `X-Powered-By` ヘッダーで PHP のバージョンを出さない |

`security.ini` はコンテナにマウントしており、PHP は起動時に読み込む。変更したら [deploy.sh](07_deploy.md) を実行してコンテナを再起動する。

確認コマンド:

```bash
docker exec wp-app-prod php -i | grep -E 'session.upload_progress.enabled|expose_php'
curl -sI https://yu-syumilog.com/ | grep -i x-powered-by   # 何も出なければOK
```

## WordPress 本体の更新

本体のバージョンは、`docker-compose.yml` のイメージのタグだけで決める。管理画面からの本体の更新・自動更新は使わない（→ [99 決定ログ](99_decision-log.md)）。

```bash
# 手元で docker-compose.yml のタグを上げてコミット・push したあと、サーバーで
cd /var/www/my-wp-blog
./deploy.sh
docker exec wp-app-prod grep "wp_version =" /var/www/html/wp-includes/version.php
docker exec wp-app-prod php -v
```

- `/var/www/html` の匿名ボリュームにすでに本体があると、公式イメージは上書きしない。そのため deploy.sh では `--renew-anon-volumes` でボリュームを作り直し、新しいイメージの本体に入れ替える
- 作り直しても、`wp-content`（バインドマウント）と DB（`wp-db-data`）は消えない。`wp-config.php` と `.htaccess` は起動時に環境変数から作り直される
- `/var/www/html` の直下に手でファイルを置くと、次のデプロイで消える
- 自動更新を止めているので、WordPress のセキュリティリリースは自分でタグを上げない限り適用されない
- プラグインとテーマは `wp-content` にあるため、これまでどおり管理画面から更新する

## Git管理

| 対象 | Git | 理由 |
| --- | --- | --- |
| 子テーマ（cocoon-child） | 管理する | 自分で編集するコード |
| 翻訳ファイル（languages/） | 管理する | 本体・プラグインの更新で書き換わる |
| compose、deploy.sh、security.ini、.env.example | 管理する | 構成の定義 |
| 親テーマ（cocoon-master）、plugins/ | 管理しない | 配布物そのもの。管理画面から更新する |
| uploads/、wp-db-data/ | 管理しない | 投稿データ。ユーザー情報や2要素認証の秘密鍵を含むDBは公開しない |
| .env、backup/ | 管理しない | 秘密情報・バックアップ |

- 管理画面からプラグインや翻訳を更新すると翻訳ファイルが書き換わるので、サーバー上ですぐにコミットして push し、手元で pull する
