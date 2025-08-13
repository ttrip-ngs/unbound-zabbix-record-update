# CLAUDE.md - Zabbix HA DNS自動更新システム

## プロジェクト概要

Zabbix HAクラスタのアクティブノード切り替えを検知し、UnboundのDNSレコードを自動更新するシステム。
開発・テストはDockerコンテナで行い、本番環境はローカルPythonまたはDockerで実行可能。

## 開発環境規則

### Docker利用ルール

- **開発・テスト時はDocker環境を使用**
- 開発時のローカルPython直接実行は禁止
- テスト、デバッグはDockerコンテナを使用
- pip installなどのパッケージ管理もコンテナ内で実施
- **本番環境はローカルPython実行またはDockerコンテナを選択可能**
- **DockerイメージはPython 3.9ベースで構築**（互換性確保）

### コンテナ構成

1. **開発用コンテナ** (`dev-updater`)
   - Python開発環境
   - デバッグツール含む
   - ボリュームマウントでソースコード編集

2. **テスト環境コンテナ群**
   - Zabbix Server HA1 (`zabbix-ha1`)
   - Zabbix Server HA2 (`zabbix-ha2`)
   - PostgreSQL (`postgres`)
   - Unbound DNS (`unbound`)
   - Updaterテスト用 (`test-updater`)

3. **本番用（オプション）** 
   - ローカルPython実行（推奨）
   - Dockerコンテナ実行も可能

## システム要件

### 動作環境要件

- **Python**: 3.9以上（Linux標準搭載バージョンを考慮）
  - RHEL/Rocky/AlmaLinux 9: Python 3.9
  - Ubuntu 20.04 LTS: Python 3.8（要3.9アップグレード）
  - Ubuntu 22.04 LTS: Python 3.10
  - Debian 11: Python 3.9
  - Debian 12: Python 3.11
- **依存ライブラリ**: 標準ライブラリを優先使用し、外部依存を最小限に

### 機能要件

1. **Zabbix HA監視**
   - Zabbix 6.0以降のHA APIを使用
   - 複数ノード（2台以上）対応
   - アクティブノードの定期的な確認

2. **DNS自動更新**
   - unbound-controlによるレコード更新
   - ローカルUnboundサーバーの制御
   - A/AAAAレコードの動的更新

3. **設定管理**
   - YAML形式の設定ファイル
   - 環境変数によるオーバーライド対応
   - Docker Secretsによる認証情報管理

4. **監視・ログ**
   - rsyslog連携
   - 詳細ログのファイル出力
   - エラー時のリトライ機構

### 非機能要件

- systemd timerによる定期実行（本番環境）
- docker-composeによる開発環境
- Kubernetes対応を考慮した設計

## ディレクトリ構造

```
/
├── docker/
│   ├── dev/
│   │   └── Dockerfile           # 開発用コンテナ
│   ├── prod/
│   │   └── Dockerfile           # 本番用コンテナ
│   └── test/
│       ├── zabbix/
│       │   └── Dockerfile       # Zabbix HAテスト用
│       └── unbound/
│           └── Dockerfile       # Unboundテスト用
├── src/
│   ├── zabbix_dns_updater.py   # メインスクリプト
│   ├── lib/
│   │   ├── __init__.py
│   │   ├── zabbix_api.py       # Zabbix API操作
│   │   ├── unbound_controller.py # Unbound制御
│   │   └── logger.py           # ログ処理
│   └── requirements.txt
├── config/
│   ├── config.yaml.example     # 設定ファイル例
│   └── .gitignore              # config.yamlを除外
├── scripts/
│   ├── docker-run-dev.sh       # 開発実行スクリプト
│   ├── docker-run-test.sh      # テスト実行スクリプト
│   └── docker-build.sh         # ビルドスクリプト
├── systemd/
│   ├── zabbix-dns-updater-local.service  # ローカルPython用
│   ├── zabbix-dns-updater-docker.service # Docker用
│   └── zabbix-dns-updater.timer
├── tests/
│   ├── unit/
│   └── integration/
├── docker-compose.yaml          # 開発・テスト環境
├── docker-compose.prod.yaml    # 本番環境用
├── Makefile                     # タスク自動化
└── docs/
    ├── SETUP.md                # セットアップ手順
    ├── CONFIGURATION.md        # 設定詳細
    └── TROUBLESHOOTING.md      # トラブルシューティング
```

## 設定ファイル仕様

```yaml
# config/config.yaml
system:
  mode: production  # production | development | test
  
zabbix:
  # 複数ノード対応
  nodes:
    - name: zabbix-ha1.example.local
      api_url: https://zabbix-ha1.example.local/api_jsonrpc.php
    - name: zabbix-ha2.example.local  
      api_url: https://zabbix-ha2.example.local/api_jsonrpc.php
  # 認証情報（Docker Secretsでも設定可）
  api_token: ${ZABBIX_API_TOKEN}
  # VIPホスト名
  vip_hostname: zabbix.example.local
  # タイムアウト設定
  timeout: 10

unbound:
  # unbound-controlパス
  control_command: /usr/sbin/unbound-control
  # 管理対象ゾーン
  zone: example.local
  # TTL設定
  ttl: 60
  # リモート制御（オプション）
  remote_host: null  # null or hostname
  remote_port: 8953

monitoring:
  # チェック間隔（秒）
  check_interval: 30
  # リトライ設定
  retry_count: 3
  retry_interval: 5
  # ヘルスチェック
  health_check_enabled: true
  health_check_port: 8080

logging:
  # ログレベル
  level: INFO  # DEBUG | INFO | WARNING | ERROR
  # syslog設定
  syslog:
    enabled: true
    facility: local0
    host: localhost
    port: 514
  # ファイルログ
  file:
    enabled: true
    path: /var/log/zabbix-dns-updater/updater.log
    max_size: 10485760  # 10MB
    backup_count: 5
```

