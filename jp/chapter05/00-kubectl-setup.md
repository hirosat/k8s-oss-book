## （共通） kubectl CLIのセットアップ

#### Windowsの場合

以下の方法で、最新版の **kubectl CLI** をダウンロードします。
```
VER=`curl.exe -L -s https://dl.k8s.io/release/stable.txt`
curl.exe -LO "https://dl.k8s.io/release/$(VER)/bin/windows/amd64/kubectl.exe"
```
その後は、**PATH環境変数**で定義されたディレクトリ（例: ~/bin/以下）に、解凍した**exe**ファイルを設置することで、**Git Bash** からCLIとして利用できるようになります。

autocomplition(自動補完)の設定
```
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

---

#### Macの場合

```
### kubectl バイナリのダウンロード
VER=`curl -L -s https://dl.k8s.io/release/stable.txt`
curl -LO "https://dl.k8s.io/release/$(VER)/bin/darwin/arm64/kubectl"

### kubectlのインストール
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

autocomplition(自動補完)の設定
```
vi ~/.zshrc
```
編集内容
```
(※. 最初の行に以下の２行を挿入)
Autoload -Uz compinit
Compinit

(※. 途中省略。最後の行に以下を挿入)
source <(kubectl completion zsh) 
```

---

#### Win/Mac共通

インストール後の動作確認
```
$ kubectl version --client
```

ショートカット(k)の設定
```
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
```
