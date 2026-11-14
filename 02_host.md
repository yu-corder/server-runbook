# 02 ホスト初期設定

[目次へ戻る](README.md)

## OS

| 項目 | 内容 |
| --- | --- |
| ディストリビューション | Ubuntu 24.04.5 LTS（noble） |
| カーネル | 6.8.0-149-generic |
| 標準サポート期限 | 2029年（LTS） |

## Docker

| 項目 | バージョン |
| --- | --- |
| Docker Engine | 29.9.3 |
| Docker Compose | v5.7.1 |

各サービスは Docker Compose で管理し、ホストには直接ミドルウェアを入れない。

## 自動更新

`unattended-upgrades` を有効にしており、セキュリティ更新は自動で適用される。
ただし再起動は自動では行わないため、カーネル更新などは [08 OSアップデート](08_os-update.md) の手順で手動で反映する。

```bash
systemctl is-enabled unattended-upgrades   # enabled
```

## ディレクトリ構成

```
/var/www/
├── my-wp-blog/      # WordPress（→ 05）
└── sauna-rally/     # sauna-rally と Nginx（→ 04, 06）
```