## Docker実行方法

### 開発環境

```bash
# 開発環境の起動
make dev-up

# 開発コンテナでスクリプト実行
make dev-run

# テスト実行
make test

# ログ確認
make logs
```

### 本番環境

#### ローカルPython実行（推奨）

```bash
# Pythonバージョン確認（3.9以上必須）
python3 --version

# 依存パッケージインストール
pip3 install -r src/requirements.txt

# 設定ファイル配置
sudo cp config/config.yaml /etc/zabbix-dns-updater/

# systemdサービスとして実行
sudo make install-local
sudo systemctl start zabbix-dns-updater.timer
```

#### Dockerコンテナ実行（オプション）

```bash
# 本番イメージビルド
make build-prod

# systemdサービスとして実行（Docker版）
sudo make install-docker
sudo systemctl start zabbix-dns-updater.timer
```

## Makefile タスク

```makefile
.PHONY: dev-up dev-down dev-run test build-prod install-local install-docker clean

# 開発環境
dev-up:
	docker-compose up -d

dev-down:
	docker-compose down

dev-run:
	docker-compose exec dev-updater python /app/src/zabbix_dns_updater.py

# テスト
test:
	docker-compose run --rm test-updater pytest /app/tests/

test-integration:
	docker-compose run --rm test-updater pytest /app/tests/integration/

# 本番ビルド
build-prod:
	docker build -f docker/prod/Dockerfile -t zabbix-dns-updater:latest .

# インストール（ローカルPython版）
install-local:
	mkdir -p /etc/zabbix-dns-updater
	cp systemd/zabbix-dns-updater-local.service /etc/systemd/system/zabbix-dns-updater.service
	cp systemd/zabbix-dns-updater.timer /etc/systemd/system/
	systemctl daemon-reload

# インストール（Docker版）
install-docker:
	cp systemd/zabbix-dns-updater-docker.service /etc/systemd/system/zabbix-dns-updater.service
	cp systemd/zabbix-dns-updater.timer /etc/systemd/system/
	systemctl daemon-reload

# クリーンアップ
clean:
	docker-compose down -v
	docker rmi zabbix-dns-updater:latest || true
```

## 実装フロー

1. **初期化**
   - 設定ファイル読み込み
   - ログ設定
   - Zabbix API接続確認

2. **メインループ**
   ```
   while True:
       1. Zabbix HA状態取得（全ノード）
       2. アクティブノード特定
       3. 前回と比較
       4. 変更があれば:
          - Unboundレコード更新
          - 更新確認（dig/nslookup）
          - ログ記録
       5. 指定間隔待機
   ```

3. **エラーハンドリング**
   - API接続エラー → リトライ
   - DNS更新エラー → ロールバック試行
   - 継続不可能エラー → アラート送信

## テスト戦略

### 単体テスト
- Zabbix API操作のモック
- Unbound制御のモック
- 設定ファイル解析

### 統合テスト（Docker環境）
- Zabbix HAフェイルオーバーシミュレーション
- DNS更新の実動作確認
- エラーケースの再現

### 負荷テスト
- 高頻度チェックでの動作確認
- 大量ノードでの性能確認

## セキュリティ考慮事項

1. **認証情報管理**
   - Docker Secrets使用
   - 環境変数での上書き
   - 設定ファイルに直書き禁止

2. **通信セキュリティ**
   - Zabbix API: HTTPS必須
   - unbound-control: 証明書認証

3. **最小権限原則**
   - コンテナは非root実行
   - 必要最小限のケイパビリティ

## トラブルシューティング

### よくある問題

1. **Zabbix API接続失敗**
   - ネットワーク疎通確認
   - API トークン有効性
   - SSL証明書検証

2. **DNS更新失敗**
   - unbound-control権限
   - ゾーンファイル権限
   - 構文エラー確認

3. **Docker関連**
   - ボリュームマウント確認
   - ネットワーク設定
   - リソース制限

## 今後の拡張計画

- Kubernetes Operator化
- Prometheus メトリクス出力
- 複数DNS製品対応（BIND, PowerDNS）
- Web UI追加
- Webhook通知機能

## 開発ルール

1. **開発・テスト時のコード実行はDockerコンテナ内で行う**
2. **Python 3.9以上の機能のみを使用（型ヒントは3.9構文を使用）**
3. **git commit前に必ずテストを実行**
4. **設定変更は設定ファイルで管理**
5. **ログは構造化ログ（JSON形式）を推奨**
6. **エラーは適切にハンドリングし、リトライ機構を実装**
7. **外部ライブラリは最小限にし、可能な限り標準ライブラリを使用**