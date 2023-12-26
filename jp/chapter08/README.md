# 8章 - Kubernetesのデータを管理しよう

## 訂正・フィードバック

現在ありません

## リンク一覧

- [The Twelve-Factor App](https://12factor.net/ja/)
- [K8sドキュメント](https://kubernetes.io/ja/docs/home/)
  - [Volume](https://kubernetes.io/ja/docs/concepts/storage/volumes/)
  - [Persistent Volumes](https://kubernetes.io/ja/docs/concepts/storage/persistent-volumes/)
  - [StatefulSet](https://kubernetes.io/ja/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes-CSIドキュメント](https://kubernetes-csi.github.io/docs/)
  - [CSI driver一覧](https://kubernetes-csi.github.io/docs/drivers.html)
- [CSI Hostpath driver](https://github.com/kubernetes-csi/csi-driver-host-path)
  - [デプロイ方法](https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/deploy-1.17-and-later.md)
- [CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter)
- [CNCF Cloud Native Storage](https://landscape.cncf.io/guide#runtime--cloud-native-storage)
  - [MinIO](https://min.io/)
  - [Ceph](https://ceph.io/en/)
  - [Rook](https://rook.io/)
  - [OpenEBS](https://openebs.io/)
  - [JuiceFS](https://juicefs.com/en/)
  - [Velero](https://velero.io/)
  - [Alluxio](https://www.alluxio.io/)
  - [Longhorn](https://longhorn.io/)
  - [Gluster](https://www.gluster.org/)
  - [CubeFS](https://cubefs.io/)
  - [Swift](https://docs.openstack.org/swift/latest/)
  - [Curve](https://www.opencurve.io/)
  - [MooseFS](https://moosefs.com/)

## コード集

### etcdについて
#### Configmapの作成

ConfigMapマニフェストの用意
```
cat << EOF > cm-sample1.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample1
data:
  player_name: "John Doe"    	# Propaty format sample
  game.properties: |    	# File format sample
    player.max-lives=5
  db-config-json: |    	# JSON format sample
    {
      "Type":  "MySQL",
      "API_ENDPOINT": {
        "dev": "dev-api.example.com",
        "prd": "api.example.com"
      }
    }
  player.yaml: |   	       # YAML format sample
    job: fighter
    env:
    - name: LANGUAGE
      Value: "English"
EOF
```

マニフェストの適用と確認
```
kubectl apply -f cm-sample1.yaml

kubectl get configmaps
```

ConfigMapのデータを確認
```
kubectl describe configmap sample1
```

サンプルの設定データを用意
```
cat << EOF > my-values.txt
USERNAME="Andy"
PASSWORD=password123
EOF
```

**--from-file** オプションを使用して、ConfigMapを作成
```
kubectl create configmap sample2 --from-file=my-values.txt

kubectl describe cm sample2
```

**--from-env-file** オプションを使用して、ConfigMapを作成
```
kubectl create configmap sample3 --from-env-file=my-values.txt

kubectl describe cm sample3
```

**--from-literal** オプションを使用して、ConfigMapを作成

```
kubectl create configmap sample4 --from-literal=HOSTNAME=myServer

kubectl describe configmaps sample4
```

#### Secretリソースの使用

Base64エンコード、および、デコード方法
```
echo "Secure Data" | base64

echo U2VjdXJlIERhdGEK | base64 -d
```

Secretマニフェストの作成 (※. 全ての行をまとめて実行)
```
cat << EOF > secret1.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret1
type: Opaque
data:
  secure-info: U2VjdXJlIERhdGEK
stringData:
  phone-number: "0123456789"
EOF
```

マニフェストの適用
```
kubectl apply -f secret1.yaml
```

Secretオブジェクトの確認と詳細
```
kubectl get secrets

kubectl describe secrets secret1
```

Secretに登録されたvalueを確認する方法
```
kubectl get secrets secret1 -o yaml
```

phone-numberキーのデコード
```
echo MDEyMzQ1Njc4OQ== | base64 -d
```

#### Secret作成コマンド

type: Opaque のSecretを作成
```
kubectl create secret generic secret2 --from-file=my-values.txt

kubectl get secret secret2 -o yaml

kubectl get secret secret2 -o jsonpath='{.data.my-values\.txt}' | base64 -d
echo VVNFUk5BTUU9IkFuZHkiClBBU1NXT1JEPXBhc3N3b3JkMTIzCg== | base64 -d
```

Basic認証のSecretを作成 (※. 最初のコマンドは、2行まとめて実行)
```
kubectl create secret generic secret3 --type=kubernetes.io/basic-auth \
--from-literal=username=ftpuser --from-literal=password=ftp123

kubectl get secret secret3 -o yaml
```

コンテナレジストリの資格情報Secretを作成 (※. 最初のコマンドは、2行まとめて実行)
```
kubectl create secret docker-registry secret4 --docker-server=mysite.com \
 --docker-username=admin --docker-password=p@ssw0rd

kubectl get secret secret4 -o yaml

kubectl get secret secret4 -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

#### ConfigMap / Secret の利用

##### 1. 環境変数として渡す
ConfigMapを利用する、Podマニフェストの作成 (※. 全ての行をまとめて実行)
```
cat << EOF > pod-env-cm.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-cm
spec:
  containers:
  - name: pod-env-cm
    image: nginx
    env:
    - name: USERINFO
      valueFrom:
        configMapKeyRef:
          name: sample1
          key: player_name
    envFrom:
    - configMapRef:
        name: sample2
EOF
```

マニフェストの適用
```
kubectl apply -f pod-env-cm.yaml
```

Podにログイン
```
kubectl exec -it pod-env-cm -- bash
```

Pod内で実行するコマンド
```
printenv USERINFO

printenv my-values.txt

exit
```

Secretを利用する、Podマニフェストの作成と適用 (※. 全ての行をまとめて実行)
```
cat << EOF > pod-env-secret.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-secret
spec:
  containers:
  - name: pod-env-secret
    image: nginx
    envFrom:
    - secretRef:
        name: secret1
    - secretRef:
        name: secret4
EOF
```

マニフェストの適用
```
kubectl apply -f pod-env-secret.yaml
```

Podにログイン
```
kubectl exec -it pod-env-secret -- bash
```

Pod内で実行するコマンド
```
printenv phone-number

printenv secure-info

printenv .dockerconfigjson

exit
```

##### 2. 設定をボリュームマウント

ボリュームマウント用のマニフェストを用意
```
cat << EOF > pod-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume
spec:
  containers:
  - name: pod-volume
    image: nginx
    volumeMounts:
    - name: cm-volume
      mountPath: /config
    - name: secret-volume
      mountPath: /etc/secrets
  volumes:
  - name: cm-volume
    configMap:
      name: sample1
  - name: secret-volume
    secret:
      secretName: secret1
EOF
```

マニフェストの適用
```
kubectl apply -f pod-volume.yaml
```

Podにログイン
```
kubectl exec -it pod-volume -- bash
```

Pod内で実行するコマンド
```
ls /config

cat /config/db-config-json

ls /etc/secrets

cat /etc/secrets/phone-number

exit
```

#### ConfigMap / Secret の更新

ConfigMap設定の更新
```
kubectl get cm sample1 -o jsonpath='{.data.player_name}'

kubectl patch cm sample1 --type merge -p '{"data":{"player_name":"Smith"}}'

kubectl get cm sample1 -o jsonpath='{.data.player_name}'
```

ボリュームマウントしたPod上のConfigMap設定を確認
```
kubectl exec pod-volume -- cat /config/player_name
```

環境変数で設定したPodのConfigMap設定を確認
```
kubectl exec pod-env-cm -- printenv player_name
```

今まで作成したオブジェクトの削除
```
kubectl delete pod pod-env-cm pod-env-secret pod-volume

kubectl delete cm sample1 sample2 sample3 sample4

kubectl delete secrets secret1 secret2 secret3 secret4
```

### K8sのストレージ
#### 1. PVの動的プロビジョニング

##### ①. CSI Driverのインストール

[CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter) (VolumeSnapshot CRDsとSnapshot Contoller)をインストール
```
git clone git@github.com:kubernetes-csi/external-snapshotter.git

kubectl create -k external-snapshotter/client/config/crd/

kubectl create -n kube-system -k external-snapshotter/deploy/kubernetes/snapshot-controller/
```

CSI Hostpath driverのインストール
```
git clone git@github.com:kubernetes-csi/csi-driver-host-path.git

csi-driver-host-path/deploy/kubernetes-latest/deploy.sh
```

動作確認
```
kubectl get pod -A
```

##### ②. StorageClass の定義

マニフェストの作成
```
cat << EOF > csi-hostpath-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-hostpath-sc
provisioner: hostpath.csi.k8s.io
EOF
```

マニフェストの適用と確認
```
kubectl apply -f csi-hostpath-sc.yaml

kubectl get storageclasses
```

##### ③. PVC の定義

マニフェストの作成
```
cat << EOF > pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-hostpath-sc
EOF
```

マニフェストの適用と確認
```
kubectl apply -f pvc.yaml

kubectl get pvc

kubectl get pv
```

##### ④. Pod にPVCをVolumeマウント

マニフェストの作成

```
cat << EOF > csi-pod.yaml
kind: Pod
apiVersion: v1
metadata:
  name: csi-pod
spec:
  containers:
    - name: my-frontend
      image: busybox
      volumeMounts:
      - mountPath: "/data"
        name: my-csi-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: my-csi-volume
      persistentVolumeClaim:
        claimName: csi-pvc
EOF
```

マニフェストの適用と確認

```
kubectl apply -f csi-pod.yaml

kubectl get pod
```

##### PVの挙動を確認
Pod内に入り、ファイルシステムを確認
```
kubectl exec -it csi-pod -- sh

df -h

ls /data
```

/dataと/tmp以下に、テキストデータを設置
```
echo "This is persistent data!" > /data/pdata.txt

echo "This is temporary" > /tmp/tmpdata.txt

cat /data/pdata.txt

cat /tmp/tmpdata.txt

exit
```

Podの削除と再作成
```
kubectl delete -f csi-pod.yaml

kubectl apply -f csi-pod.yaml
```

再びPodに入り、先程作成したデータを確認。
```
kubectl exec -it csi-pod -- sh

ls /tmp

ls /data

cat /data/pdata.txt

exit
```

K8sオブジェクトの削除
```
kubectl delete pod csi-pod

kubectl delete pvc csi-pvc

kubectl get pv
```

#### 2. StatefulSetの利用

##### ① Headless Serviceの作成

オブジェクトの作成と確認
```
kubectl create service clusterip sts-app --clusterip="None"

kubectl get svc
```

##### ② StatefulSetの作成

StatefulSetマニフェストの用意
```
cat << EOF > web-sts.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-sts
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sts-app
  serviceName: sts-app
  template:
    metadata:
      labels:
        app: sts-app
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "csi-hostpath-sc"
      resources:
        requests:
          storage: 1Gi
EOF
```

(別端末) podの監視
```
kubectl get pod -w
```

マニフェストの適用
```
kubectl apply -f web-sts.yaml
```

オブジェクトの確認
```
kubectl get pod,sts
```

PVとPVCの確認
```
kubectl get pv

kubectl get pvc
```

##### ③ StatefulSetへのアクセス
各PodのPVにデータを書き込み
```
TOPPAGE=/usr/share/nginx/html/index.html

kubectl exec web-sts-0 -- sh -c 'echo $(hostname) > '$TOPPAGE

kubectl exec web-sts-1 -- sh -c 'echo $(hostname) > '$TOPPAGE

kubectl exec web-sts-2 -- sh -c 'echo $(hostname) > '$TOPPAGE
```

調査用Podを起動
```
kubectl run -i -t --rm --image=brix4dayz/swiss-army-knife -- sh
```

Headless Serviceによる名前解決
```
nslookup web-svc

nslookup web-sts-0.web-svc
```

Service名を指定して、curlでWebページにアクセス（※. 何度か実行）
```
curl web-svc
```

Pod名を指定して、curlでWebページにアクセス
```
curl web-sts-0.sts-app

curl web-sts-1.sts-app
```

調査用端末から抜ける
```
exit
```

##### ④ StatefulSetのローリングアップデート

(別端末) podの監視
```
kubectl get pod -w
```

StatefulSetのロールアウト
```
kubectl rollout restart statefulsets/web-sts
```

ロールアウト後もPVがコンテンツを保持していることを確認
```
kubectl exec web-sts-0 -- sh -c 'cat /usr/share/nginx/html/index.html'
```

オブジェクトのお掃除
```
kubectl delete -f web-sts.yaml

kubectl delete service sts-app

kubectl delete pvc www-web-sts-0 www-web-sts-1 www-web-sts-2
```

#### 3. Ephemeral Volumeの利用

##### ①	empdyDirの作成

マニフェストの用意
```
cat << EOF > pod-ev.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ev
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: vol-ev
      mountPath: /data
  volumes:
  - name: vol-ev
    emptyDir: {}
EOF
```

マニフェストの適用
```
kubectl apply -f pod-ev.yaml
```

Pod状態の監視
```
kubectl get pod pod-ev -w
```

##### ②	コンテナ障害時のemptyDirの挙動

(別端末) Podの中に入る
```
kubectl exec -it pod-ev -- /bin/bash
```

(別端末) テストデータを配置した後、コンテナ障害を発生させる
```
echo ephemeral > /data/ev.txt

echo tmpdata > /tmp/tmp.txt

cat /data/ev.txt

cat /tmp/tmp.txt

kill 1
```

コンテナ障害後のデータを確認
```
kubectl exec -it pod-ev -- cat /tmp/tmp.txt

kubectl exec -it pod-ev -- cat /data/ev.txt
```

##### ③	Pod再作成時のemptyDirの挙動

Podを削除して再作成
```
kubectl delete pod pod-ev

kubectl apply -f pod-ev.yaml
```

/data以下のデータを確認
```
kubectl exec -it pod-ev -- cat /data/ev.txt
```

作成したPodのお掃除
```
kubectl delete pod pod-ev
```