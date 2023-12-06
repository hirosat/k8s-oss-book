# （共通） kubectl CLIのセットアップ

### Windowsの場合

以下の方法で、最新版の **kubectl CLI** をダウンロードします。

```
VER=`curl.exe -L -s https://dl.k8s.io/release/stable.txt`
curl.exe -LO "https://dl.k8s.io/release/$(VER)/bin/windows/amd64/kubectl.exe"
```

その後は、**PATH環境変数**で定義されたディレクトリ（例: ~/bin/以下）に、解凍した**exe**ファイルを設置することで、**Git Bash** からCLIとして利用できるようになります。


### Macの場合

```
### kubectl バイナリのダウンロード
$ VER=`curl -L -s https://dl.k8s.io/release/stable.txt`
$ curl -LO "https://dl.k8s.io/release/$(VER)/bin/darwin/amd64/kubectl"

### kubectlのインストール
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

### (共通) インストール後の動作確認

```
$ kubectl version --client
```