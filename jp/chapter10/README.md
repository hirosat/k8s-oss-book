# 10章 - CI/CDパイプライン

## 訂正・フィードバック

現在ありません

## リンク一覧

### K8sのノードリソースについて

- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [(GCP)クラスタのサイズを増やす](https://cloud.google.com/kubernetes-engine/docs/how-to/resizing-a-cluster?hl=ja)

### コンテナレジストリについて

- [CNCF Landscape Guide](https://landscape.cncf.io/guide)
  - [Container Registry](https://landscape.cncf.io/guide#provisioning--container-registry)
- [Harbor](https://goharbor.io/)
  - [Deploying Harbor with High Availability via Helm](https://goharbor.io/docs/2.9.0/install-config/harbor-ha-helm/) : Helmを使ってK8s上にインストールする方法
  - [Helmチャート(ArtifactHub)](https://artifacthub.io/packages/helm/harbor/harbor)
  - [Scanner Adaptorの検証状況](https://goharbor.io/docs/2.9.0/install-config/harbor-compatibility-list/#scanner-adapters)
- [NFS Ganesha server and external provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner)
  - [Helmチャート(ArtifactHub)](https://artifacthub.io/packages/helm/nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner)
- [Cosign](https://github.com/sigstore/cosign)

### CIツールについて

- [kpack(GitHub)](https://github.com/buildpacks-community/kpack)
  - [kpackのReleaseページ](https://github.com/buildpacks-community/kpack/releases)
  - [kp CLIのReleaseページ](https://github.com/buildpacks-community/kpack-cli/releases)
  - [Tutorial](https://github.com/buildpacks-community/kpack/blob/main/docs/tutorial_kp.md)

### CDツールについて

- [CNCF Landscape Guide](https://landscape.cncf.io/guide)
  - [Continuout Integration & Delivery](https://landscape.cncf.io/guide#app-definition-and-development--continuous-integration-delivery)
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/)
  - [Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
  - [Argo CD CLIのReleaseページ(GitHub)](https://github.com/argoproj/argo-cd/releases/latest)
- [argoproj-labs(GitHub)](https://github.com/argoproj-labs)
  - [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/en/stable/)
    - [Installation](https://argocd-image-updater.readthedocs.io/en/stable/install/installation/)
    - [Registries](https://argocd-image-updater.readthedocs.io/en/stable/configuration/registries/)
    - [Images](https://argocd-image-updater.readthedocs.io/en/stable/configuration/images/)

## コード集

### K8sのノードリソースについて

ノードリソースの確認
```
kubectl describe node 
```

#### Workerノードの追加

(※. 以下、kubeadm環境向けの話となります)

**Control Plane** で、ノード追加コマンドを表示
```
kubeadm token create --print-join-command
```

**新たなVM** で、ノード追加 (※. ↑で発行されたコマンドを実行)
```
sudo kubeadm join (Control Plane IP) --token (Token値) --discovery-token-ca-cert-hash (hash値)
```

**PCクライアント** で、ノードが追加されたことを確認
```
kubectl get node
```

---
### コンテナレジストリについて

#### Harborについて

##### 共有ストレージの用意

NFS Server ProvisionerのHelmリポジトリを登録
```
helm repo add nfs-ganesha-server-and-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/
```

Helmのインストール
```
helm install nfs-server-provisioner \
  nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
  --set persistence.enabled=true --set persistence.size=30Gi \
  -n nfs-server --create-namespace --wait
```

デプロイされたオブジェクトを確認
```
kubectl get pod -n nfs-server

kubectl get sc
```

テスト用のPVCを作成
```
cat <<EOF | kubectl apply -f -
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
EOF
```

PVCの動作確認と削除
```
kubectl get pvc

kubectl delete pvc test-pvc
```


##### harbor namespaceの作成

harbor namespaceを作成
```
kubectl create ns harbor
```

##### Harbor用ドメインと証明書の用意

```
DOMAIN="harbor.hirosat.com"   # あなたのドメインを入力

cat << EOF > harbor-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: harbor-cert
spec:
    secretName: harbor-tls-cert
    commonName: "$DOMAIN"
    dnsNames:
    - "$DOMAIN"
    issuerRef:
      name: acme-issuer
      kind: ClusterIssuer
EOF

kubectl apply -f harbor-cert.yaml -n harbor
```

証明書Secretが作成されたことを確認
```
kubectl get certificate -n harbor

kubectl get secret -n harbor
```

---
##### Harbor設定の編集

HarborのHelmリポジトリを登録
```
helm repo add harbor https://helm.goharbor.io

helm show values harbor/harbor > harbor-values-templates.yaml
```

Harbor設定ファイルの作成
```
DOMAIN="harbor.hirosat.com"   # あなたのドメインを入力
PASS="Harbor12345"   # 任意のパスワードを入力
cat << EOF > harbor-values.yaml
expose:
  tls:
    certSource: secret
    secret:
      secretName: "harbor-tls-cert"
  ingress:
    hosts:
      core: $DOMAIN
externalURL: https://$DOMAIN
persistence:
  persistentVolumeClaim:
    registry:
      size: 20Gi
harborAdminPassword: "$PASS"
EOF
```

---
##### Harborのインストール

helmでHarborをインストール
```
helm install -f harbor-values.yaml harbor harbor/harbor -n harbor
```

作成されたオブジェクトの確認
```
kubectl get pod -n harbor

kubectl get ingress -n harbor
```

---
##### Harborの基本操作

Nginxイメージを取得、確認
```
docker pull nginx

docker images | grep nginx
```

Harborにログイン
```
DOMAIN=harbor.hirosat.com   # 自分のドメインを入力。

docker login $DOMAIN
```

Harbor用のtagを用意し、Harborにpush
```
docker tag nginx:latest $DOMAIN/test/nginx:dev1

docker push $DOMAIN/test/nginx:dev1
```

---
##### イメージのScan

alpineイメージをpush
```
docker pull nginx:alpine

docker tag nginx:alpine $DOMAIN/test/nginx:alpine

docker push $DOMAIN/test/nginx:alpine
```

---
### CIツールについて

#### kpackとkp CLIのインストール

kpackのインストール
```
VER=0.11.5
curl -LO https://github.com/buildpacks-community/kpack/releases/download/v$VER/release-$VER.yaml

kubectl apply -f release-$VER.yaml

kubectl get pod -n kpack
```

kpのインストール
```
VER=0.12.1

ARCH=darwin-arm64    # Windowsは、windows-amd64

curl -LO https://github.com/buildpacks-community/kpack-cli/releases/download/v$VER/kp-$ARCH-$VER

sudo install kp-$ARCH-$VER /usr/local/bin/kp    # Windowsは、~/bin/kp

kp version
```

---
#### kpackのセットアップ

##### 1.	kpackの初期設定

Harbor接続用の**認証情報Secret**を作成。
```
DOMAIN=harbor.hirosat.com   # ドメインを入力。

docker login $DOMAIN -u admin

kp secret create harbor-cred --registry $DOMAIN --registry-user admin -n kpack

kp secret list -n kpack
```

kpackで使用する**リポジトリ**を設定
```
kp config default-repository $DOMAIN/kpack/images

kp config default-repository
```

kpackが使用する**SA**を設定
```
kp config default-service-account default

kp config default-service-account

kubectl describe sa -n kpack default
```

---
##### 2.	buildpackage (ビルド用パッケージ) の用意

**ClusterStore**を作成
```
kp clusterstore save default -b paketobuildpacks/builder-jammy-base

kp clusterstore list
```

**buildpack** 一覧を表示
```
kubectl get clusterstore default -o jsonpath='{range .status.buildpacks[*]}{.id}{"\n"}'
```

---
##### 3.	stack (baseイメージ)の用意

**ClusterStack** の作成
```
kp clusterstack save base --build-image paketobuildpacks/build-jammy-base \
 --run-image paketobuildpacks/run-jammy-base

kp clusterstack status base
```

---
##### 4.	ビルドを行うnamespaceの設定

ビルドしたいnamespaceに、認証情報Secretを設置。
```
kp secret create harbor-cred --registry $DOMAIN --registry-user admin -n default
```

---
##### 5.	builderの作成

orderマニフェストの用意
```
cat << EOF > order.yaml
- group:
  - id: paketo-buildpacks/dotnet-core
- group:
  - id: paketo-buildpacks/go
- group:
  - id: paketo-buildpacks/python
- group:
  - id: paketo-buildpacks/web-servers
- group:
  - id: paketo-buildpacks/java-native-image
- group:
  - id: paketo-buildpacks/java
- group:
  - id: paketo-buildpacks/nodejs
- group:
  - id: paketo-buildpacks/procfile
EOF
```

ClusterBuilderの作成
```
kp clusterbuilder save kp-builder --stack base --store default \
  --tag $DOMAIN/kpack/builder -o order.yaml
```

作成されたClusterBuilderの状態を確認
```
kp clusterbuilder status kp-builder
```

---
##### 6.	Imageリソースの作成 (kpackでビルド)

Imageの作成(ビルドの実行)
```
GIT=https://github.com/hirosat/k8s-oss-sample-app  #あなたのリポジトリを入力

kp image save my-image -c kp-builder --tag $DOMAIN/kpack/app --git $GIT
```

ビルド中の状態を確認
```
kp image list

kp image status my-image
```

ビルドログを確認
```
kp build logs my-image
```

ビルドリストを確認
```
kp build list
```

ビルド後の状態を確認
```
kp image status my-image
```

---
##### 7.	ビルドしたコンテナをデプロイ

ビルドされたイメージをK8sにデプロイ
```
kubectl run kp-sample --image=harbor.hirosat.com/kpack/app:latest

kubectl get pod kp-sample

kubectl expose pod kp-sample --port=3000 --name=kp-sample-svc --type=LoadBalancer

kubectl get svc kp-sample-svc
```

---
##### 8.	kpackの再ビルド

ソースコード内容を変更して、gitにpush
```
cd k8s-oss-sample-app    # サンプルアプリのgitプロジェクトに移動

vi src/App.css
(.App-headerクラスのbackground-colorを、お好きな色に変更してみましょう。)

git diff

git add .

git commit -m “bg changed to green”

git push
```

新たなBuildが生成されたことを確認
```
kp build list
```

再デプロイ
```
kubectl delete pod kp-sample

kubectl run kp-sample --image=harbor.hirosat.com/kpack/app:latest
```

デプロイしたpodとserviceを削除
```
kubectl delete svc kp-sample-svc

kubectl delete pod kp-sample
```

---
### CDツールについて

#### Argo CDについて

##### 1.	Argo CDのインストール

argocd namespaceを作成
```
kubectl create namespace argocd
```

ArgoCDのインストール
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pod -n argocd
```

---
##### 2.	Argo CD CLIのインストール

ArgoCD CLIのインストール
```
VER=v2.9.3          # 2024年1月9日時点の検証バージョン。最新版をお使い下さい

ARCH=darwin-arm64   # 左記はMac用。Windows用は、windows-amd64.exeとする

curl -LO https://github.com/argoproj/argo-cd/releases/download/$VER/argocd-$ARCH

sudo install argocd-$ARCH /usr/local/bin/argocd   # Windowsは、~/bin/argocd

argocd version
```

---
##### 3. Argo CD API Server へのアクセス

Service type: LoadBalancerに変更
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc -n argocd argocd-server
```

---
##### 4.	CLIを使ったログイン

adminユーザ用の初期パスワードを生成
```
argocd admin initial-password -n argocd
```

argocdにログイン
```
argocd login (あなたのIPアドレス)
```

adminパスワードを更新
```
argocd account update-password
```

---
##### 5.	K8sマニフェスト用Gitリポジトリの準備

Gitリポジトリをローカル環境にCloneして移動
```
cd ~/git

GITREPO=git@github.com:hirosat/argocd-manifests.git  # あなたのGitリポジトリ

git clone $GITREPO

cd argocd-manifests
```

ディレクトリ作成と、マニフェストの設置
```
mkdir k8s-oss-sample-app

cd k8s-oss-sample-app

DOMAIN=harbor.hirosat.com  # あなたのHarbor用ドメイン

kubectl create deployment kp-sample --image=$DOMAIN/kpack/app:latest --replicas=2 --dry-run=client -o yaml > deployment.yaml

kubectl create service loadbalancer kp-sample --tcp=80:3000 --dry-run=client -o yaml > service.yaml

ls
```

Gitリポジトリにpush
```
git add .

git commit -m 'Added manifests'

git push
```

※. [参考: この変更によるGitリポジトリの状態](https://github.com/hirosat/argocd-manifests/commit/a8e1855ae3a61d5844cfada348a90d50fa383022)

---
##### 7.	Appのデプロイ

デプロイされたAPPを確認
```
kubectl get deploy kp-sample

kubectl get svc kp-sample
```

---
##### 8.	Appの更新

pod数を **3replicas** に更新してpush
```
cd ~/git/argo-manifests/k8s-oss-sample-app

vi deployment.yaml
(replicasの行を探し、3に変更)
  replicas: 3

git add .

git commit -m 'Changed to 3 replicas'

git push
```

[※. 参考: replicasを3に変更](https://github.com/hirosat/argocd-manifests/commit/75fc99ee74dd1d3cc4a76e5a32724e6334798122)

replicasが変更されたことを確認
```
kubectl get deploy kp-sample
```

---
#### CIとCDの統合

##### 1.	マニフェスト用リポジトリの更新

kustomization.yaml を設置してpush
```
cd ~/git/argocd-manifests

cat << EOF > k8s-oss-sample-app/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
EOF

git add .

git commit -m 'Use kustomize'

git push
```

[※. 参考: kustomization.yaml](https://github.com/hirosat/argocd-manifests/blob/main/k8s-oss-sample-app/kustomization.yaml)

---
##### 2.	Argo CD Image Updaterのインストール

Image Updaterのマニフェストを適用
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

---
##### 3.	Argo CD Image Updaterの動作確認

Image Updaterのバージョン表示
```
POD=`kubectl get pod -n argocd -o name | grep argocd-image`

kubectl exec -it -n argocd $POD -- argocd-image-updater version
```

コンテナリポジトリへの接続テスト
```
REPO=harbor.hirosat.com/kpack/app   # あなたのコンテナリポジトリ

kubectl exec -it -n argocd $POD -- argocd-image-updater test $REPO --loglevel info
```

---
##### 4.	Argo CD Applicationへの設定

ArgoCD Applicationに、Image Updaterの **アノテーション** を付与
```
kubectl edit applications -n argocd kp-sample
(途中、省略。以下、編集内容)
metadata:   # metadata: 行の下に、以下の3行のエントリを追加
  annotations:
    argocd-image-updater.argoproj.io/image-list: app=harbor.hirosat.com/kpack/app:latest
    argocd-image-updater.argoproj.io/app.update-strategy: digest
```

---
##### 5.	動作確認

**別端末** で、Image Updaterのログを追跡するよう仕込む。
```
kubectl -n argocd logs --selector app.kubernetes.io/name=argocd-image-updater -f
```

ソースコードを変更してpush
```
vi src/App.css
(※.App-header Class内の背景を青に変更)
.App-header {
  background-color: blue;

git add .

git commit -m 'set blue background'

git push
```

kpackのビルドの様子を確認
```
kp build logs my-image
```

Imageのdigest値が更新されたことを確認
```
kubectl get deploy kp-sample -o yaml | grep image:
```
