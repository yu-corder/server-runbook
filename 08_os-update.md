# 08 OSアップデート

[目次へ戻る](README.md)

セキュリティ更新は unattended-upgrades で自動適用されるが、カーネル更新の反映（再起動）は手動で行う。

## 手順

```bash
# 1. 更新できるパッケージを確認
sudo apt update
apt list --upgradable

# 2. 更新
sudo apt upgrade
# linux-image が保留されたら
sudo apt full-upgrade

# 3. 再起動が必要か確認
cat /var/run/reboot-required 2>/dev/null
cat /var/run/reboot-required.pkgs 2>/dev/null

# 4. DB を正常に停止してから再起動
cd /var/www/my-wp-blog
docker compose -f docker-compose.prod.yml stop -t 60 wp-db
docker logs wp-mariadb-prod --tail 2   # 「Shutdown complete」を確認
sudo reboot
```

## 再起動後の確認

```bash
uname -r                                     # カーネルが新しくなったか
lsb_release -ds                              # OSのバージョン
docker ps                                    # コンテナが起動しているか
curl -sI https://yu-syumilog.com/ | head -1  # サイトが表示されるか
docker exec saunas-nginx nginx -t            # Nginx の設定が正常か
systemctl is-active cloudflare-iptables      # active（80/443 の Cloudflare 限定が入ったか）
modprobe -n -v sctp | tail -1                # install /bin/false（→ 02）
cat /var/run/reboot-required 2>/dev/null     # 何も出ない
```

`restart: always` のコンテナは、手動で停止していても Docker の起動時に自動で起動する。停止しておきたいコンテナは再起動ポリシーを `no` にしておく（→ [06](06_sauna-rally.md)）。

再起動の前に DB を停止するのは、ホストの停止時に Docker がコンテナを待つのは10秒だけで、MariaDB が正常に停止しきれない可能性があるため。同じバージョンならクラッシュリカバリで起動できるが、次にバージョンを上げるときに「Upgrade after a crash is not supported」で止まる原因になる（→ [99](99_decision-log.md)）。

## 更新中に出る画面

| 画面 | 選ぶもの |
| --- | --- |
| 設定ファイルを置き換えるか（sshd_config など） | 自分で編集したものは「keep the local version currently installed」 |
| console-setup の文字セット | 「Guess optimal character set」 |

## メジャーアップグレードについて

24.04 → 26.04 のような LTS 間のアップグレードは `do-release-upgrade` で行う。24.04 のサポートは2029年まであるため、急がず、バックアップを取ったうえで時間に余裕のあるときに行う。
