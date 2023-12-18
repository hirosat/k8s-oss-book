# 7章 - Kubernetesネットワークを使いこなそう

## 訂正・フィードバック

現在ありません

## リンク一覧

### ドメインの利用

- ドメインプロバイダ
  - [GoDaddy](https://www.godaddy.com/) : ドメイン登録数世界一の実績を誇る
  - [お名前.com](https://www.onamae.com/) : 日本最大のドメイン登録実績を誇る
  - [Freenom](https://www.freenom.com/) : 非推奨
    - [Unable to update DDNS using API for some TLDs](https://community.cloudflare.com/t/unable-to-update-ddns-using-api-for-some-tlds/167228) : DNSサービス側で、無料TLD (.cf / .ga / .gq / .ml / .tk) を禁止している事例
- [Supported DNS01 providers](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers) : **cert-menager**の**DNS連携機能**に対応するプロバイダ一覧
- [Cloudflare](https://www.cloudflare.com/)

### Ingressの利用

- [Service Proxy](https://landscape.cncf.io/guide#orchestration-management--service-proxy) : CNCF Landscape Guideより
- Ingress Controller の OSS 一覧
  - [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
  - [ingress-nginx](https://github.com/kubernetes/ingress-nginx) : K8s Community管理のIngress NGINX Controller
  - [kubernetes-ingress](https://github.com/nginxinc/kubernetes-ingress) : NGINX社管理のNGINX Ingress Controller
  - [Skipper](https://opensource.zalando.com/skipper/)
- [Contour](https://projectcontour.io/)
  - [Getting Started](https://projectcontour.io/getting-started/)

### CNIの検討

- [kindnet](https://github.com/aojea/kindnet)
- [Cloud Native Network](https://landscape.cncf.io/guide#runtime--cloud-native-network) : CNCF Landscape Guideより
- CNI の OSS 一覧
  - [Flannel](https://github.com/flannel-io/flannel)
  - [Weave Net](https://www.weave.works/oss/net/)
  - [Project Calico](https://www.tigera.io/project-calico/)
  - [Submariner](https://submariner.io/)
  - [Kube-router](https://www.kube-router.io/)
  - [Multus](https://github.com/k8snetworkplumbingwg/multus-cni)
  - [Kilo](https://kilo.squat.ai/)
  - [Kube-OVN](https://kubeovn.github.io/docs/en/)
  - [Antrea](https://antrea.io/)
- [Cilium](https://cilium.io/)
  - [Cilium Quick Instration](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)
  - [Cilium CLI Releaseページ](https://github.com/cilium/cilium-cli/releases) (GitHub)
  - [Setting up Hubble Observability](https://docs.cilium.io/en/stable/gettingstarted/hubble_setup/)
  - [Hubble CLI Releaseページ](https://github.com/cilium/hubble/releases) (GitHub)
  - [Service Map & Hubble UI](https://docs.cilium.io/en/stable/gettingstarted/hubble/)
  - [Getting Started with the Star Wars Demo](https://docs.cilium.io/en/stable/gettingstarted/demo/)

## コード集

### ドメインの利用
#### レコード登録とK8sとの連携

DNSレコードの正引き情報を問い合わせ (※. あなたのDNSレコードに置き換えましょう)
```
nslookup www.hirosat.com
```

ドメイン所有者情報の表示 (※. あなたのDNSレコードに置き換えましょう)
```
whois hirosat.com
```

ドメインを使ってアプリにアクセス (※. あなたのDNSレコードに置き換えましょう)
```
curl www.hirosat.com -s | grep Demo
```

#### ワイルドカードDNSレコード
ワイルドカードDNSレコードの挙動テスト (※. あなたのDNSレコードに置き換えましょう)
```
nslookup hoge.hirosat.com

nslookup my-app.hirosat.com

nslookup hoge.my-app.hirosat.com

nslookup fuga.my-app.hirosat.com
```


### Ingressの利用
#### Contourの導入
Contourのインストール
```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

作成されたオブジェクトを確認
```
kubectl get service -n projectcontour

kubectl get deploy -n projectcontour

kubectl get daemonset -n projectcontour
```

> **※. kindクラスタのみ、以下の対応が必要**<br>
> 以下の全行を一度に実行することで、envoy serviceのTYPEと、使用するNodePortの値が書き換わります。
> ```
> kubectl patch -n projectcontour service envoy --type='json' -p \
> '[{"op":"replace","path":"/spec/type","value":"NodePort"}, 
> {"op":"replace","path":"/spec/ports/0/nodePort","value":30080},
> {"op":"replace","path":"/spec/ports/1/nodePort","value":30443}]'
> ```
> Serviceの確認
> ```
> kubectl get service -n projectcontour envoy
> ```

#### Ingressリソースの使用

サンプルマニフェストのダウンロードと、ファイル内容の確認
```
curl -LO https://projectcontour.io/examples/httpbin.yaml

cat httpbin.yaml
```

マニフェストの適用と、オブジェクトの確認
```
kubectl apply -f httpbin.yaml

kubectl get po,svc,ing -l app=httpbin
```

出し分け用に、nginxとkuardのpodと、それに紐づくservice(type: ClusterIP)をデプロイ
```
kubectl run nginx --image=nginx

kubectl run kuard --image=gcr.io/kuar-demo/kuard-amd64:blue

kubectl expose pod nginx --port=80 --target-port=80 --name=nginx-svc

kubectl expose pod kuard --port=80 --target-port=8080 --name=kuard-svc
```

作成されたオブジェクトを確認
```
kubectl get pod,svc
```

Ingress ルールの追加 (※。以下の全文をまとめて実行することで、編集後の内容と同等になります。)
```
cat << EOF >> httpbin.yaml
  rules:
  - host: app1.foo.com
    http:
      paths:
      - backend:
          service:
            name: nginx-svc
            port:
              number: 80
        pathType: Prefix
        path: /
  - host: app2.bar.com
    http:
      paths:
      - backend:
          service:
            name: kuard-svc
            port:
              number: 80
        pathType: Prefix
        path: /
EOF
```

マニフェストの適用とIngressの確認
```
kubectl apply -f httpbin.yaml

kubectl get ingress
```

curlにホストヘッダを付けて**偽装**することで、DNS登録せずにIngressのルーティングを確認する方法
```
IP="外部IPアドレス(LB、または、NodePortの場合はノードIP)"

curl -sH "Host:app1.foo.com" $IP | grep '<title>'

curl -sH "Host:app2.bar.com" $IP | grep '<title>'

curl -sH "Host:hoge.com" $IP | grep '<title>'

curl -s $IP | grep '<title>'
```

#### ワイルドカードDNSレコードの活用
DNSレコードの内容に合わせて、httpbin.yaml を修正
```
DOMAIN="my-cluster.hirosat.com"  # 自分で用意したドメイン名、かつ、*を抜いてを入力

# MacOSの場合
sed -i '' -e s/foo.com/$DOMAIN/g -e s/bar.com/$DOMAIN/g httpbin.yaml

# Windows(Git Bash)の場合
sed -i -e s/foo.com/$DOMAIN/g -e s/bar.com/$DOMAIN/g httpbin.yaml

# 修正内容を確認
cat httpbin.yaml
```

マニフェストを適用
```
kubectl apply -f httpbin.yaml
```

動作確認
```
curl -s app1.$DOMAIN | grep '<title>'

curl -s app2.$DOMAIN | grep '<title>'

curl -s app3.$DOMAIN | grep '<title>'
```

> ※. kindクラスタの場合の確認方法
> ```
> curl -s app1.$DOMAIN:30080 | grep '<title>'
> 
> curl -s app2.$DOMAIN:30080 | grep '<title>'
> 
> curl -s app3.$DOMAIN:30080 | grep '<title>'
> ```

Ingressオブジェクトの削除
```
kubectl delete ingress httpbin
```

#### HTTPProxyリソースの利用
nginxアプリ用のDOMAIN変数を定義 (※. 任意のAPP名 + 用意したワイルドカードDNSドメイン となる値)
```
DOMAIN=app1.my-cluster.hirosat.com
```

HTTPProxyマニフェストを作成（※. 以下の全文をまとめて実行）
```
cat << EOF > httpproxy-app1.yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: app1
spec:
  virtualhost:
    fqdn: $DOMAIN
  routes:
    - conditions:
      - prefix: /
      services:
        - name: nginx-svc
          port: 80
EOF
```

マニフェストの適用と動作確認（※. kindクラスタは、**:30080** が必要）
```
kubectl apply -f httpproxy-app1.yaml

curl -s (外部IP) | grep '<title>'

curl -s app1.my-cluster.hirosat.com | grep '<title>'
```

HTTPProxyオブジェクトの確認
```
kubectl get httpproxy
```

（検証）同じドメインを持つマニフェストを用意して適用
```
cp httpproxy-app1.yaml httpproxy-app1-same.yaml

vi httpproxy-app1-same.yaml
(metadata.name を app1-same に変更)
  name: app1-same

kubectl apply -f httpproxy-app1-same.yaml
```

エラー状況の確認
```
kubectl get httpproxy

kubectl describe httpproxy app1 | grep Message
```

app1-sameを削除して、正常に戻す
```
kubectl delete httpproxy app1-same

kubectl get httpproxy
```

#### HTTPProxyリソースによるパスの振り分け

kuardアプリおよびhttpbinアプリ用のDOMAIN変数を定義 (※. 任意のAPP名 + 用意したワイルドカードDNSドメイン となる値)
```
DOMAIN=app2.my-cluster.hirosat.com
```

HTTPProxyマニフェストを作成（※. 以下の全文をまとめて実行）

```
cat << EOF > httpproxy-app2.yaml
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: app2
spec:
  virtualhost:
    fqdn: $DOMAIN
  routes:
    - conditions:
      - prefix: /
      services:
        - name: kuard-svc
          port: 80
    - conditions:
      - prefix: /api
      services:
        - name: httpbin
          port: 80
      pathRewritePolicy:
        replacePrefix:
        - prefix: /api
          replacement: /
EOF
```

マニフェストの適用と確認
```
kubectl apply -f httpproxy-app2.yaml

kubectl get httpproxy
```

動作確認
```
curl -s $DOMAIN | grep '<title>'

curl -s $DOMAIN/api | grep '<title>'

curl -s $DOMAIN/api/ip
```

> **kindクラスタ**の場合の動作確認
> ```
> curl -s $DOMAIN:30080 | grep '<title>'
> 
> curl -s $DOMAIN:30080/api | grep '<title>'
> 
> curl -s $DOMAIN:30080/api/ip
> ```

今までに作成したオブジェクトを削除
```
kubectl delete httpproxy app1 app2

kubectl delete service httpbin kuard-svc nginx-svc

kubectl delete deploy httpbin

kubectl delete pod kuard nginx
```

### CNIの検討

#### Ciliumを導入する

※. 詳細は、[Cilium Quick Instration](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/) をご確認下さい。

クラスタの準備

- kindの場合 : [Cilium用のkindクラスタ設定](./kind-cilium.md) を用意しました。
- kubeadmの場合 : calicoの削除
```
kubectl delete -f  https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
```

Cilium CLIのインストール
- CLI_VER : [Releaseページ](https://github.com/cilium/cilium-cli/releases) の最新バージョン
- ARCH : Windowsは「windows-amd64」、 Macは「darwin-arm64」
- DEST : パスが通っていれば、どこでも可。推奨は、Windowsは「~/bin」、 Macは「/usr/lobal/bin」

```
CLI_VER=v0.15.18

ARCH=windows-amd64

DEST="~/bin"

curl -LO https://github.com/cilium/cilium-cli/releases/download/$CLI_VER/cilium-$ARCH.tar.gz

sudo tar xzvfC cilium-$ARCH.tar.gz $DEST

cilium version
```

K8sクラスタにインストール
```
cilium install
```

ciliumの動作状況を確認
```
cilium status --wait
```

(おまけ) 接続性テスト
```
cilium connectivity test
```

#### Hubbleを導入する

※. 詳細は、[Setting up Hubble Observability](https://docs.cilium.io/en/stable/gettingstarted/hubble_setup/) をご確認下さい。

Hubbleの有効化
```
cilium hubble enable
```

Hubble Relay項目の状態を確認
```
cilium status
```

Hubble CLIのインストール

- CLI_VER : [Releaseページ](https://github.com/cilium/hubble/releases) の最新バージョン
- ARCH : Windowsは「windows-amd64」、 Macは「darwin-arm64」
- DEST : パスが通っていれば、どこでも可。推奨は、Windowsは「~/bin」、 Macは「/usr/lobal/bin」

```
HUBBLE_VER=v0.12.3

ARCH=windows-amd64

DEST="~/bin"

curl -LO https://github.com/cilium/hubble/releases/download/$HUBBLE_VER/hubble-$ARCH.tar.gz

sudo tar xzvfC hubble-$ARCH.tar.gz $DEST

hubble version
```

Hubble APIへの接続方法
```
cilium hubble port-forward&
```

Hubbleの状態を確認
```
hubble status
```

直近のFlow処理内容を確認
```
hubble observe
```

#### Hubble UI の使い方

※. 詳細は、[Service Map & Hubble UI](https://docs.cilium.io/en/stable/gettingstarted/hubble/) をご確認下さい。

Hubbleの無効化
```
cilium hubble disable
```

Hubble UIの有効化
```
cilium hubble enable --ui
```

Hubble UIの起動
```
cilium hubble ui
```

#### Cilium の基本操作

※. 詳細は、[Getting Started with the Star Wars Demo](https://docs.cilium.io/en/stable/gettingstarted/demo/) をご確認下さい。

デモアプリのデプロイ
```
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml
```

デプロイされたオブジェクトを確認
```
kubectl get pod

kubectl get svc
```

Cilium Podの確認 (Workerノード側のPod名を確認)
```
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
```

Workor Node側のcilium podで、`cilium endpoint list`を実行
```
PODNAME=(前のコマンドで調べたpod名)
kubectl -n kube-system exec $PODNAME -- cilium endpoint list
```

xwingとtiefighterの各Podから、curlでdeathstarに着陸リクエストを送信
```
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

Hubble UIの起動
```
cilium hubble ui
```
確認後は、**Ctrl+C** で停止

#### L3/L4ポリシーの適用


CiliumNetworkPolicyのマニフェストを作成 (※. 以下の全文をまとめて実行)
```
cat << EOF > cilium-policy-l4.yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L3-L4 policy to restrict deathstar access to empire ships only"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
EOF
```

マニフェストの適用
```
kubectl apply -f cilium-policy-l4.yaml
```

再度、各Podから、curlでdeathstarに着陸リクエストを送信
```
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
(※. 2つ目のコマンドは応答が返らないので、**Ctrl+C** でリクエストを終了させます）

Hubble UIの起動 (※. 上記のコマンドとは別端末にしたほうが確認しやすい)
```
cilium hubble ui
```
確認後は、**Ctrl+C** で停止

再度、Workor Node側のcilium podで、`cilium endpoint list`を実行
```
kubectl -n kube-system exec $PODNAME -- cilium endpoint list
```

#### L7ポリシーの適用

tiefighter podから、curlでメンテナンス用APIにリクエスト
```
kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port
```

L7レベルの制御を行うポリシーのマニフェストを作成
```
cat << EOF > cilium-policy-l7.yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "rule1"
spec:
  description: "L7 policy to restrict access to specific HTTP call"
  endpointSelector:
    matchLabels:
      org: empire
      class: deathstar
  ingress:
  - fromEndpoints:
    - matchLabels:
        org: empire
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "POST"
          path: "/v1/request-landing"
EOF
```

マニフェストの適用
```
kubectl apply -f cilium-policy-l7.yaml
```

再度、いくつかのcurlリクエストを実施
```
kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

kubectl exec tiefighter -- curl -s -XPUT deathstar.default.svc.cluster.local/v1/exhaust-port

kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```

デモアプリの削除
```
kubectl delete cnp rule1

kubectl delete -f https://raw.githubusercontent.com/cilium/cilium/HEAD/examples/minikube/http-sw-app.yaml

```
