# 2章 - GitOps を始めよう

## 訂正・フィードバック

現在ありません

## リンク一覧

- [GitHub](https://github.com/)
- [Git for windows](https://gitforwindows.org/)
- [Guide To GitOps](https://www.weave.works/technologies/gitops/)

## コード集

### gitの初期設定
```
$ git config --global user.name (GitHubアカウント名)
$ git config --global user.email (GitHubに登録したemailアドレス)
```
(動作確認)
```
$ cat ~/.gitconfig
```

### gitリポジトリをローカルclone
作業用ディレクトリを作成して、移動
```
$ mkdir ~/git
$ cd ~/git
```
gitのclone
```
$ git clone (リモート上のgitリポジトリのパス)
$ ls
$ cd k8s-oss-sample-app/
$ ls -al
```