# NodePortに対応したkindクラスタ設定
kindクラスタは**Docker Network**上に存在します。

NodePortを利用するには、**ローカルPCにport転送**する必要があるため、事前に以下を実施しましょう。

## コード集


#### kindのクラスタ設定を用意

 (※. 以下の全行をまとめて実行すると、編集後の内容と同等になります。)
```
cat << EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 30080
    hostPort: 30080
  - containerPort: 30443
    hostPort: 30443
EOF
```
- **containerPort** : NodePort用に使用する、30000-32767の範囲内の**任意のport番号**
- **hostPort** : ローカルPCに転送する**任意のport番号**。 ※. ローカルPCで既に使われているport番号は使用できないので注意。

上記の例では、http(80)アクセス用とhttps(443)アクセス用に**2つの接続用port**を用意しています。もし **複数の「type:NodePort」** を扱いたい場合、必要数分のportを**事前に用意**しておく必要があります。

#### kindクラスタの再作成
```
kind delete cluster

kind create cluster --config kind-config.yaml
```

#### DeploymentとServiceを再作成
```
kubectl create deployment my-app --image=gcr.io/kuar-demo/kuard-amd64:blue --replicas=3

kubectl apply -f service-my-app.yaml
```