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

## 命名規則

| 種別 | パターン | 例 |
|------|---------|-----|
| Namespace（本番） | `platform-<service>-main` | `platform-lipl-main` |
| Namespace（開発） | `platform-<service>-dev` | `platform-lipl-dev` |
| ArgoCD Application名 | `platform-<service>-<env>-app` | `platform-lipl-main-app` |

`kigawa-net-k8s` 側の規約（Ingress Class `haproxy`、レジストリ `harbor.kigawa.net`、Secret管理 = Bitwarden Secrets Manager、DB = mariadb-operator、ストレージクラス `rook-cephfs`/`rook-ceph-rbd`）は同一クラスタ上の設定のため、このリポジトリでも同じものを使う。詳細は `kigawa-net-k8s` のREADME・CLAUDE.mdを参照。

## セットアップ（初回のみ・手動）

このリポジトリ自体をArgoCDに認識させるには、ルートアプリケーションを一度だけ手動で登録する必要がある。

```bash
kubectl apply -f apps/apps-app.yml
```

以降は `apps/` 配下へのファイル追加だけで新しいアプリが自動的に同期される。
