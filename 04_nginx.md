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

cron で毎日 3:00 に更新スクリプトを実行している。

```
0 3 * * * /var/www/sauna-rally/renew-cert.sh >> /var/www/sauna-rally/renew.log 2>&1
```

結果は `/var/www/sauna-rally/renew.log` で確認する。
