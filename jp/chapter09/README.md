# 9章 - アプリケーションの管理とデプロイ

## 訂正・フィードバック

現在ありません

## リンク一覧

### K8sのNamespace

- [K8s API Overview](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.27/) ※. リンク先URL内の、「v.1.27」の部分は最新バージョンに置き換えて下さい

### K8sのユーザ管理

- [K8s公式ドキュメント](https://kubernetes.io/docs/home/)
  - [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
  - [Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [CNCF Landscape Guide](https://landscape.cncf.io/guide)
  - [Key Management](https://landscape.cncf.io/guide#provisioning--key-management)
- [kubelogin](https://github.com/int128/kubelogin)
- [Pinniped](https://pinniped.dev/)
  - [GitHub releaseページ](https://github.com/vmware-tanzu/pinniped/releases/)
  - [Concierge with Webhook](https://pinniped.dev/docs/tutorials/concierge-only-demo/) (Conciegeのチュートリアル)

### K8sのApp管理

- [CNCF Application Definition & Image Build](https://landscape.cncf.io/guide#app-definition-and-development--application-definition-image-build)
- [Helm](https://helm.sh/)
  - [Helmのインストール](https://helm.sh/ja/docs/intro/install/)
  - [HelmのGitHub release](https://github.com/helm/helm/releases)
- [Artifacthub](https://artifacthub.io/)
  - [検索ページ](https://artifacthub.io/packages/search)

### K8sのHTTPS接続

- [Let's Encrypt](https://letsencrypt.org/)
- [cert-manager](https://cert-manager.io/)
  - [インストールページ](https://cert-manager.io/docs/installation/)
  - [DNS01](https://cert-manager.io/docs/configuration/acme/dns01/)
  - [Cloudflare](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/)

## コード集

### K8sのNamespace

Namespaceの表示
```
kubectl get namespaces
```

Namespaceの作成
```
kubectl create namespace test

kubectl get namespace test
```

---

### K8sのユーザ管理

#### K8sの認証

K8sクラスタのクライアント認証データを取得
```
kubectl config view --minify --raw -o jsonpath='{.users[].user.client-certificate-data}'
```

K8sクラスタのクライアント認証データをデコードして、証明書内容を確認
```
kubectl config view --minify --raw -o jsonpath='{.users[].user.client-certificate-data}' |\
base64 -d | openssl x509 -text
```

---

#### Pinnipedについて

##### Pinnipedのインストール

Mac環境のPinniped CLIインストール方法 (※.VER変数は最新のものに置き換える)
```
VER=v0.28.0

TARGET=pinniped-cli-darwin-arm64

curl -LO https://github.com/vmware-tanzu/pinniped/releases/download/$VER/$TARGET

chmod +x $TARGET

sudo mv $TARGET /usr/local/bin/pinniped
```

Windows環境のPinniped CLIインストール方法 (※.VER変数は最新のものに置き換える)
```
VER=v0.28.0

TARGET=pinniped-cli-windows-amd64.exe

curl -LO https://github.com/vmware-tanzu/pinniped/releases/download/$VER/$TARGET

mv $TARGET ~/bin/pinniped.exe
```

Pinniped CLIの動作確認
```
pinniped version
```

Pinniped Conciergeのインストール
```
kubectl apply -f https://get.pinniped.dev/$VER/install-pinniped-concierge-crds.yaml

kubectl apply -f https://get.pinniped.dev/$VER/install-pinniped-concierge-resources.yaml
```

---

##### Pinniped Conciergeの利用

デモ用の認証アプリ(local-user-authenticator)のインストール
```
kubectl apply -f https://get.pinniped.dev/v0.28.0/install-local-user-authenticator.yaml
```

認証アプリにユーザ登録
```
kubectl create secret generic demo \
  --namespace local-user-authenticator \
  --from-literal=groups=group1,group2 \
  --from-literal=passwordHash=$(htpasswd -nbBC 10 x password123 | sed -e "s/^x://")
```

認証アプリのCA bundleを取得し、/tmp以下に一時保存
```
kubectl get secret local-user-authenticator-tls-serving-certificate \
  --namespace local-user-authenticator \
  -o jsonpath={.data.caCertificate} \
  | tee ./local-user-authenticator-ca-base64-encoded
```

Conciergeの接続先設定
```
cat <<EOF | kubectl create -f -
apiVersion: authentication.concierge.pinniped.dev/v1alpha1
kind: WebhookAuthenticator
metadata:
  name: local-user-authenticator
spec:
  endpoint: https://local-user-authenticator.local-user-authenticator.svc/authenticate
  tls:
    certificateAuthorityData: $(cat ./local-user-authenticator-ca-base64-encoded)
EOF
```

pinniped CLIで認証して、demoユーザ用のkubeconfigを生成
```
pinniped get kubeconfig \
  --static-token "demo:password123" \
  --concierge-authenticator-type webhook \
  --concierge-authenticator-name local-user-authenticator \
  > ./demo-kubeconfig
```

生成したkubeconfigの情報を表示
```
pinniped whoami --kubeconfig ./demo-kubeconfig
```

生成したkubeconfigを使用して、pod情報を取得
```
kubectl --kubeconfig ./demo-kubeconfig get pods -n pinniped-concierge
```

管理者ユーザで、demoユーザにクラスタの読み取り権限を付与
```
kubectl create clusterrolebinding demo-view --clusterrole view --user demo
```

再び、生成したkubeconfigを使用して、pod情報を取得
```
kubectl --kubeconfig ./demo-kubeconfig get pods -n pinniped-concierge
```

---

#### Service Account
Service Accountの表示と作成
```
kubectl get sa

kubectl create sa test

kubectl get sa
```

---

#### K8sの認可

##### クラスタ管理者の権限を確認

**cluster-admin** の **ClusterRole** を表示
```
kubectl describe clusterrole cluster-admin
```

**ClusterRoleBinding** を一覧表示し、 **cluster-admin** の名前で **grep**
```
kubectl get clusterrolebinding | grep cluster-admin
```

検索に引っかかった **ClusterRoleBinding** (cluster-admin)の詳細を表示
```
kubectl describe clusterrolebinding cluster-admin
```

逆引き(**system:masters** Groupから探す)
```
kubectl describe clusterrolebinding | grep -B10 system:masters
```

---

##### Roleの作成

Podの読み取り権限を持つRoleを作成
```
kubectl create role pod-only --resource=pods --verb="*"

kubectl describe role pod-only
```

---

##### RoleBindingの作成

以前作成したClusterRoleBindingを削除し、demoユーザの権限がない事を確認
```
kubectl delete clusterrolebinding demo-view

kubectl --kubeconfig /tmp/demo-kubeconfig get pod
```

作成したRoleとdemoユーザを紐付ける
```
kubectl create rolebinding demo-pod-only --role=pod-only --user=demo

kubectl describe rolebinding demo-pod-only
```

demoユーザで、いくつかの操作を行う
```
kubectl --kubeconfig /tmp/demo-kubeconfig run nginx --image=nginx

kubectl --kubeconfig /tmp/demo-kubeconfig get pod

kubectl --kubeconfig /tmp/demo-kubeconfig delete pod nginx

kubectl --kubeconfig /tmp/demo-kubeconfig get svc
```

---

### K8sのApp管理

#### Helmについて

##### Helm CLI のインストール
```
VER=v3.13.3            # 左記は本書で検証したバージョン。最新版をご利用下さい

ARCH=darwin-arm64      # 左記はMacの例。Windowsの場合は、windows-amd64

curl -LO https://get.helm.sh/helm-$VER-$ARCH.tar.gz

tar zxvf helm-$VER-$ARCH.tar.gz

mv $ARCH/helm /usr/local/bin/  # Windowsの場合、~/bin/ 等PATHの場所にする
```

---

##### Helm chart の見つけ方

Chart一覧の表示
```
helm search hub

helm search hub wordpress

helm search hub wordpress | wc -l
```

bitnamiリポジトリの登録
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```

リポジトリ一覧を表示
```
helm repo list
```

---

##### Chart名を確認
Chart一覧を表示・検索
```
helm search repo

helm search repo wordpress
```

---

##### Chartのインストール

default StorageClassの設定 (※. 既にdefaultが存在する場合は、そちらを使っても良い。)
```
kubectl get storageclass

kubectl patch storageclass csi-hostpath-sc -p \
'{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

wordpress chartをインストール
```
helm install my-wordpress bitnami/wordpress
```

インストールされたK8sオブジェクトを確認
```
kubectl get pod

kubectl get service

kubectl get pvc
```

Helm Releaseの一覧を確認
```
helm list
```

Helm Release情報の格納先を確認
```
kubectl get secret | grep helm
```

---

##### Chartのカスタマイズ

現在のReleaseに対する設定内容を確認
```
helm get values my-wordpress
```

Chart設定のひな形を取得
```
helm show values bitnami/wordpress > wordpress-values.yaml
```

kindクラスタ用に、type: LoadBalancerをNodePortに変更 (※. kindクラスタ以外の方は、**使用ディスク量の変更**に挑戦してみましょう)
```
vi wordpress-values.yaml
(534行目辺りまで移動し、以下の様にtype: LoadBalancerをtype: NodePortに変更)
## WordPress service parameters
##
service:
  ## @param service.type WordPress service type
  ##
  type: NodePort
```

(続き) kindクラスタ用に、nodePorts設定をkindクラスタ設定に合わせる。
```
  ## Node ports to expose
  ## @param service.nodePorts.http Node port for HTTP
  ## @param service.nodePorts.https Node port for HTTPS
  ## NOTE: choose port between <30000-32767>
  ##
  nodePorts:
    http: "30080"
    https: "30443"
```

設定ファイルをReleaseに適用
```
helm upgrade my-wordpress bitnami/wordpress -f wordpress-values.yaml
```

my-wordpress serviceの状態を確認
```
kubectl get service my-wordpress
```

再度、Release一覧を確認
```
helm list
```

今回Helmに適用した設定内容を確認
```
helm get values my-wordpress
```

変更点のみを記載したマニフェストを用意
```
cat << EOF > wordpress-simple.yaml
service:
  type: NodePort
  nodePorts:
    http: "30080"
    https: "30443"
EOF
```

上記のマニフェストで更新
```
helm upgrade my-wordpress bitnami/wordpress -f wordpress-simple.yaml
```

再び、適用した設定内容を確認

```
kubectl get service my-wordpress

helm get values my-wordpress
```

---

##### Wordpressの動作確認

Chartの状態を再確認
```
helm status my-wordpress
```

WordpressのURLを取得 (LoadBalancerの場合)
```
export SERVICE_IP=$(kubectl get svc --namespace default my-wordpress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")

echo "WordPress URL: http://$SERVICE_IP/"

echo "WordPress Admin URL: http://$SERVICE_IP/admin"
```

WordpressのURLを取得 (NodePortの場合)
```
$ export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-wordpress)

$ export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")

$ echo "WordPress URL: http://$NODE_IP:$NODE_PORT/"

$ echo "WordPress Admin URL: http://$NODE_IP:$NODE_PORT/admin"
```

Wordpressのユーザとパスワードを取得
```
echo Username: user

echo Password: $(kubectl get secret --namespace default my-wordpress -o jsonpath="{.data.wordpress-password}" | base64 -d)
```

---

##### ReleaseのRollback

過去のリビジョンに戻す
```
helm list

helm rollback my-wordpress 1

kubectl get service my-wordpress

helm list
```

Helm設定のリリース履歴を確認
```
helm history my-wordpress
```

---

##### Releaseの削除

my-wordpressのReleaseを削除
```
helm delete my-wordpress
```

PVCの確認
```
kubectl get pvc
```

PVCを削除
```
kubectl delete pvc data-my-wordpress-mariadb-0

kubectl get pvc
```
---

### K8sにおけるHTTPS接続

#### cert-managerについて

##### cert-managerのインストール
インストールの実施と、作成されたPodを確認
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

kubectl get pod -n cert-manager
```

---

##### API Tokenの発行

API Tokenを格納したSecretを作成
```
TOKEN=(API Tokenを入力)

kubectl create secret generic cf-token --from-literal=api-token=$TOKEN -n cert-manager
```

---

##### ClusterIssuerの作成
マニフェストの用意
```
cat << EOF > clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: acme-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: acme-ca-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cf-token
            key: api-token
EOF
```

マニフェストを適用して確認
```
kubectl apply -f clusterissuer.yaml

kubectl get clusterissuer
```

作成されたCAキーのSecretを確認
```
kubectl get secret -n cert-manager acme-ca-key -o jsonpath='{.data.tls\.key}' | base64 -d
```

---

##### Certificate(証明書)の作成

Certificateマニフェストの用意 (cat以降は、複数行をまとめて実行)
```
DOMAIN="あなたのドメインを入力"

cat << EOF > certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-cert
spec:
    secretName: acme-tls-cert
    commonName: "$DOMAIN"
    dnsNames:
    - "$DOMAIN"
    issuerRef:
      name: acme-issuer
      kind: ClusterIssuer
EOF
```

マニフェストを適用し、状態を確認
```
kubectl apply -f certificate.yaml

kubectl get certificate

kubectl describe certificate acme-cert
```

(数分後)TLS証明書が生成されたことを確認
```
kubectl get certificate

kubectl get certificaterequests

kubectl get order
```

TLSのSecret内容を確認
```
kubectl get secret acme-tls-cert

kubectl get secret acme-tls-cert -o jsonpath='{.data.tls\.crt}' | base64 -d
```

このSecretのデータを、opensslコマンドで、X.509証明書形式に変換 (※. 2行まとめて実行)
```
kubectl get secret acme-tls-cert -o jsonpath='{.data.tls\.crt}' |\
base64 -d | openssl x509 -text -noout
```

---

##### Ingress(HttpProxy)に証明書を適用

7章のHttpProxyを用いたApp構成を再現
```
kubectl run nginx --image=nginx

kubectl expose pod nginx --port=80 --target-port=80 --name=nginx-svc

kubectl apply -f httpproxy-app1.yaml

kubectl get httpproxy

# ※.kindクラスタ以外の場合、以下の「:30080」は付ける必要がありません。
curl -s app1.test.hirosat.com:30080 | grep '<title>'
```

HttpProxyリソースを編集して、HTTPS化
```
cp httpproxy-app1.yaml httpproxy-app1-https.yaml

vi httpproxy-app1-https.yaml
(※.途中省略)
  virtualhost:
    fqdn: (あなたのドメイン)
(※.以下の2行を追記)
    tls:
      secretName: acme-tls-cert
```

HttpProxyでTLS SECRETが使われたことを確認
```
kubectl apply -f httpproxy-app1-https.yaml

kubectl get httpproxy
```

curlによる動作確認 (※.kindクラスタ以外の場合、以下の「:30080」及び「:30443」の指定は必要ありません)
```
DOMAIN="あなたのドメイン"

curl -s $DOMAIN:30080 | grep '<title>'

curl -s https://$DOMAIN:30443 | grep '<title>'
```

今回作成したオブジェクトを削除
```
kubectl delete httpproxy app1

kubectl delete svc nginx-svc

kubectl delete pod nginx
```
