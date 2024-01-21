# 2章 - GitOps を始めよう

## 訂正・フィードバック

現在ありません

## リンク一覧

- [GitHub](https://github.com/)
  - [GitHubアカウントへの新しいSSHキーの追加](https://github.com/hirosat/k8s-oss-book)
- [Git for windows](https://gitforwindows.org/)
- [Guide To GitOps](https://www.weave.works/technologies/gitops/)
- [Node.js 公式サイト](https://nodejs.org/en/)
- [Homebrew](https://brew.sh/)
- [Create React App](https://create-react-app.dev/)
- (参考) [サンプルアプリケーション](https://github.com/hirosat/k8s-oss-sample-app)

## コード集

### gitの初期設定

```
git config --global user.name (GitHubアカウント名)

git config --global user.email (GitHubに登録したemailアドレス)
```
(動作確認)
```
cat ~/.gitconfig
```

### SSHキーを作成してGitHubに登録

SSHキーペアの有無を確認
```
ls -a ~/.ssh
```

SSHキーペアの作成 (この後聞かれる質問には、全てEnterを押す)
```
ssh-keygen -t ed25519

ls -a ~/.ssh
```

GitHubへの接続テスト
```
ssh -T git@github.com
```


### gitリポジトリをローカルclone

作業用ディレクトリを作成して、移動
```
mkdir ~/git

cd ~/git
```

[本書と連動したgitリポジトリ](https://github.com/hirosat/k8s-oss-book) のclone
```
git clone git@github.com:hirosat/k8s-oss-book.git
```

(上記実行後の、ファイル構成の確認)
```
ls

ls k8s-oss-book
```

今回あなたが作成したgitリポジトリのclone
```
git clone (リモート上のgitリポジトリのパス)
```

(上記実行後の、ファイル構成の確認)
```
ls

ls k8s-oss-sample-app
```

### gitリポジトリの更新

#### 1. ソースコードの変更

- vi操作方法の参考: [ラスクエンジニアブログ](https://tech-blog.rakus.co.jp/entry/20210715/vi)

```
cd ~/git/k8s-oss-sample-app/

vi README.md
```

(編集内容のサンプル)
```
# k8s-oss-sample-app
this is sample app.
```

変更後の確認コマンド
```
cat README.md

git status

git diff
```

#### 2. git add
```
git add .

git status
```

#### 3. git commit
```
git commit -m "(任意のコミットメッセージ)"

git status
```

#### 4. git push

```
git push
```

### Node.jsのインストール

#### Windowsの場合
インストーラの指示に従う

#### Macの場合

Homebrewのインストール
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Node.jsのインストール
```
brew install node
```

#### インストール後の確認

```
node -v

npm -v
```

### 最初のアプリケーションの作成

Reactプロジェクトの作成
```
cd ~/git/k8s-oss-sample-app/

npx create-react-app .
```

Reactアプリケーションの起動
```
npm start
```

### アプリケーションをgitリポジトリに登録

```
git add .

git commit -m "1st application"

git push
```