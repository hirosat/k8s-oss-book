# （パターン2） クラウドサービスの利用

## リンク一覧

- [Google Cloud](https://console.cloud.google.com/) : gmailアカウントがあれば、利用可能
- [Google Kubernetes Engineドキュメント](https://cloud.google.com/kubernetes-engine/docs?hl=ja)
  - [クイックスタート: アプリを GKE クラスタにデプロイする](https://cloud.google.com/kubernetes-engine/docs/deploy-app-cluster)
- [Google Cloud CLI のドキュメント](https://cloud.google.com/sdk/docs?hl=ja)
  - [gcloud CLI をインストールする](https://cloud.google.com/sdk/docs/install?hl=ja)

## コード集

### gcloud CLIのセットアップ

(前提条件) Pythonが使用可能なことを確認
```
$ python3 -V
```

Google SDK のダウンロード 
```
wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-408.0.1-darwin-x86_64.tar.gz

tar zxvf google-cloud-cli-408.0.1-darwin-x86_64.tar.gz
```

gcloud CLIのインストール
```
./google-cloud-sdk/install.sh
```

gcloud CLIの動作確認
```
gcloud version
```

初期設定
```
gcloud init
```

### GKEクラスタへの接続

画面に表示された「コマンドラインアクセス」を実行
```
gcloud container clusters get-credentials cluster-1 --zone us-central1-c --project (あなたのGKEプロジェクト)
```

登録されたコンテキストの確認
```
kubectl config current-context
```

GKEノード状態の確認
```
kubectl get node
```