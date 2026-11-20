# 04 リバースプロキシ / 証明書

[目次へ戻る](README.md)

## saunas-nginx

| 項目 | 内容 |
| --- | --- |
| イメージ | sauna-rally-nginx（自前ビルド） |
| ベースイメージ | alpine:3.24（`apk add nginx`） |
| Nginx | 1.30.7 |
| HTTP/2 | 無効（`listen` に `http2` を付けず、`http2 on;` も書かない） |
| 公開ポート | 80, 443 |
| restart | always |
| 管理元 | /var/www/sauna-rally |

もともと sauna-rally 用に立てた Nginx だが、現在は両ドメインの入口を担当している。

| ドメイン | 処理 |
| --- | --- |
| yu-syumilog.com | wp-app-prod へ転送（`http://host.docker.internal:8080`） |
| sauna-rally.com | サービス終了ページを返す |

## 注意点

sauna-rally の php コンテナは停止しているため、Nginx の設定に `saunas-php` への参照が残っていると、Nginx の再起動時に名前解決に失敗して起動できなくなる（＝ブログも止まる）。
設定を変えたら、反映前に必ず構文チェックを行う。

```bash
docker exec saunas-nginx nginx -t
docker exec saunas-nginx nginx -s reload
```

## WordPress への転送

saunas-nginx と wp-app-prod は別の Docker ネットワークにいるため、ホストの 8080 番を経由して転送する。
転送先にはホストの IP を書かず、`host.docker.internal` を使う。

```yaml
# docker-compose.prod.yml（nginx）
extra_hosts:
  - "host.docker.internal:host-gateway"
```

`host-gateway` はコンテナから見たホストのアドレスに置き換わる。通信はサーバーの中で完結し、パケットフィルタは通らない。

## HTTP/2 を無効にしている理由

Cloudflare との通信は HTTP/1.1 で足りるため、オリジンでは HTTP/2 を有効にしていない。
HTTP/2 の実装に脆弱性が見つかったときに影響を受けにくくなる（→ [99](99_decision-log.md)）。

```bash
# 1.1 と表示されれば無効
curl -sk --http2 -o /dev/null -w '%{http_version}\n' https://localhost/ -H 'Host: yu-syumilog.com'
```

## Nginx の更新

Nginx は Alpine のパッケージから入れているため、バージョンはベースイメージ（`docker-config/nginx/Dockerfile` の `FROM`）で決まる。
新しい Nginx が必要なときは、そのバージョンを含む Alpine に上げる。

```bash
# 1. 手元で Dockerfile の FROM を変更して push

# 2. サーバーで、ロールバック用に今のイメージを残してからデプロイ
cd /var/www/sauna-rally
docker tag sauna-rally-nginx sauna-rally-nginx:backup-<今のバージョン>
./deploy.sh

# 3. 確認
docker exec saunas-nginx nginx -v
docker exec saunas-nginx nginx -t
curl -sI https://yu-syumilog.com/ | head -1
curl -sI https://sauna-rally.com/ | head -1

# 4. deploy.sh で起動した sauna-rally のアプリと DB を止める（→ 06）
docker update --restart=no saunas-php saunas-mariadb
docker stop saunas-php saunas-mariadb
```

うまく動かないときは、残しておいたイメージに戻す。

```bash
docker tag sauna-rally-nginx:backup-<前のバージョン> sauna-rally-nginx
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --no-deps --no-build nginx
```

数日問題なく動いたら、バックアップのイメージは `docker rmi` で削除する。

## 証明書の更新

Let's Encrypt の証明書（yu-syumilog.com / sauna-rally.com）を、root の cron で毎日 3:00 に更新スクリプトを実行して更新している。

```
# sudo crontab -l
0 3 * * * /var/www/sauna-rally/renew-cert.sh >> /var/www/sauna-rally/renew.log 2>&1
```

### renew-cert.sh の処理

| 順番 | 処理 | 内容 |
| --- | --- | --- |
| 1 | `docker run --rm certbot/dns-cloudflare:v5.4.0 renew --no-random-sleep-on-renew` | 期限が近い証明書だけを更新する。Cloudflare の DNS で認証する |
| 2 | `docker exec saunas-nginx nginx -s reload` | Nginx に新しい証明書を読み込ませる |

- 証明書と certbot の設定はホストの `/etc/letsencrypt` にあり、コンテナにマウントして使う
- certbot は期限の30日前になるまで更新しない。それ以外の日は「not yet due for renewal」と出て何もしない
- 手順2の reload は毎日実行されるが、接続を切らずに設定を読み直すだけなのでサイトは止まらない

### 確認

```bash
tail -20 /var/www/sauna-rally/renew.log   # 各証明書の期限と、更新したかどうか
```

### certbot のイメージを消さない

`certbot/dns-cloudflare:v5.4.0` は更新のときだけ `docker run --rm` で起動するため、普段はコンテナがなく、`docker image ls` で使用中（`U`）の印が付かない。イメージを整理するときに消さないこと。消しても次の実行で再取得されるが、そのときに Docker Hub に繋がらないと更新に失敗する。

`certbot/certbot:latest` は使っていない（2026-11-20 に削除）。
