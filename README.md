# notes-deploy

[`liaooliver/notes`](https://github.com/liaooliver/notes) 的 GitOps manifest，**Argo CD 唯一的真實來源**。

這個 repo 裡只有部署設定，一行程式碼都沒有。

## 為什麼跟程式碼分開

Argo CD 官方 Best Practices 建議把「程式碼 repo」和「部署設定 repo」分開。除此之外，這個專案還有一個具體理由：

`notes` 的 `main` 與 `staging` 跑在 **classic branch protection** 上，required status check **沒有任何針對特定身分的豁免開關**。CI 產生的 image bump commit 不會觸發任何 check，所以那條規則必然成立——換 GitHub App token 或 PAT 都一樣失敗。

**這個 repo 刻意不設任何 branch protection**，CI 用 deploy key 直接 push 進 `main`。完整推導見 [`docs/gitops-roadmap.md` 第 2.2 節](https://github.com/liaooliver/notes/blob/main/docs/gitops-roadmap.md)。

## 結構

```
base/                共用的 k8s 資源
├── web-deployment.yaml
├── web-service.yaml
├── ingress.yaml
└── kustomization.yaml

overlays/            兩個環境靠「目錄」分，不是靠分支
├── staging/         → namespace notes-staging、host notes-staging.local
└── production/      → namespace notes-production、host notes.local

argocd/              Argo CD 的 Application 定義（kubectl apply -f argocd/）
```

`overlays/*/kustomization.yaml` 的 `images[].newTag` 由 `notes` 的 CI 自動覆寫，目前是佔位值 `sha-000…`。

## 誰會寫這個 repo

`notes` 的 `ci.yml` 裡的 `bump-staging` / `bump-production` 兩個 job，用 deploy key（`notes` 的 secret `DEPLOY_REPO_SSH_KEY`）push。commit 訊息長這樣：

```
chore(deploy): bump staging image to sha-<40 碼>
```

**不帶 `[skip ci]`** —— 這是另一個 repo，`notes` 的 workflow 不會被觸發，不需要那層保險。

## Rollback

```bash
git revert <那個 bump commit>
git push
```

這個 repo 不設保護，不必開 PR。Argo CD 偵測到 image tag 變回舊值就會自動 rolling update 回去。

「production 現在跑哪個版本？」：

```bash
git show main:overlays/production/kustomization.yaml
```

## ⚠️ 這個 repo 是 public：不要放 Secret

**永遠不要 commit k8s 的 `Secret` 資源。** `Secret` 的 `data` 欄位只是 base64 編碼，不是加密，任何人一行指令就還原。

目前的設計剛好不需要任何 Secret：`notes` 是 public repo → GHCR package 也是 public → 拉 image 不需要憑證。`web-deployment.yaml` 裡的 `imagePullSecrets` 維持註解狀態。

哪天真的需要（例如把 `notes` 改成 private），要先用 Sealed Secrets 或 SOPS 加密再進版。

## 本機驗證

```bash
kubectl kustomize overlays/staging
kubectl kustomize overlays/production
```

CI 的 `Manifest Check` workflow 每次 push 與 PR 都會跑同樣的檢查。
