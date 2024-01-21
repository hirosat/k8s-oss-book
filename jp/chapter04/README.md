# 4章 - コンテナを作ろう

## 訂正・フィードバック

現在ありません

## リンク一覧

- [DockerHub](https://hub.docker.com/)
  - [アカウント登録](https://hub.docker.com/signup)
  - [コンテナイメージ検索](https://hub.docker.com/search?q=)
  - [Sign In](https://login.docker.com/u/login/)
- [Dockerfileリファレンス](https://docs.docker.jp/engine/reference/builder.html)
- [Cloud Native Buildpacks](https://buildpacks.io/)
  - [(参考): Build](https://buildpacks.io/docs/concepts/operations/build/)
  - [(参考): Rebase](https://buildpacks.io/docs/concepts/operations/rebase/)
- [pack CLIダウンロード](https://github.com/buildpacks/pack/releases)
- Paketo Buildpacks
  - [builder-jammy-full](https://github.com/paketo-buildpacks/builder-jammy-full)
  - [builder-jammy-base](https://github.com/paketo-buildpacks/builder-jammy-base)
  - [builder-jammy-tiny](https://github.com/paketo-buildpacks/builder-jammy-tiny)

## コード集

### [定番] Dockerfile によるコンテナのビルド

#### 1. Dockerfileの作成とコンテナのビルド

シンプルなDockerfile作成例
```
cd ~/git/k8s-oss-sample-app/
vi Dockerfile
```

(ファイル内容)
```
FROM node:18.9.0
CMD ["sleep", "600"]
```

コンテナのビルド
```
docker build -t test1 .
```

コンテナイメージのリスト表示
```
docker images
```

#### 2. 作成したイメージの動作確認

コンテナの起動
```
docker run -d test1
```

実行中のコンテナ一覧
```
docker ps
```

対象コンテナに対して、任意のコマンドを実行
```
docker exec (CONTAINER ID) node -v
```

コンテナの停止
```
docker stop (CONTAINER ID)
```

#### 3. ソースコードを取り込んだビルド

実際のプログラムを組み込むDockerfileの作成例
```
cd ~/git/k8s-oss-sample-app/
vi Dockerfile
```

(ファイル内容)
```
FROM node:18.9.0
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```

タグを付与したコンテナのビルド (※. タグを付与しなくても動作に支障はありません)
```
docker build -t node-app:v1.0.0 .
```

イメージ一覧
```
docker images
```

proxyオプションを付与したコンテナの起動
```
docker run -d -p 3000:3000 node-app:v1.0.0
```

#### 4. レジストリに格納

DockerHubにpushする際は、事前にdocker loginが必要。<br>
 (※. UsernameとPasswordには、DockerHubのアカウントを使用する)
```
docker login
```

（docker pushの失敗例: 権限エラーとなる)
```
docker push node-app:v1.0.0
```

namespace付きのタグを生成
```
docker tag node-app:v1.0.0 (Docker ID)/node-app:v1.0.0
```

namespace付きのタグを持つイメージを確認
```
docker images
```

namespace付きのタグを使ってpush (成功例)
```
docker push (Docker ID)/node-app:v1.0.0
```

#### Dockerfileのベストプラクティス

##### イメージレイヤーの最適化

package*.json を事前にコピーする様に設定
```
FROM node:18.9.0
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
```

.dockerignore ファイルを用意
```
vi .dockerignore
```

(ファイル内容)
```
node_modules
```

2回目以降のビルドで差が出る
```
docker build -t node-app:v1.0.1 .
```

##### マルチステージ・ビルド

ビルド用コンテナと動作用コンテナを別々に用意する
```
# 1st stage
FROM node:18.9.0 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 2nd stage
FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

ビルド
```
docker build -t node-app:v1.0.2 .
```

イメージ一覧(小さいイメージになったことを確認)
```
docker images
```

コンテナ起動
```
docker run -d -p 8080:80 node-app:v1.0.2
```

##### RUN命令の結合

(非推奨パターン) Dockerfileの記載例
```
RUN apt-get update
RUN apt-get install curl
```
(推奨パターン) Dockerfileの記載例
```
RUN apt-get update && apt-get install curl
```

##### コンテナイメージへのアノテート

LABEL命令の付与したDockerfileの記載例
```
LABEL org.opencontainers.image.authors="Jane Smith <jsmith@example.com>"
```

docker image inspectで確認
```
docker image inspect nginx --format="{{json .Config.Labels}}"
```

##### ARG/ENV変数の活用

ARG変数とENV変数を使用したDockerfileの例
```
ENV DEFAULT_USER=admin
ARG NGINX_URL=”http://nginx.org/download”
ARG NGINX_VERSION=1.18.0
RUN curl -SL ${NGINX_URL}/nginx-${NGINX_VERSION}.tar.gz | tar -zxC && ...
```

##### 非rootユーザ化

Dockerfileの設定例
```
FROM ubuntu:latest
RUN apt-get update && apt-get -y install gosu
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

entrypoint.shの内容例
```
#!/bin/bash
useradd -u 1001 -o -m developer
groupmod -g 1001 developer
export HOME=/home/developer
exec gosu developer "$@"
```

### [オススメ] Cloud Native Buildpacks によるコンテナのビルド

#### OSSのインストール先について

(Windowsユーザ) ~/binディレクトリの作成
```
mkdir ~/bin
```

PATH環境変数の表示
```
echo $PATH
```

#### pack CLIのインストール

(Windowsユーザ) pack CLIのインストール方法
```
VER=v0.32.1        # 左記は本書で検証したバージョンです。最新版を取得しましょう。

FILE=pack-$VER-windows.zip

curl -LO https://github.com/buildpacks/pack/releases/download/$VER/$FILE

unzip $FILE

mv ./pack* ~/bin
```

(Macユーザ) pack CLIのインストール方法
```
VER=v0.32.1        # 左記は本書で検証したバージョンです。最新版を取得しましょう。

FILE=pack-$VER-macos-arm64.tgz

curl -LO https://github.com/buildpacks/pack/releases/download/$VER/$FILE

tar zxvf $FILE

mv pack/pack /usr/local/bin
```

pack CLIのバージョン確認
```
pack --version
```

#### packでビルド

2章で作成したNode.jsアプリのディレクトリに移動し、バグ対策の設定を追加 (※.[バグ内容](https://github.com/paketo-buildpacks/builder-jammy-base/issues/375))
```
cd ~/git/k8s-oss-sample-app/

echo ”DISABLE_ESLINT_PLUGIN=true” > .env
```

packによるコンテナビルドを実行
```
pack build cnb-app --builder paketobuildpacks/builder-jammy-base
```

ビルド後のイメージ確認 (※.参考: [CNBのReproducibility](https://buildpacks.io/docs/features/reproducibility/))
```
docker images
```

コンテナの起動
```
docker run -d -p 3000:3000 --platform linux/x86_64 cnb-app:latest
```

#### packでアップデート

アプリのトップページの編集
```
cd ~/git/k8s-oss-sample-app/

vi src/App.js
```

(変更内容: `Edit <code>〜`の行を、`Hello CNB App!`にする)
```
       <header className="App-header">
         <img src={logo} className="App-logo" alt="logo" />
         <p>
           Hello CNB App!
         </p>
```

変更内容の確認
```
git diff
```

再度、pack build
```
pack build cnb-app --builder paketobuildpacks/builder-jammy-base
```

コンテナイメージの確認
```
docker images
```

コンテナを停止して起動
```
docker ps

docker stop (Container ID)

docker run -d -p 3000:3000 --platform linux/x86_64 cnb-app:latest
```

#### builderの選定

推奨builderの表示
```
pack builder suggest
```

#### packのrebase

rebase方法
```
pack rebase my-app:my-tag
```