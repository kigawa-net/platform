# platform

kigawa.net インフラの Kubernetes マニフェストリポジトリです。ArgoCD による GitOps で管理し、`main` ブランチへのマージが自動デプロイをトリガーします。

## 関連リポジトリ

| リポジトリ | 説明 |
|-----------|------|
| [kigawa-net/kigawa-net-k8s](https://github.com/kigawa-net/kigawa-net-k8s) | 既存のKubernetesマニフェストリポジトリ（keruta, lp 等が稼働中） |
| [kigawa-net/infra](https://github.com/kigawa-net/infra) | インフラ基盤の設定・プロビジョニング（物理・仮想マシン等） |

> **注記**: このリポジトリは `kigawa-net-k8s` とは独立した新規リポジトリとして作成した。既存アプリの移行は行っておらず、`kigawa-net-k8s` は引き続き稼働中。両リポジトリの関係（このリポジトリに一本化するか、新規アプリのみここに置くか等）は今後整理する。

## アーキテクチャ概要

```
GitHub (main branch)
    └── ArgoCD (apps/apps-app.yml)
            └── apps/ 配下の Application リソースを再帰的に同期
```

- **ルートアプリ**: `apps/apps-app.yml` — すべての子 Application を管理し、`platform` AppProject を定義
- **同期ポリシー**: 自動同期 + prune 有効
- **プロジェクト権限**: `platform-*` にマッチする Namespace、および `https://github.com/kigawa-net/*` 配下のリポジトリを許可

既存の `kigawa-net-k8s` が使う `kigawa-net` AppProject / `kigawa-net-*` Namespace とは重複しない、独立した名前空間を使う（同一クラスタ上でリソース競合を避けるため）。

## 新しいアプリの追加方法

1. マニフェストディレクトリを作成: `<service-name>/main/`（環境が複数ある場合は `<service-name>/<env>/`）
2. ArgoCD Application リソースを作成: `apps/<service-name>-<env>-app.yml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-<service>-<env>-app
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/kigawa-net/platform.git
    targetRevision: main
    path: <service>/<env>
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: platform-<service>-<env>
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

3. `apps/apps-app.yml` が `apps/` を再帰的に読むため、追加ファイルだけで自動認識される

## デプロイトリガーの規則

このリポジトリに置くアプリは、原則として以下の2環境構成で運用する（`main` という名前の本番専用環境は用意しない）。

| トリガー | 環境 | Namespace |
|---------|------|-----------|
| アプリ本体リポジトリで PR を作成・更新 | **dev** | `platform-<service>-dev` |
| アプリ本体リポジトリの `main` へマージ | **stg** | `platform-<service>-stg` |

- **dev環境はPRごとに独立させない**（単一の共有dev環境に、最後にpushされたPRの内容を上書きデプロイする）
- **stg環境が実質的に各アプリの唯一の稼働環境**（別途の本番環境が必要になった場合は `main` 環境を追加した3環境構成に拡張する。Namespace例: `platform-<service>-main`）

## 命名規則

| 種別 | パターン | 例 |
|------|---------|-----|
| Namespace（stg） | `platform-<service>-stg` | `platform-lipl-stg` |
| Namespace（dev） | `platform-<service>-dev` | `platform-lipl-dev` |
| ArgoCD Application名 | `platform-<service>-<env>-app` | `platform-lipl-stg-app` |

`kigawa-net-k8s` 側の規約（Ingress Class `haproxy`、レジストリ `harbor.kigawa.net`、Secret管理 = Bitwarden Secrets Manager、DB = mariadb-operator、ストレージクラス `rook-cephfs`/`rook-ceph-rbd`）は同一クラスタ上の設定のため、このリポジトリでも同じものを使う。詳細は `kigawa-net-k8s` のREADME・CLAUDE.mdを参照。

## デプロイ済みアプリ

| アプリ | ソースリポジトリ | dev | stg |
|--------|-----------------|-----|-----|
| Lipl | [kigawa-net/lipl](https://github.com/kigawa-net/lipl) | `lipl/dev/`（`platform-lipl-dev`、`lipl-dev.kigawa.net`） | `lipl/stg/`（`platform-lipl-stg`、`lipl.kigawa.net`） |

Liplの詳細な設計は [kigawa-net/lipl の docs/infrastructure.md](https://github.com/kigawa-net/lipl/blob/main/docs/infrastructure.md) を参照。CIから`develop-<sha>`/`main-<sha>`タグでイメージをpush後、`lipl/dev/`・`lipl/stg/`配下のDeployment manifestのimageタグを更新してこのリポジトリへコミットする運用（詳細は `lipl` リポジトリの `.github/workflows/`）。

### 前提として未整備の項目（デプロイ実行前に対応が必要）

- `harbor-registry`（イメージpull用Secret）が `platform-lipl-dev` / `platform-lipl-stg` namespaceに未作成
- `bitwarden-sec`（Bitwarden同期用トークン）を `kigawa-system/secret-provider/bitwarden-sync-crn.yaml` の `TARGET_NAMESPACES` に追加していない
- `apps/lipl-dev-app.yml` / `apps/lipl-stg-app.yml` はまだArgoCDに登録されていない（`kubectl apply` が必要。上記「新しいアプリの追加方法」参照）
- `lipl-dev.kigawa.net` / `lipl.kigawa.net` のDNS解決（`*.kigawa.net` ワイルドカードの実在確認）が未確認

## セットアップ（初回のみ・手動）

このリポジトリ自体をArgoCDに認識させるには、ルートアプリケーションを一度だけ手動で登録する必要がある。

```bash
kubectl apply -f apps/apps-app.yml
```

以降は `apps/` 配下へのファイル追加だけで新しいアプリが自動的に同期される。
