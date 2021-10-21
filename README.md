# Cloud Deploy による継続的デリバリー

GitHub Actions を CI、Cloud Deploy を CD 基盤とした CI/CD 環境を作ります。

## 1. GitHub リポジトリの用意

このリポジトリを fork してください。

## 2. Google Cloud に実行環境を作成

利用する機能を有効化します。

```bash
gcloud services enable cloudresourcemanager.googleapis.com compute.googleapis.com \
    container.googleapis.com serviceusage.googleapis.com stackdriver.googleapis.com \
    monitoring.googleapis.com logging.googleapis.com clouddeploy.googleapis.com \
    cloudbuild.googleapis.com artifactregistry.googleapis.com
```

コンテナのリポジトリを Artifact Registry に作り

```bash
gcloud artifacts repositories create cd-demo \
    --repository-format=docker --location=asia-northeast1 \
    --description="Docker repository for CI/CD demos"
```

実行環境として GKE クラスタを 1 つ作成します。

```bash
gcloud container clusters create cd-demo --zone asia-northeast1-a \
    --release-channel stable --machine-type "e2-standard-4" \
    --num-nodes 1 --preemptible
```

GitHub に渡すサービスアカウントと、鍵を生成します。

```bash
gcloud iam service-accounts create sa-cd-demo
PROJECT_ID=$(gcloud config get-value project)
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:sa-cd-demo@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.admin"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:sa-cd-demo@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.writer"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:sa-cd-demo@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/clouddeploy.releaser"
PROJECT_NUMBER="$( gcloud projects list --filter="${PROJECT_ID}" \
    --format='value(PROJECT_NUMBER)' )"
gcloud iam service-accounts add-iam-policy-binding \
    ${PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
    --member="serviceAccount:sa-cd-demo@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"
gcloud iam service-accounts keys create credential.json \
    --iam-account=sa-cd-demo@${PROJECT_ID}.iam.gserviceaccount.com
cat credential.json
```

## 3. GitHub Actions の Secrets に鍵などを登録

GitHub から Google Cloud 上のリソースにアクセスするための変数をセットします。

- GOOGLECLOUD_PROJECT_ID: プロジェクト ID
- GOOGLECLOUD_SA_KEY: 4 の最後に出力された JSON 鍵

## 4. Cloud Deploy のパイプラインを作成

clouddeploy.yaml を作り

```bash
cat << EOF >deploy/clouddeploy.yaml
apiVersion: deploy.cloud.google.com/v1beta1
kind: DeliveryPipeline
metadata:
  name: kustomize-pipeline
serialPipeline:
  stages:
  - targetId: dev
  - targetId: prod
    profiles: ["prod"]
---
apiVersion: deploy.cloud.google.com/v1beta1
kind: Target
metadata:
  name: dev
gke:
  cluster: projects/${PROJECT_ID}/locations/asia-northeast1-a/clusters/cd-demo
---
apiVersion: deploy.cloud.google.com/v1beta1
kind: Target
metadata:
  name: prod
gke:
  cluster: projects/${PROJECT_ID}/locations/asia-northeast1-a/clusters/cd-demo
EOF
```

Cloud Deploy のパイプラインを作成します。

```bash
gcloud beta deploy apply --file deploy/clouddeploy.yaml --region us-central1
```

### Cloud Deploy の設定

deploy 以下にアプリケーションのビルドやデプロイに関する定義があります。

- [skaffold.yaml](https://github.com/google-cloud-japan/appdev-cicd-handson/blob/main/cloud-deploy/sample-resources/kustomize/skaffold.yaml): ビルド対象は src 以下、デプロイは Kustomize で実施することを定義
- [deploy/k8s/base](https://github.com/google-cloud-japan/appdev-cicd-handson/tree/main/cloud-deploy/sample-resources/kustomize/deploy/k8s/base): prod プロファイルを指定しない限りはこちらがデプロイされる
- [deploy/k8s/overlays/prod](https://github.com/google-cloud-japan/appdev-cicd-handson/tree/main/cloud-deploy/sample-resources/kustomize/deploy/k8s/overlays/prod): prod プロファイル指定時にはこれ以下が base にマージされる

## 5. GitHub Actions の設定

main ブランチの変更により、Cloud Deploy にリリースが作成される Action を定義します。

```bash
mkdir -p .github/workflows
cat << EOF >.github/workflows/release.yaml
name: Release
on:
  push:
    branches:
      - main
env:
  GOOGLECLOUD_REGION: "asia-northeast1"
  CLOUDDEPLOY_REGION: "us-central1"
jobs:
  lint-code:
    name: Lint code
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Lint
      uses: golangci/golangci-lint-action@v2
      with:
        version: v1.42
        working-directory: src
        skip-go-installation: true
  lint-template:
    name: Test templates
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Install Skaffold
      run: |
        curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
        chmod +x skaffold && sudo mv skaffold /usr/local/bin
        skaffold version
    - name: Rendering
      run: |
        skaffold render --digest-source='none' -o prod.yaml --profile prod
    - name: Kubeval k8s manifests
      uses: azure/k8s-lint@v1
      with:
        manifests: |
            prod.yaml
  release:
    name: Release
    needs: 
      - lint-code
      - lint-template
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Setup gcloud
      uses: google-github-actions/setup-gcloud@master
      with:
        project_id: \${{ secrets.GOOGLECLOUD_PROJECT_ID }}
        service_account_key: \${{ secrets.GOOGLECLOUD_SA_KEY }}
        export_default_credentials: true
    - name: Setup credential helper
      run: gcloud auth configure-docker "\${{ env.GOOGLECLOUD_REGION}}-docker.pkg.dev"
    - name: Install Skaffold
      run: |
        curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
        chmod +x skaffold && sudo mv skaffold /usr/local/bin
        skaffold version
    - name: Build & Push
      run: skaffold build --default-repo '\${{ env.GOOGLECLOUD_REGION}}-docker.pkg.dev/\${{ secrets.GOOGLECLOUD_PROJECT_ID }}/cd-demo' --push --file-output=build.out
    - name: Archive the build result
      uses: actions/upload-artifact@v2
      with:
        name: build-result
        path: build.out
    - name: Make a release
      run: |
        gcloud components install beta
        gcloud beta deploy releases create "git-\${GITHUB_SHA::7}" --region \${{ env.CLOUDDEPLOY_REGION }} --delivery-pipeline=kustomize-pipeline --build-artifacts=build.out --annotations="commitId=\${GITHUB_SHA},author=\${GITHUB_ACTOR},date=\$(date '+%Y-%m-%d %H:%M:%S')"
EOF
```

## 6. GitHub への push（パイプラインの起動）

```bash
git add --all
git commit -m "add ci/cd templates"
git push origin main
```

## 7. Cloud Deploy の dev 環境の様子を確認

GitHub Actions の状況や  
Cloud Deploy パイプラインの状況、  
https://console.cloud.google.com/deploy/delivery-pipelines

GKE のワークロードの変化を確認してみてください。  
https://console.cloud.google.com/kubernetes/workload

## 8. プロモーション

本番環境にも同じリリースをロールアウトしてみます。

```bash
gcloud beta deploy releases promote --delivery-pipeline=kustomize-pipeline \
    --release=git-$(git rev-parse --short HEAD) --region=us-central1
```

## 10. クリーンアップ

```bash
gcloud beta deploy delivery-pipelines delete kustomize-pipeline --force \
    --region us-central1 --quiet
gcloud artifacts repositories delete cd-demo --location=asia-northeast1 --quiet
gcloud container clusters delete cd-demo --zone asia-northeast1-a --quiet
```
