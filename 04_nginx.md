# 04 リバースプロキシ / 証明書

[目次へ戻る](README.md)

## saunas-nginx

| 項目 | 内容 |
| --- | --- |
| イメージ | sauna-rally-nginx（自前ビルド） |
| 公開ポート | 80, 443 |
| restart | always |
| 管理元 | /var/www/sauna-rally |

もともと sauna-rally 用に立てた Nginx だが、現在は両ドメインの入口を担当している。

| ドメイン | 処理 |
| --- | --- |
| yu-syumilog.com | wp-app-prod へ転送 |
| sauna-rally.com | サービス終了ページを返す |

## 注意点

sauna-rally の php コンテナは停止しているため、Nginx の設定に `saunas-php` への参照が残っていると、Nginx の再起動時に名前解決に失敗して起動できなくなる（＝ブログも止まる）。
設定を変えたら、反映前に必ず構文チェックを行う。

```bash
docker exec saunas-nginx nginx -t
docker exec saunas-nginx nginx -s reload
```

## 証明書の更新

cron で毎日 3:00 に更新スクリプトを実行している。

```
0 3 * * * /var/www/sauna-rally/renew-cert.sh >> /var/www/sauna-rally/renew.log 2>&1
```

結果は `/var/www/sauna-rally/renew.log` で確認する。
