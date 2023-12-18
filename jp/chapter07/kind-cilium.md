# Cilium用のkindクラスタ設定

Ciliumを入れるためのクラスタの準備として、通常だと**kindnet**がCNIとして使われるため、**CNIを抜いたクラスタ**を用意する必要があります。

## コード集

#### kindのクラスタ設定を用意

[ciliumのkind用クラスタ設定](https://raw.githubusercontent.com/cilium/cilium/main/Documentation/installation/kind-config.yaml)を参照すると、ポイントは`disableDefaultCNI: true` の設定が必要なため、既存の設定にマージします。

 (※. 以下の全行をまとめて実行すると、編集後の内容と同等になります。)
```
cat << EOF > kind-cni-disable.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
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

#### kindクラスタの再作成
```
kind delete cluster

kind create cluster --config kind-cni-disable.yaml
```

