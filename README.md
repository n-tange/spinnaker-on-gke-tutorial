# MacOSX mojaveにhalyardをインストールして、GKE上にSpinnakerをデプロイする
2018/12/28

## 参考元
* Spinnaker公式：[Try out Halyard on GKE](https://www.spinnaker.io/setup/quickstart/halyard-gke/)

## Mac の準備
### gcloudをインストールする
  [Cloud SDKのインストール](https://cloud.google.com/sdk/downloads#interactive)にある手順に従い、gcloudをインストールする。

  gcloudを認証してデフォルトのGCPプロジェクトを設定する。  
  自分のGCP管理アカウントでgcloudを認証するには以下のコマンドを実施する。
  ```console
  gcloud auth login
  ```

  デフォルトのgcloudプロジェクトを設定する。
  ```console
  gcloud config set project <PROJECT_NAME>
  ```

### kubectlをインストールする
  kubectlは[homebrewでインストール可能](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-homebrew-on-macos)なので、以下のコマンドでインストールする。
  ```console
  brew install kubernetes-cli
  ```

  以下のコマンドで動作をテストする。
  ```console
  kubectl version
  ```

### halyardをインストールする
  [Spinnaker公式](https://www.spinnaker.io/setup/install/halyard/#install-on-debianubuntu-and-macos)のインストール手順に従いインストールする。
  ```console
  curl -O https://raw.githubusercontent.com/Spinnaker/halyard/master/install/macos/InstallHalyard.sh

  sudo bash InstallHalyard.sh

  hal -v
  ```

  halyardはMacOSの場合、HighSierraのみ検証されている。  
  Mojaveだとインストール後のバージョン確認でUnknownと表示されるが、Spinnakerのデプロイには成功した。

### bash-completionをインストールする
  必須ではないが、bash-completionをインストールしておくとTabによるコマンド補完が可能になり便利。homebrewでインストール可能。
  ```console
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
  ```console
  source ~/.bash_profile
  ```

  注）halyardのコマンド補完はこのリポジトリにあるhalファイルを/usr/local/etc/bash_completion.d/にコピーして設定を再反映させることで有効にできる。  
  正規手順ではないため、動作は保証しない。

## GKE の準備
### Kubernetesクラスタを作成する
  GKEで使用可能なクラスタバージョンは頻繁に変更されるので以下のコマンドで「validMasterVersions:」に表示されているバージョンを確認しておく。
  ```console
  gcloud container get-server-config
  ```

  以下のコマンドを実行して、kubernetesクラスタを作成する。  
  クラスタ名は「k8s」、ゾーンは「asia-northeast1-a」とした。  
  他、ノード数を3、クラスタバージョンを1.11.3-gke.24にした。
  ```console
  gcloud container clusters create k8s \
  --machine-type n1-standard-1 \
  --zone asia-northeast1-a \
  --num-nodes 3 \
  --cluster-version 1.11.3-gke.24
  ```

  クラスタの作成が完了したら、[Google Cloud ConsoleのGKEセクション](https://console.cloud.google.com/kubernetes/list)から作成したクラスタを編集し
  「以前の承認」を有効にする。


### APIを有効にする
  Google Cloud Consoleに移動して、以下のAPIを有効にする。
  * [Google Identity and Access Management (IAM) API](https://console.developers.google.com/apis/api/iam.googleapis.com/overview)
  * [Google Cloud Resource Manager API](https://console.developers.google.com/apis/api/cloudresourcemanager.googleapis.com/overview)

### 資格情報を設定する
  参照元のSpinnaker公式にある資格情報ではSpinnakerのデプロイに失敗するので所有者の資格情報を作成して使用した。本来であれば適切な資格情報を作成すべし。

  SpinnakerがGCPサービスを利用するためのサービスアカウントを作成する。
  ```console
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
  ```console
  GKE_CLUSTER_NAME=k8s
  GKE_CLUSTER_ZONE=asia-northeast1-a

  gcloud config set container/use_client_certificate true

  gcloud container clusters get-credentials $GKE_CLUSTER_NAME \
      --zone=$GKE_CLUSTER_ZONE
  ```

### GCPプロジェクト用サービスアカウントのjsonファイルを取得す
  以下のコマンドで、先に作成した所有者権限のサービスアカウントのjsonファイルを取得する。
  ```console
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
  ```console
  hal config version edit --version $(hal version latest -q)
  ```

### SpinnakerとGoogleCloudStorageの連携設定
  Spinnakerの永続データの配置先をGCSに設定する。
  ```console
  hal config storage gcs edit \
    --project $(gcloud info --format='value(config.project)') \
    --json-path ~/.gcp/gcp.json

  hal config storage edit --type gcs
  ```

### SpinnakerとGoogleContainerRegistryの連携設定
  GCRからイメージをプルできるように設定する。
  ```console
  hal config provider docker-registry account add my-gcr-account \
    --address gcr.io \
    --password-file ~/.gcp/gcp.json \
    --username _json_key

  hal config provider docker-registry enable
  ```

### Kubernetesの連携設定
  Kubernates Providerの機能を有効にする。
  ```console
  hal config provider kubernetes account add my-k8s-account \
      --docker-registries my-gcr-account \
      --context $(kubectl config current-context)

  hal config provider kubernetes enable
  ```

## Spinnakerのデプロイ
### デプロイの実行
  Kubernetes Providerの設定で作成したアカウントを指定してSpinnakerのデプロイを実施する。Spinnakerの各コンポーネントを分散配置するオプションを指定。
  ```console
  hal config deploy edit \
    --account-name my-k8s-account \
    --type distributed

  hal deploy apply
  ```

### Spinnakerにアクセス
  Spinnaker UIにアクセスするためにport-forwardを実行する。
  ```console
  hal deploy connect
  ```

  ブラウザで「http://localhost:9000」にアクセスする。
