# MacOSX mojaveにhalyardをインストールして、GKE上にSpinnakerをデプロイする
2019/01/29 改定

## 参考元
* Spinnaker公式：[Try out Halyard on GKE](https://www.spinnaker.io/setup/quickstart/halyard-gke/)
* Spinnaker公式：[Kubernetes Provider V2 (Manifest Based)](https://www.spinnaker.io/setup/install/providers/kubernetes-v2/)
* Spinnaker公式：[Configuring GCS Artifact Credentials](https://www.spinnaker.io/setup/artifacts/gcs/)

## Mac の準備
### gcloudをインストールする
  [Cloud SDKのインストール](https://cloud.google.com/sdk/downloads#interactive)にある手順に従い、gcloudをインストールする。

  gcloudを認証してデフォルトのGCPプロジェクトを設定する。  
  自分のGCP管理アカウントでgcloudを認証するには以下のコマンドを実施する。
  ```sh
  gcloud auth login
  ```

  デフォルトのgcloudプロジェクトを設定する。
  ```sh
  gcloud config set project <PROJECT_NAME>
  ```

### kubectlをインストールする
  kubectlは[homebrewでインストール可能](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-homebrew-on-macos)なので、以下のコマンドでインストールする。
  ```sh
  brew install kubernetes-cli
  ```

  以下のコマンドで動作をテストする。
  ```sh
  kubectl version
  ```

### halyardをインストールする
  [Spinnaker公式](https://www.spinnaker.io/setup/install/halyard/#install-on-debianubuntu-and-macos)のインストール手順に従いインストールする。
  ```sh
  curl -O https://raw.githubusercontent.com/Spinnaker/halyard/master/install/macos/InstallHalyard.sh

  sudo bash InstallHalyard.sh

  hal -v
  ```

  halyardはMacOSの場合、HighSierraのみ検証されている。  
  Mojaveだとインストール後のバージョン確認でUnknownと表示されるが、Spinnakerのデプロイには成功した。

### bash-completionをインストールする
  必須ではないが、bash-completionをインストールしておくとTabによるコマンド補完が可能になり便利。homebrewでインストール可能。
  ```sh
  brew install bash-completion
  ```

  インストール後、~/.bash_profileに以下を追記する。
  ```bash
  # Bash completion
  if [ -f `brew --prefix`/etc/bash_completion ]; then
    . `brew --prefix`/etc/bash_completion
  fi
  ```

  設定を反映させる。
  ```sh
  source ~/.bash_profile
  ```

  注）halyardのコマンド補完はこのリポジトリにあるhalファイルを/usr/local/etc/bash_completion.d/にコピーして設定を再反映させることで有効にできる。  
  正規手順ではないため、動作は保証しない。

## GKE の準備
### Kubernetesクラスタを作成する
  GKEで使用可能なクラスタバージョンは頻繁に変更されるので以下のコマンドで「validMasterVersions:」に表示されているバージョンを確認しておく。
  ```sh
  gcloud container get-server-config
  ```

  以下のコマンドを実行して、kubernetesクラスタを作成する。  
  クラスタ名は「k8s」、ゾーンは「asia-northeast1-a」とした。  
  他、ノード数を3、クラスタバージョンを1.11.6-gke.3にした。
  ```sh
  gcloud container clusters create k8s \
  --machine-type n1-standard-1 \
  --zone asia-northeast1-a \
  --num-nodes 3 \
  --cluster-version 1.11.6-gke.3
  ```

### APIを有効にする
  Google Cloud Consoleに移動して、以下のAPIを有効にする。
  * [Google Identity and Access Management (IAM) API](https://console.developers.google.com/apis/api/iam.googleapis.com/overview)
  * [Google Cloud Resource Manager API](https://console.developers.google.com/apis/api/cloudresourcemanager.googleapis.com/overview)

### 資格情報を設定する
  参照元のSpinnaker公式にある資格情報ではSpinnakerのデプロイに失敗するので所有者の資格情報を作成して使用した。本来であれば適切な資格情報を作成すべし。

  SpinnakerがGCPサービスを利用するためのサービスアカウントを作成する。
  ```sh
  GCP_PROJECT=$(gcloud info --format='value(config.project)')
  GCP_SA=owner-service-account

  gcloud iam service-accounts create $GCP_SA \
      --project=$GCP_PROJECT \
      --display-name $GCP_SA

  GCP_SA_EMAIL=$(gcloud iam service-accounts list \
      --project=$GCP_PROJECT \
      --filter="displayName:$GCP_SA" \
      --format='value(email)')

  gcloud projects add-iam-policy-binding $GCP_PROJECT \
      --role roles/owner \
      --member serviceAccount:$GCP_SA_EMAIL
  ```

## 資格情報の収集
### GKEクラスタの設定ファイルを取得する
  SpinnakerをデプロイするGKEクラスタの設定ファイル「~/.kube/config」を生成する。
  ```sh
  GKE_CLUSTER_NAME=k8s
  GKE_CLUSTER_ZONE=asia-northeast1-a

  gcloud config set container/use_client_certificate true

  gcloud container clusters get-credentials $GKE_CLUSTER_NAME \
      --zone=$GKE_CLUSTER_ZONE
  ```

### GCPプロジェクト用サービスアカウントのjsonファイルを取得する
  以下のコマンドで、先に作成した所有者権限のサービスアカウントのjsonファイルを取得する。
  ```sh
  GCP_SA=owner-service-account
  GCP_SA_DEST=~/.gcp/gcp.json

  mkdir -p $(dirname $GCP_SA_DEST)

  GCP_SA_EMAIL=$(gcloud iam service-accounts list \
      --filter="displayName:$GCP_SA" \
      --format='value(email)')

  gcloud iam service-accounts keys create $GCP_SA_DEST \
      --iam-account $GCP_SA_EMAIL
  ```

## Spinnakerの設定
### Spinnakerのバージョンを指定
  HalyardがSpinnakerの最新バージョンを使用するように設定する。
  ```sh
  hal config version edit --version $(hal version latest -q)
  ```

### SpinnakerとGoogleCloudStorageの連携設定
  Spinnakerの永続データの配置先をGCSに設定する。
  ```sh
  hal config storage gcs edit \
    --project $(gcloud info --format='value(config.project)') \
    --json-path ~/.gcp/gcp.json

  hal config storage edit --type gcs
  ```

### SpinnakerとGoogleContainerRegistryの連携設定
  GCRからイメージをプルできるように設定する。
  ```sh
  hal config provider docker-registry account add my-gcr-account \
    --address gcr.io \
    --password-file ~/.gcp/gcp.json \
    --username _json_key

  hal config provider docker-registry enable
  ```

### GoogleCloudStorageアーティファクト認証情報の設定
  SpinnakerのパイプラインステージでGCSオブジェクトをアーティファクトとして使用しデータを読み取るための設定をする。
  GCPのロールroles/storage.adminが付与されたiamサービスアカウントを作成し、資格情報jsonファイルを取得する。
  ```sh
  SERVICE_ACCOUNT_NAME=spin-gcs-artifacts-account
  SERVICE_ACCOUNT_DEST=~/.gcp/gcs-artifacts-account.json

  gcloud iam service-accounts create \
      $SERVICE_ACCOUNT_NAME \
      --display-name $SERVICE_ACCOUNT_NAME

  SA_EMAIL=$(gcloud iam service-accounts list \
      --filter="displayName:$SERVICE_ACCOUNT_NAME" \
      --format='value(email)')

  PROJECT=$(gcloud info --format='value(config.project)')

  gcloud projects add-iam-policy-binding $PROJECT \
      --role roles/storage.admin --member serviceAccount:$SA_EMAIL

  mkdir -p $(dirname $SERVICE_ACCOUNT_DEST)

  gcloud iam service-accounts keys create $SERVICE_ACCOUNT_DEST \
      --iam-account $SA_EMAIL
  ```

  アーティファクトサポートを有効にする。
  ```sh
  hal config features edit --artifacts true
  ```

  アーティファクトアカウントを追加する。
  ```sh
  ARTIFACT_ACCOUNT_NAME=my-gcs-artifact-account

  hal config artifact gcs account add $ARTIFACT_ACCOUNT_NAME \
    --json-path $SERVICE_ACCOUNT_DEST
  ```

  GCSアーティファクトサポートを有効にする。
  ```sh
  hal config artifact gcs enable
  ```

### Kubernetesの連携設定（マニフェスト ベース）
#### Kubernetesサービスアカウントを作成する
  Spinnakerを[Kubernetesサービスアカウント](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)に関連付ける。
  ```sh
  CONTEXT=$(kubectl config current-context)

  # 下記spinnaker.ioから取得したマニフェストではClusterAdminロールを使用したサービスアカウントが作成されますが
  # ClusterAdmin権限が不要であれば、より限定的な権限を適用することも可能です。
  kubectl apply --context $CONTEXT \
      -f https://spinnaker.io/downloads/kubernetes/service-account.yml

  TOKEN=$(kubectl get secret --context $CONTEXT \
     $(kubectl get serviceaccount spinnaker-service-account \
         --context $CONTEXT \
         -n spinnaker \
         -o jsonpath='{.secrets[0].name}') \
     -n spinnaker \
     -o jsonpath='{.data.token}' | base64 --decode)

  kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

  kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
  ```

#### Kubernetes Provider V2 (Manifest base)の機能を有効にする。
  kubernetes provider を有効にする。
  ```sh
  hal config provider kubernetes enable
  ```
  アカウントを追加する。
  ```sh
  CONTEXT=$(kubectl config current-context)

  hal config provider kubernetes account add my-k8s-v2-account \
      --provider-version v2 \
      --context $CONTEXT
  ```

## Spinnakerのデプロイ
### デプロイの実行
  Kubernetes Providerの設定で作成したアカウントを指定してSpinnakerのデプロイを実施する。Spinnakerの各コンポーネントを分散配置するオプションを指定。
  ```sh
  hal config deploy edit \
    --account-name my-k8s-v2-account \
    --type distributed

  hal deploy apply
  ```

### Spinnakerにアクセス
  Spinnaker UIにアクセスするためにport-forwardを実行する。
  ```sh
  hal deploy connect
  ```

  ブラウザで「[localhost:9000](http://localhost:9000)」にアクセスする。
