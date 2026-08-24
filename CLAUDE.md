# CLAUDE.md

このファイルは Claude Code (claude.ai/code) がリポジトリで作業する際のガイダンスを提供します。
日本語で話す。

## Repository Overview

kigawa.net インフラの Kubernetes マニフェスト。ArgoCD による GitOps 管理。**このリポジトリ自身**の `main` へのマージが（ArgoCDの同期対象としての）自動デプロイをトリガーする。各アプリの実際の環境（dev/stg）を何がデプロイするかは、アプリ本体リポジトリ側のCI/CDがこのリポジトリの `<service>/dev/` または `<service>/stg/` を更新することで決まる（下記「デプロイトリガーの規則」参照）。

`kigawa-net-k8s`（既存の別リポジトリ）と同一クラスタを対象とするが、AppProject・Namespaceプレフィックスは独立している（`platform` / `platform-*`）。既存アプリ（keruta, lp 等）は引き続き `kigawa-net-k8s` 側で稼働中で、このリポジトリには含まれない。

## ArgoCD Application Structure

- **ルート**: `apps/apps-app.yml` — 全ての子 Application を再帰的に同期 + `platform` AppProject を定義
- **同期ポリシー**: 自動同期 + prune 有効
- **プロジェクト権限**: `platform-*` にマッチする Namespace、`https://github.com/kigawa-net/*` のリポジトリを許可

## 新しいアプリの追加

1. `<service>/dev/` と `<service>/stg/` にマニフェストを配置（本番専用の `main` 環境は原則用意しない）
2. `apps/<service>-dev-app.yml`, `apps/<service>-stg-app.yml` に ArgoCD Application を作成（`project: platform`、`namespace: platform-<service>-dev` / `platform-<service>-stg`）
3. コミット前に `kubectl apply --dry-run=client -f <manifest-file>` で検証する

## デプロイトリガーの規則（アプリ本体リポジトリ側のCI/CD）

このリポジトリ自体の `main` ブランチ（後述、GitOpsのトリガー）とは別に、各アプリの**本体リポジトリ**（例: `lipl`）側でのCI/CDは以下の規則に従う:

| トリガー | 環境 | Namespace |
|---------|------|-----------|
| アプリ本体リポジトリで PR を作成・更新 | **dev** | `platform-<service>-dev` |
| アプリ本体リポジトリの `main` へマージ | **stg** | `platform-<service>-stg` |

dev環境はPRごとに独立させず、単一の共有環境（最後にpushされたPRの内容で上書き）とする。stg環境が実質的に各アプリの唯一の稼働環境になる（本番環境が別途必要になった場合のみ `main` 環境を追加する3環境構成に拡張する）。

## 命名規則

- Namespace: `platform-<service>-dev` / `platform-<service>-stg`
- ArgoCD Application名: `platform-<service>-<env>-app`

## クラスタ共通設定（`kigawa-net-k8s` と同一クラスタのため共有）

- Ingress Class: `haproxy`
- ベースドメイン: `kigawa.net`
- コンテナレジストリ: `harbor.kigawa.net`（`private/<service>` または `library/<service>`）
- イメージタグ: 本番 `main-<commit-sha>`（`IfNotPresent`）、開発 `develop-<commit-sha>`（`Always`）
- Secret管理: Bitwarden Secrets Manager（`k8s.bitwarden.com/v1 BitwardenSecret` CRD）。新規Namespaceは `kigawa-net-k8s` 側の `kigawa-system/secret-provider/bitwarden-sync-crn.yaml` の `TARGET_NAMESPACES` にも追記が必要（`bitwarden-sec` 認証トークン同期のため）
- DB: アプリ専用の MariaDB を mariadb-operator（`k8s.mariadb.com/v1alpha1`）で構築。共有インスタンスは使わない
- ストレージクラス: `rook-cephfs`（共有ファイルシステム）、`rook-ceph-rbd`（ブロックデバイス）
- 認証: `user.kigawa.net`（Keycloak）

## デプロイ手順

```bash
# ドライラン検証
kubectl apply --dry-run=client -f <manifest-file>

# 同期状態確認
argocd app list
argocd app get <app-name>
```

> **注意**: `main` へのマージは即時デプロイを意味する。本番影響がある変更は十分に検証してからマージすること。

## 関連ドキュメント

- [README.md](README.md) — アーキテクチャ概要・命名規則
- [kigawa-net-k8s](https://github.com/kigawa-net/kigawa-net-k8s) — 既存の同クラスタ向けリポジトリ（規約の詳細はこちらを参照）
