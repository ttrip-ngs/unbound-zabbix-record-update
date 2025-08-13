# Zabbix HA DNS自動更新システム

Zabbix HAクラスタのアクティブノード切り替えを検知し、UnboundのDNSレコードを自動更新するシステム

## 概要

本システムは、Zabbix 6.0以降のHA機能を使用している環境で、アクティブノードの切り替えを監視し、自動的にDNSレコードを更新することで、クライアントが常に正しいZabbixサーバーにアクセスできるようにします。

## 特徴

- Zabbix HA APIを使用したアクティブノード監視
- unbound-controlによる動的DNS更新
- 複数Zabbixノード対応（2台以上）
- 設定ファイルによる柔軟な構成
- systemd timerによる定期実行
- 詳細なログ出力とエラーハンドリング

## 動作要件

- Python 3.9以上
- Zabbix Server 6.0以上（HA構成）
- Unbound DNS Server
- Linux (RHEL/Rocky/AlmaLinux 9, Ubuntu 22.04 LTS, Debian 11以降推奨)

## クイックスタート

### 開発環境

```bash
# リポジトリのクローン
git clone https://github.com/ttrip-ngs/unbound-zabbix-record-update.git
cd unbound-zabbix-record-update

# 開発環境の起動
make dev-up

# テスト実行
make test
```

### 本番環境

```bash
# 依存パッケージのインストール
pip3 install -r src/requirements.txt

# 設定ファイルの配置
sudo cp config/config.yaml.example /etc/zabbix-dns-updater/config.yaml
sudo vi /etc/zabbix-dns-updater/config.yaml  # 環境に合わせて編集

# systemdサービスのインストール
sudo make install-local

# サービスの起動
sudo systemctl start zabbix-dns-updater.timer
sudo systemctl enable zabbix-dns-updater.timer
```

## 設定

設定ファイル（`/etc/zabbix-dns-updater/config.yaml`）の例：

```yaml
zabbix:
  nodes:
    - name: zabbix-ha1.example.local
      api_url: https://zabbix-ha1.example.local/api_jsonrpc.php
    - name: zabbix-ha2.example.local  
      api_url: https://zabbix-ha2.example.local/api_jsonrpc.php
  api_token: "your-api-token"
  vip_hostname: zabbix.example.local

unbound:
  control_command: /usr/sbin/unbound-control
  zone: example.local
  ttl: 60

monitoring:
  check_interval: 30
  retry_count: 3
```

## ドキュメント

- [セットアップガイド](docs/SETUP.md)
- [設定詳細](docs/CONFIGURATION.md)
- [トラブルシューティング](docs/TROUBLESHOOTING.md)
- [開発ガイド](CLAUDE.md)

## ライセンス

MIT License

## コントリビューション

Issue、Pull Requestを歓迎します。開発に参加する場合は[CLAUDE.md](CLAUDE.md)を参照してください。

## サポート

問題が発生した場合は、[Issues](https://github.com/ttrip-ngs/unbound-zabbix-record-update/issues)で報告してください。