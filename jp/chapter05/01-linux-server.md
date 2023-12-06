# （パターン1） Linuxサーバの利用

## 概要

### リンク一覧
- [kubeadm 概要](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) ( [日本語](https://kubernetes.io/ja/docs/reference/setup-tools/kubeadm/) )
  - [Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) ( [日本語](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/) ) : kubeadm を使ってクラスタ構築するための様々な方法を紹介

## 1. ハードウェアの準備

### リンク一覧

- [Amazon (通販サイト)](https://www.amazon.co.jp/)
- [MINISFORUM (通販サイト)](https://store.minisforum.jp/)
- [NUCで始めるVMware Tanzu](https://qiita.com/advent-calendar/2020/nuc-vmware-tanzu)
  - [(2日目) NUC紹介](https://qiita.com/hirosat/items/9b9885f229d696ca1788) : ベアボーン用パーツの選定方法を紹介
  - [(4日目) NUCのセットアップ](https://qiita.com/hirosat/items/bd4c67f7d248cef640db) : ベアボーンにパーツをセットアップする様子を紹介
- [VMware vSphere Hypervisor 8 製品評価センター](https://customerconnect.vmware.com/jp/evalcenter?p=free-esxi8) : ESXi(isoファイル)のダウンロードが可能(要アカウント作成)
- [VMUG](https://www.vmug.com/)
  - [VMUG Advantage](https://www.vmug.com/membership/vmug-advantage-membership/)

## 2. ESXiのインストール

### リンク一覧

- [VMware vSphere Hypervisor 8 製品評価センター](https://customerconnect.vmware.com/jp/evalcenter?p=free-esxi8) : ESXi(isoファイル)のダウンロードが可能(要アカウント作成)
  - [やってみよう！ 初めての無償版ESXi](https://blogs.vmware.com/vmware-japan/2014/05/vspherehypervisor.html) : 無償版ESXiの基礎知識に関するブログ
- [NUCで始めるVMware Tanzu](https://qiita.com/advent-calendar/2020/nuc-vmware-tanzu)
  - [(5日目) ESXiのセットアップ](https://qiita.com/hirosat/items/e78d75f0032f0f1369f0) : MacでESXiをセットアップする例を紹介
- [はじめよう、おうちクラウド](https://github.com/tuna-jp/ouchi-cloud/tree/main) : Software Design 誌に掲載された連載、「(VMware x Kubernetes) **はじめよう、おうちクラウド**」を紹介したGitHubサイト。筆者も執筆に携わっている。
  - [第3回 仮想化基盤を作ってみよう](https://github.com/tuna-jp/ouchi-cloud/blob/main/vol3/README.md)
    - [ESXiのインストールと基本設定](https://github.com/tuna-jp/ouchi-cloud/blob/main/vol3/03_esxi_setup.md) : ESXiのインストールに加え、ESXiの初期設定やvSphere Clientの初期操作についても、詳しく書かれています。

### 補足
Windowsで、ダウンロードしたISOファイルを**USBメモリ**に展開するには、**Rufus** や **UNetbootin** といったフリーソフトを利用すると、簡単に作成できるはずです。

- [Rufus](https://rufus.ie/ja/)
- [UNetbootin](https://unetbootin.github.io/)

## 3. ゲストOS (Ubuntu) のインストール

> 対象 : **Control Plane用** と **Worker用** の2VM分を、同様の手順で用意します。

別ページにセットアップ方法を用意しました。

- [ESXi上にUbuntuをセットアップする方法](./01a-ubuntu-setup.md)

## 4. 必要パッケージのインストール

> 対象 : **Control Plane VM** および、 **Worker VM** 

### リンク一覧

- [K8s公式ページ](https://kubernetes.io/)
  - [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) ([日本語版](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)) : こちらの手順に沿って進める
  - [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) ([日本語版](https://kubernetes.io/ja/docs/setup/production-environment/container-runtimes/)) : コンテナランタイムに関しては、こちらを参照
- [docker docs](https://docs.docker.com/)
  - [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) : Docker用aptリポジトリ登録の方法を参照する

### カーネルモジュールのロード

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF 

sudo modprobe overlay && sudo modprobe br_netfilter
```
動作確認
``` 
lsmod | grep -e br_netfilter -e overlay
```

### カーネルパラメータを設定

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```
動作確認 (期待値: 全ての値が1となっている)
```
$ sysctl net.bridge.bridge-nf-call-iptables net.bridge.
```

### Dockerの公式GPGキーを追加
```
sudo apt-get update && sudo apt install -y ca-certificates curl gnupg

curl -fsSL https://download.docker.com/linux/ubuntu/gpg |\
 sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Dockerレポジトリの追加

```
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu jammy stable" |\
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### containerd のインストール
```
$ sudo apt-get update && sudo apt-get install -y containerd.io
```
### containerd 設定 (systemd cgroupドライバを追加)
```
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd && sudo systemctl enable containerd
```
動作確認 (Runningとなっていることを確認)
```
$ sudo systemctl status containerd
```
### Kubernetes レポジトリの追加
```
sudo apt-get install -y apt-transport-https

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key |\
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" |\
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Kubernetes関連ツールのインストール
```
sudo apt update && sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

## 5. Kubernetesクラスタのセットアップ

> 対象 : **Control Plane VM** のみ

### kubeadm init
```
CONTROL_PLANE_IP=(Control planeのノードのIPを入力)

sudo kubeadm init --control-plane-endpoint=${CONTROL_PLANE_IP}:6443 
```

### kubeconfigの登録
```
mkdir -p $HOME/.

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### kubectl CLI 動作確認
クラスタの稼働状態
```
kubectl cluster-info
```

ノード一覧
```
kubectl get node
```

Pod一覧
```
kubectl get pod -A
```

## 6. CNIの導入

> 対象 : **Control Plane VM** のみ

### リンク一覧

- [Calico公式ページ](https://www.tigera.io/project-calico/)

### calicoのインストール
```
kubectl apply -f  https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
```

#動作確認
```
kubectl get node

kubectl get pod -A -o wide
```

### (おまけ) calico のCIDR情報確認
```
kubectl cluster-info dump | grep cidr 
```


## 7. Kubernetesクラスタへの参加

> 対象 : **Worker VM** のみ

### worker ノード追加
```
CONTROL_PLANE_IP=(Control planeのノードのIPを入力)

sudo kubeadm join ${CONTROL_PLANE_IP}:6443 --token (kubectl init時の結果)
```

動作確認
```
kubectl get node
```


## 8. ローカル環境からの接続

### kubeconfigを、Control Planeからローカル環境にコピー
```
mkdir -p $HOME/.kube

CP_IP="あなたの環境のControl Plane IPアドレス"

scp ubuntu@${CP_IP}:~/.kube/config ~/.kube/config

sudo chown $(id -u):$(id -g) ~/.kube/config
```

### クラスタ情報の確認

```
kubectl cluster-info
```
