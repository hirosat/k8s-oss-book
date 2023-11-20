# 3章 - コンテナ環境のセットアップ

## 訂正・フィードバック

現在ありません

## リンク一覧

- [Docker Desktop 公式ページ](https://www.docker.com/products/docker-desktop/)
- [Rancher Desktop 公式ページ](https://rancherdesktop.io/)

## コード集

### dockerの状態を確認 (docker info)
```
docker info
```

### コンテナの作成
動作中のコンテナ一覧
```
docker ps
```

nginxコンテナを作成
```
docker run nginx
```

コンテナ一覧の再確認
```
docker ps
```

nginxコンテナを作成 (起動オプションを付与)
```
$ docker run -p 80:80 -d --name nginx-test nginx:latest
```

コンテナ一覧の再確認
```
docker ps
```
