# 02 ホスト初期設定

[目次へ戻る](README.md)

## OS

| 項目 | 内容 |
| --- | --- |
| ディストリビューション | Ubuntu 24.04.5 LTS（noble） |
| カーネル | 6.8.0-151-generic |
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

## 無効にしているカーネルモジュール

| モジュール | 設定ファイル | 理由 |
| --- | --- | --- |
| sctp | `/etc/modprobe.d/blacklist-sctp.conf` | 未修正の SCTP の脆弱性で、コンテナ内からホストの root を取られるのを防ぐ（→ [99](99_decision-log.md)） |

```
# /etc/modprobe.d/blacklist-sctp.conf
blacklist sctp
install sctp /bin/false
```

- `blacklist` はソケット作成時の自動読み込みを止め、`install ... /bin/false` は明示的な `modprobe sctp` も失敗させる
- カーネルはホストとコンテナで共通のため、ホストに置けばすべてのコンテナに効く
- 戻すときは設定ファイルを削除する

```bash
modprobe -n -v sctp | tail -1   # install /bin/false
lsmod | grep -E '^sctp'         # 何も出ない
```

## ディレクトリ構成

```
/var/www/
├── my-wp-blog/      # WordPress（→ 05）
└── sauna-rally/     # sauna-rally と Nginx（→ 04, 06）
```
