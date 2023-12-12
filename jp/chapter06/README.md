# 6章 - Kubernetesを使おう

## 訂正・フィードバック

現在ありません

## リンク一覧

- [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/) : 全APIの説明を網羅
- [kuard](https://github.com/kubernetes-up-and-running/kuard) : Demo用アプリケーション
- [Kubernetes Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) : Jobリソースのサンプルと解説
- [GKEドキュメント](https://cloud.google.com/kubernetes-engine/docs?hl=ja)
  - [NodePortタイプのServiceの作成](https://cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps?hl=ja#creating_a_service_of_type_nodeport)
  - [LoadBalancerタイプのServiceの作成](https://cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps?hl=ja#creating_a_service_of_type_loadbalancer)
- [MetalLB](https://metallb.universe.tf/)
  - [INSTALLATION](https://metallb.universe.tf/installation/)
  - [Layer 2 Congiruration](https://metallb.universe.tf/configuration/)

## コード集

### Kubernetesクラスタへの接続方法

~/.kube/config の内容確認
```
cat ~/.kube/config
```

コンテキスト一覧を表示 (*が付与されているのが、現在のコンテキスト)
```
kubectl config get-contexts
```

コンテキストの切り替え
```
kubectl config use-context kubernetes-admin@kubernetes
```

現在のコンテキスト名を表示
```
kubectl config current-context
```

<br><br>

### Kubernetes操作の考え方
#### 1. kubectl の使い方を調べる

kubectlで利用可能なサブコマンド一覧
```
kubectl -h
```

サブコマンドを指定した詳細なhelpの例
```
kubectl run -h
```

#### 2. Podを作成してみる

NginX Podの作成
```
kubectl run nginx --image=nginx
```

対象イメージを、正式なフォーマットで指定 (※. 直前のコマンドを実行している場合、既にPodが存在するためエラーとなる)
```
kubectl run nginx --image=index.docker.io/library/nginx:latest
```

Podの状態を表示
```
kubectl get pod
```

#### 3. YAML化する (kubernetesマニフェスト)

Podを作成するマニフェストをファイルに保存
```
kubectl run nginx2 --image=nginx -o yaml --dry-run=client > nginx2.yaml
```

ファイル内容の表示
```
cat nginx2.yaml
```

マニフェストの適用
```
kubectl apply -f nginx2.yaml
```

Podの状態を表示
```
kubectl get pod
```

#### 4. 詳細仕様の付与
対象Pod内でenvコマンドを実行
```
kubectl exec nginx2 -- env
```

マニフェストファイルの内容を表示
```
cat nginx2.yaml
```

利用可能なapiVersionとkindの一覧
```
kubectl api-resources
```

K8sリソース内の利用可能なフィールドの説明
```
kubectl explain pod.spec
```

K8sリソース内の利用可能なフィールドの説明（全フィールドを再帰的に表示）
```
kubectl explain pod.spec --recursive
```

**env**の名前の付くフィールドを探す
```
kubectl explain pod.spec | grep -E "^\s+env"

kubectl explain pod.spec.containers | grep -E "^\s+env"
```

envフィールドの説明を読む
```
kubectl explain pod.spec.containers.env --recursive
```

マニフェストの編集 (※. 以下の全行をまとめて実行すると、編集後の内容と同等になります。)
```
cat << EOF > nginx2.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx2
  name: nginx2-mod
spec:
  containers:
  - image: nginx
    name: nginx2
    env:
    - name: APPNAME
      value: myapp
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
EOF
```

編集したマニフェストを再び適用
```
kubectl apply -f nginx2.yaml
```

Pod一覧の表示
```
kubectl get pod
```

nginx2-pod 内でenvコマンドを実行 (※. 結果の中に、APPNAME=myapp が入っていたら成功)
```
kubectl exec nginx2-mod -- env
```

<br><br>

### Podへのアクセス方法
#### 1. Podに直接ログイン
nginx Pod内に入る方法
```
kubectl exec nginx -i -t -- bash
```

(nginx pod内) localhostのコンテンツを取得
```
curl localhost
```

(nginx pod内) ログアウト
```
exit
```


#### 2. 別Podからアクセス
Podの内部IPアドレスを調べる方法
```
kubectl get pod -o wide
```

調査用Podを起動しつつログイン
```
kubectl run -i -t --rm --image=brix4dayz/swiss-army-knife -- bash
```

(調査用Pod内) nginx Podのコンテンツを取得<br>
(※. カッコ内は、冒頭で調べたIPアドレスを使う)
```
curl (nginx podの内部IP)
```

(調査用Pod内) ログアウト
```
exit
```

#### 3. 開発マシンにPort開放する
クライアント上のPortに転送
```
kubectl port-forward pod/nginx 8080:80
```

ブラウザのアドレスバー内で、以下のアドレスにアクセス
```
localhost:8080
```
アクセスを確認できたら、`kubectl port-forward`している端末上で**Ctrl+C**をして停止しましょう。

<br><br>

### Podの削除方法
```
kubectl delete pod nginx nginx2 nginx2-mod
```

<br><br>

### KubernetesのWorkloadリソースについて
#### Pod / ReplicaSet / Deployment
Deploymentのマニフェストを作成して保存
```
kubectl create deployment my-dep --image=gcr.io/kuar-demo/kuard-amd64:blue --replicas=2 --dry-run=client -o yaml > my-dep.yaml
```

ファイル内容を表示
```
cat my-dep.yaml
```

マニフェストを適用
```
kubectl apply -f my-dep.yaml
```

Pod / ReplicaSet / Deployment の表示
```
kubectl get pod

kubectl get replicaset

kubectl get deployment
```

Port転送
```
kubectl port-forward deployment/my-dep 8080:8080
```
※. ブラウザ確認後は、**Ctrl+C** で停止

DeploymentのScale (Pod数の変更)
```
kubectl scale deployment my-dep --replicas=3
```

Deploymentマニフェストの更新 (※. 以下の全行をまとめて実行すると、編集後の内容と同等になります。)
```
cat << EOF > my-dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: my-dep
  name: my-dep
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-dep
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: my-dep
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard-amd64
        env:
        - name: APPNAME
          value: myapp
        resources: {}
status: {}
EOF
```

(別端末) watchコマンドで監視
```
watch kubectl get pod
```

現在のRevision(Deploymentの世代)番号を確認
```
kubectl rollout history deployment/my-dep
```

マニフェストの適用
```
kubectl apply -f my-dep.yaml
```

Podとリビジョンの履歴を確認
```
kubectl get pod

kubectl rollout history deployment/my-dep
```

過去のリビジョンに戻る
```
kubectl rollout undo deployment/my-dep

kubectl rollout history deployment/my-dep

kubectl get pod
```

Deploymentの削除
```
kubectl delete -f my-dep.yaml
```

<br><br>

#### Job / CronJob

(別端末) Pod状態の監視
```
kubectl get pod -w
```

Jobの作成 (πを2000桁計算)
```
kubectl create job pi --image=perl:5.34.0 -- perl "-Mbignum=bpi" "-wle" "print bpi(2000)"
```

Jobの一覧
```
kubectl get job
```

Podのログ表示 (計算結果の確認)
```
kubectl get pod

kubectl logs (Pod名)
```

Jobの削除
```
kubectl delete jobs pi

kubectl get pod,jobs
```

CronJobの作成
```
kubectl create cronjob my-job --image=busybox --schedule="*/1 * * * *" -- date
```

CronJobの一覧
```
kubectl get cronjobs
```

Job / Pod / ログの確認
```
kubectl get jobs

kubectl get pod

kubectl logs (Pod名)
```

CronJobの削除
```
kubectl delete cronjob my-job

kubectl get cronjobs,jobs,pod
```

<br><br>

#### DaemonSet
DaemonSetの作成
```
cat << EOF > ds-nginx.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
EOF
```

マニフェストの適用
```
kubectl apply -f ds-nginx.yaml
```

DeamonSet, Podの確認
```
kubectl get daemonsets

kubectl get pod -o wide
```

Workload配置対象の除外設定の確認<br>
(※. Taint設定に`NoSchedule`が入っているノードにはWorkloadが配置されない。<br>
Taintの順番は、kubectl get nodeの表示順。)
```
kubectl get node

kubectl describe node | grep Taint
```

DaemonSetの削除
```
kubectl delete daemonsets ds-nginx

kubectl get daemonsets,pod
```

<br><br>

### KubernetesのServiceリソースについて
#### Service type: ClusterIP 

Service公開対象のアプリ(Deployment)を作成
```
kubectl create deployment my-app --image=gcr.io/kuar-demo/kuard-amd64:blue --replicas=3
```

Service用マニフェストを作成して保存
```
kubectl expose deployment my-app --port=80 --target-port=8080 --dry-run=client -o yaml > service-my-app.yaml
```

マニフェスト内容を確認
```
cat service-my-app.yaml
```

マニフェストを適用
```
kubectl apply -f service-my-app.yaml
```

service, endpoint を確認
```
kubectl get service

kubectl get endpoints
```

調査用Podを作成しつつ起動
```
kubectl run -i -t --rm --image=brix4dayz/swiss-army-knife -- bash
```

(調査用Podで実行) DNS確認、および、DNS名を対象に指定してコンテンツの取得
```
nslookup my-app

curl -s my-app | grep my-app
```

<br><br>


#### Service type: NodePort

> ※. kindクラスタを利用している方は、事前に以下の手順を実施して下さい。

[NodePortに対応したkindクラスタ設定](./kind-nodeport.md)


> 以下、共通手順

先程のkuardアプリを対象に、追加でNodePortのサービス用のマニフェストを作成・保存
```
kubectl expose deployment my-app --port=80 --target-port=8080 --name=my-app2 --type=NodePort --dry-run=client -o yaml > service-my-app2.yaml
```

特定のNodePortを指定 (※. 以下の全行をまとめて実行すると、編集後の内容と同等になります。)
```
cat << EOF > service-my-app2.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-app
  name: my-app2
spec:
  ports:
  - port: 80
    nodePort: 30080
    protocol: TCP
    targetPort: 8080
  selector:
    app: my-app
  type: NodePort
EOF
```

マニフェストの適用
```
kubectl apply -f service-my-app2.yaml
```

service一覧を確認
```
kubectl get service
```

<br><br>

#### Service type: LoadBalancer

> ※. 以下の手順は、**kubeadm** のK8sクラスタのみが対象です。

MetalLBのインストール <br>
(※. コマンド内のバージョン(v0.13.7)は、[ドキュメント](https://metallb.universe.tf/installation/#installation-by-manifest)を参考に、最新版に差し替えて下さい)
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

IP Pool設定のマニフェストの用意 (※. (IP address)の部分は、お使いの環境により異なります)
```
cat << EOF > first-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - (IP address)-(IP address)
EOF
```

Advertise用のマニフェストの用意
```
cat << EOF > l2adv.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
EOF
```

マニフェストの適用
```
kubectl apply -f first-pool.yaml l2adv.yaml
```

Service type:LoadBalancer用マニフェストの作成
```
kubectl expose deployment my-app --port=80 --target-port=8080 --name=my-app3 --type=LoadBalancer --dry-run=client -o yaml > service-my-app3.yaml
```

マニフェストの適用とService確認
```
kubectl apply -f service-my-app3.yaml

kubectl get service
```

今まで作成したオブジェクトの削除
```
kubectl delete service my-app my-app2 my-app3

kubectl delete deployment my-app
```