# （パターン3） ローカル環境の利用

## リンク一覧

- [Kubernetes 公式サイト](https://kubernetes.io/ja/)
  - [Install Tools](https://kubernetes.io/docs/tasks/tools/) ([日本語](https://kubernetes.io/ja/docs/tasks/tools/)) : ローカル環境上でK8sを実行するツールを紹介
- [kind 公式サイト](https://kind.sigs.k8s.io/)
  - [Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) : kindの始め方を紹介
- [kind GitHubページ]()
  - [Releaseページ](https://github.com/kubernetes-sigs/kind/releases) : kind バイナリの配布元
  - [Issue#3277](https://github.com/kubernetes-sigs/kind/issues/3277) : Rancher-Desktop と kind v0.20.0 の組み合わせでクラスタが作成できない問題が報告されている
- [minikube 公式サイト](https://minikube.sigs.k8s.io/docs/)

## コード集

### kind のインストール

Windowsの場合

```
# ダウンロード
curl -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.19.0/kind-windows-amd64

# Pathの通った場所に移動
mv ./kind-windows-amd64.exe ~/bin/kind.exe
```

Macの場合

```
# ダウンロード
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.19.0/kind-darwin-arm64

# 実行権を付与
chmod +x ./kind

# PATHの通った場所に移動
mv ./kind /usr/local/bin/kind
```

動作確認

```
kind version
```

### マルチノード設定

設定ファイルの用意 (以下、全ての行をまとめて実行する)

```
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF
```

### kind クラスタの作成

クラスタ作成
```
kind create cluster --config kind-config.yaml
```

kindクラスタの一覧表示
```
kind get clusters
```

### ローカル環境のkubectlで動作確認

K8sクラスタ情報の取得
```
kubectl cluster-info --context kind-kind
```

ノード一覧の確認
```
kubectl get node
```