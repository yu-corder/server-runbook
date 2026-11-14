# 06 sauna-rally

[目次へ戻る](README.md)

## 現在の状態

サービスは終了しており、sauna-rally.com には Nginx がサービス終了ページを返している。

| コンテナ名 | イメージ | restart | 状態 |
| --- | --- | --- | --- |
| saunas-nginx | sauna-rally-nginx | always | 稼働（→ [04](04_nginx.md)） |
| saunas-php | sauna-rally-web | no | 停止 |
| saunas-mariadb | mariadb | no | 停止 |

## 停止の方法

再起動ポリシーが `always` のままだと、ホストの再起動時にコンテナが自動で起動してしまう。そのため、停止と合わせて再起動ポリシーも変更している。

```bash
docker update --restart=no saunas-php saunas-mariadb
docker stop saunas-php saunas-mariadb
```

## 注意点

- `docker update` で変えた設定はそのコンテナにだけ効く。sauna-rally のディレクトリで `docker compose up -d` を実行すると、compose ファイルの設定でコンテナが作り直され、自動起動が元に戻る
- php コンテナを止めた状態でも Nginx が起動できることを、`docker exec saunas-nginx nginx -t` で確認済み
