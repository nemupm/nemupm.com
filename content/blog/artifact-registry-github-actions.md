---
date: 2022-06-01T00:00:00+09:00
title: Google Artifact Registry へ GitHub Actions でキー無しでイメージをアップロード with Terraform
authors: ["nemupm"]
categories:
  - terraform
  - gcp
  - github-actions
slug: 2022/06/01/artifact-registry-github-actions
---

# 概要

- GitHub Actions で GCP Registry に Docker イメージをアップロードしたい
- GCP の Registry は最近は Artifact Registry 推奨らしい
- 認証は [Workload Identity 連携](https://cloud.google.com/blog/ja/products/identity-security/enable-keyless-access-to-gcp-with-workload-identity-federation) がセキュアらしい
  - サービスアカウントキーを発行せずに権限を付与 
- 全部 Terraform でやりたい

# Terraform で Registry とサービスアカウントを準備

### registry.tf

```hcl
resource "google_project_service" "artifactregistry" {
  service = "artifactregistry.googleapis.com"
}

resource "google_artifact_registry_repository" "default" {
  provider = google-beta

  location      = local.location
  format        = "DOCKER"
  description   = "default repository for docker"
  repository_id = "default"

  depends_on = [google_project_service.artifactregistry]
}

resource "google_service_account" "registry_admin" {
  account_id   = "registry-admin"
  display_name = "Service Account to manage images on registry"
}

resource "google_artifact_registry_repository_iam_member" "registry_admin" {
  provider = google-beta

  location   = google_artifact_registry_repository.default.location
  repository = google_artifact_registry_repository.default.name
  role       = "roles/artifactregistry.repoAdmin"
  member     = "serviceAccount:${google_service_account.registry_admin.email}"
}

output "registry_service_account_email" {
  description = "service account for registry"
  value       = google_service_account.registry_admin.email
}

output "registry_default_host" {
  description = "you may need to run 'gcloud auth configure-docker <this value>.'"
  value       = "${google_artifact_registry_repository.default.location}-docker.pkg.dev"
}

output "registry_default_image_name_prefix" {
  description = "image name prefix for default registry"
  value       = "${google_artifact_registry_repository.default.location}-docker.pkg.dev/${var.project_id}/${google_artifact_registry_repository.default.repository_id}"
}
```

### github_actions.tf

gh-oidcモジュールをお借りしています

```hcl
locals {
  github_repo = "user/repo"
}

module "gh_oidc" {
  source      = "terraform-google-modules/github-actions-runners/google//modules/gh-oidc"
  version     = "v3.0.0"
  project_id  = var.project_id
  pool_id     = "github-pool"
  provider_id = "github-pool-provider"
  sa_mapping = {
    (google_service_account.registry_admin.id) = {
      sa_name   = google_service_account.registry_admin.name
      attribute = "attribute.repository/${local.github_repo}"
    }
  }
}

output "github_actions_workload_identity_pool_provider" {
  description = "Workload Identity Pool Provider ID"
  value       = module.gh_oidc.provider_name
}
```

# GitHub Actionsの準備

## Secretの作成

- `IMAGE_PREFIX`
    - `registry_default_image_name_prefix`の値を使う
- `GOOGLE_IAM_WORKLOAD_IDENTITY_POOL_PROVIDER`
    - `github_actions_workload_identity_pool_provider`の値を使う
- `SERVICE_ACCOUNT_EMAIL`
    - `registry_service_account_email`の値を使う
- `REGISTRY_HOST`
    - `registry_default_host`の値を使う

## .github/workflows/upload_example_image.yml

```yml
name: Upload image to GCP registry
on:
  push:
    branches:
      - "main"
env:
  IMAGE: ${{ secrets.IMAGE_PREFIX }}/example_image
  TAG: latest
  GOOGLE_IAM_WORKLOAD_IDENTITY_POOL_PROVIDER: ${{ secrets.GOOGLE_IAM_WORKLOAD_IDENTITY_POOL_PROVIDER }}
  SERVICE_ACCOUNT_EMAIL: ${{ secrets.SERVICE_ACCOUNT_EMAIL }}
jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - name: Checkout the repository
        uses: actions/checkout@v3
      - id: auth
        uses: google-github-actions/auth@v0.8.0
        with:
          workload_identity_provider: "${{ env.GOOGLE_IAM_WORKLOAD_IDENTITY_POOL_PROVIDER }}"
          service_account: "${{ env.SERVICE_ACCOUNT_EMAIL }}"
      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v0.6.0
      - name: Authorize Docker push
        run: gcloud auth configure-docker ${{ secrets.REGISTRY_HOST }}
      - name: Build a docker image
        run: docker build -t ${{ env.IMAGE }}:${{ env.TAG }} -f Dockerfile .
        working-directory: ./docker/example_image
      - name: Push the docker image
        run: docker push ${{ env.IMAGE }}:${{ env.TAG }}
      - name: Clean up Container images
        run: |
          gcloud container images list-tags "${IMAGE}" \
            --filter="NOT tags:${TAG}" --format="get(digest)" | \
          while read digest
          do
            gcloud container images delete -q --force-delete-tags "${IMAGE}@$digest"
          done
```

# 感想

Workload Identity連携すごい便利（小並）。どんどん使って行きたい

# 参考にしたサイト

[Cloud RunをGithub ActionsとTerraformで管理する](https://zenn.dev/nnabeyang/articles/05ce98c4955123)
