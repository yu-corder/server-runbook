# 07 デプロイ

[目次へ戻る](README.md)

## 手順

```bash
# 手元で変更を push したあと、サーバーで
cd /var/www/my-wp-blog
./deploy.sh
```

## deploy.sh の処理内容

| 順番 | 処理 | 目的 |
| --- | --- | --- |
| 1 | `git pull --ff-only origin main` | 最新を取得。履歴が食い違っていたら止める |
| 2 | `docker compose config -q` | compose の設定を検証する |
| 3 | `docker compose pull wordpress` | イメージを取得する |
| 4 | `docker compose up -d --no-deps wordpress` | 設定やイメージが変わっていればコンテナを作り直す |
| 5 | `docker compose restart wordpress` | PHP の設定（security.ini）を読み直させる |
| 6 | コンテナの稼働確認 | 起動していなければエラーで終了する |
| 7 | イメージ・PHP設定の表示 | 意図した設定になっているか目で確認する |

- `set -Eeuo pipefail` により、途中でエラーが出たらその場で止まる
- 手順5の再起動で、数秒だけサイトが止まる

## なぜ再起動が必要か

`up -d` は、イメージや compose の設定が変わったときにしかコンテナを作り直さない。マウントしている ini ファイルの中身を変えただけでは「変更なし」と判断されるため、明示的に再起動して PHP に設定を読み直させる。

また、Git がファイルを差し替えると、単体ファイルのバインドマウントでは古いファイルを見続けることがあり、これも再起動で解消される。

## 履歴が食い違ったとき

`git pull --ff-only` が失敗した場合は、サーバーと手元のどちらかで別のコミットが作られている。

```bash
git fetch
git log --oneline --graph main origin/main -5
```

で状況を確認してから対応する。サーバー側を GitHub に合わせる場合は、`git status` で未コミットの変更がないことを確認してから実行する。

```bash
git reset --hard origin/main
```
