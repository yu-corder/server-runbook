# 03 ネットワーク・アクセス制限

[目次へ戻る](README.md)

## 方針

- Web への通信は Cloudflare を経由させ、オリジンサーバーの 80/443 番は Cloudflare 以外から接続できないようにする
- アクセス制限は OS のファイアウォールではなく、さくらのVPSのパケットフィルタで行う

## 待ち受けポート

| ポート | プロセス | 外部からの接続 | 用途 |
| --- | --- | --- | --- |
| 22 | sshd | 許可 | SSH |
| 80 / 443 | docker-proxy（saunas-nginx） | Cloudflare のみ | Web |
| 8080 | docker-proxy（wp-app-prod） | 遮断 | Nginx → WordPress の転送用 |

## Cloudflare

- DNS を Cloudflare で管理し、プロキシ（オレンジ雲）を有効にしている
- レスポンスの `Server` ヘッダーは Cloudflare によって `cloudflare` に置き換わる

## Dockerネットワーク

| 名前 | 所属コンテナ |
| --- | --- |
| my-wp-blog_default | wp-app-prod, wp-mariadb-prod |
| sauna-rally_default | saunas-nginx, saunas-php, saunas-mariadb |

## 確認コマンド

```bash
sudo ss -tlnp                                      # 待ち受けポート
curl -sI https://yu-syumilog.com/ | grep -i server # server: cloudflare
```
