# TASKS.md - Zabbix HA DNS自動更新システム タスク管理

## プロジェクト概要
Zabbix HAクラスタのアクティブノード切り替えを検知し、UnboundのDNSレコードを自動更新するシステムの開発

## 現在のステータス
- 開始日: 2025-08-13
- フェーズ: 初期開発
- 進捗: 10% (仕様策定完了)

## 完了済みタスク ✅

### フェーズ1: 計画・設計
- [x] プロジェクト仕様策定
- [x] CLAUDE.md作成（プロジェクト仕様書）
- [x] システムアーキテクチャ設計
- [x] ディレクトリ構造設計
- [x] README.md作成
- [x] Gitリポジトリ初期化
- [x] GitHubリポジトリ作成

## 進行中タスク 🚧
なし

## 未着手タスク 📋

### フェーズ2: 基盤構築
- [ ] Docker環境構築（docker-compose.yaml作成）
- [ ] 開発用Dockerfile作成（docker/dev/Dockerfile）
- [ ] テスト環境用Dockerfile作成（Zabbix HA、Unbound）
- [ ] Makefile作成（タスク自動化）
- [ ] pre-commit設定（.pre-commit-config.yaml）

### フェーズ3: コア機能実装
- [ ] メインスクリプト実装（src/zabbix_dns_updater.py）
- [ ] Zabbix APIライブラリ作成（src/lib/zabbix_api.py）
- [ ] Unbound制御ライブラリ作成（src/lib/unbound_controller.py）
- [ ] ログ処理ライブラリ作成（src/lib/logger.py）
- [ ] 設定ファイルサンプル作成（config/config.yaml.example）

### フェーズ4: デプロイメント準備
- [ ] systemdサービス定義作成
  - [ ] zabbix-dns-updater-local.service
  - [ ] zabbix-dns-updater-docker.service
  - [ ] zabbix-dns-updater.timer
- [ ] 実行スクリプト作成
  - [ ] docker-run-dev.sh
  - [ ] docker-run-test.sh
  - [ ] docker-build.sh

### フェーズ5: テスト実装
- [ ] 単体テスト実装（tests/unit/）
  - [ ] test_zabbix_api.py
  - [ ] test_unbound_controller.py
  - [ ] test_logger.py
- [ ] 統合テスト実装（tests/integration/）
  - [ ] test_ha_failover.py
  - [ ] test_dns_update.py

### フェーズ6: CI/CD設定
- [ ] GitHub Actions CI/CD設定
  - [ ] Python品質チェック（ruff、mypy）
  - [ ] テスト自動実行
  - [ ] Dockerイメージビルド
  - [ ] セキュリティスキャン

### フェーズ7: ドキュメント作成
- [ ] SETUP.md（セットアップガイド）
- [ ] CONFIGURATION.md（設定詳細）
- [ ] TROUBLESHOOTING.md（トラブルシューティング）

## 優先度別タスク

### 優先度: 高 🔴
1. Docker環境構築（開発環境の基盤）
2. メインスクリプト基本実装
3. Zabbix APIライブラリ（コア機能）
4. Unbound制御ライブラリ（コア機能）

### 優先度: 中 🟡
1. テスト環境構築
2. 単体テスト実装
3. systemdサービス定義
4. pre-commit設定

### 優先度: 低 🟢
1. 統合テスト実装
2. GitHub Actions設定
3. 詳細ドキュメント作成

## 次のアクション

1. **Docker開発環境の構築**
   - docker-compose.yaml作成
   - 開発用Dockerfile作成
   - 基本的なPython環境セットアップ

2. **コア機能の実装開始**
   - メインスクリプトの骨組み作成
   - 設定ファイル読み込み機能
   - 基本的なログ出力

3. **Zabbix API連携**
   - HA状態取得機能
   - アクティブノード判定ロジック

## リスク・課題

### 技術的課題
- Zabbix APIトークンの安全な管理方法
- DNS更新失敗時のロールバック戦略
- 大規模環境でのスケーラビリティ

### 環境依存
- Python 3.9以上の要件（一部ディストリビューションで要アップグレード）
- unbound-control権限設定
- systemd timer精度

## メモ
- 開発はDockerコンテナ内で実施（CLAUDE.md記載のルール遵守）
- 標準ライブラリ優先使用で外部依存最小化
- エラーハンドリングとリトライ機構を重視

## 更新履歴
- 2025-08-13: 初版作成、フェーズ1完了