# 8. Production Readyなベストプラクティス (Real World Docker App)

## 1. サンプルアプリのVoting Appはすでにクローン済み (Clone the repo)
```
git clone https://github.com/dockersamples/example-voting-app.git
```

## 2. [example-voting-app/docker-compose.yml](example-voting-app/docker-compose.yml)をチェック
```
version: "3"

services:
  # 5つのコンテナが定義されている (vote, result, worker, redis, db)
  vote:
    # voteのDockerfileフォルダーを指定
    build: ./vote
    # voteのContainerが起動した時のコマンドを指定
    command: python app.py
    volumes:
     - ./vote:/app
    # VoteのContainerのホスト：コンテナ間のポートマッピング
    ports:
      - "5000:80"
    ＃コンテナが存在するネットワークを指定
    networks:
      - front-tier
      - back-tier

  result:
    build: ./result
    command: nodemon server.js
    volumes:
      - ./result:/app
    ports:
      - "5001:80"
      - "5858:5858"
    networks:
      - front-tier
      - back-tier

  worker:
    build:
      context: ./worker
    depends_on:
      - "redis"
      - "db"
    networks:
      - back-tier

  redis:
    image: redis:alpine
    container_name: redis
    ports: ["6379"]
    networks:
      - back-tier

  db:
    image: postgres:9.4
    container_name: db
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"
    volumes:
      - "db-data:/var/lib/postgresql/data"
    networks:
      - back-tier

volumes:
  db-data:

＃2つのネットワークを作成
networks:
  front-tier:
  back-tier:
```

## 3. voting appをdocker-composeで起動
```
docker-compose -f example-voting-app/docker-compose.yml up -d
```

2つのDocker network `example-voting-app_front-tier` と `example-voting-app_back-tier`が作られたのがわかる
```
WARNING: The Docker Engine you're using is running in swarm mode.

Compose does not use swarm mode to deploy services to multiple nodes in a swarm. All containers will be scheduled on the current node.

To deploy your application across the swarm, use `docker stack deploy`.

Creating network "example-voting-app_front-tier" with the default driver
Creating network "example-voting-app_back-tier" with the default driver
Creating volume "example-voting-app_db-data" with default driver

```

その後コンテナイメージの作成されている
```
Building vote
Step 1/7 : FROM python:2.7-alpine
2.7-alpine: Pulling from library/python
aad63a933944: Pull complete
259d822268fb: Pull complete
10ba96d218d3: Pull complete
44ba9f6a4209: Pull complete
Digest: sha256:724d0540eb56ffaa6dd770aa13c3bc7dfc829dec561d87cb36b2f5b9ff8a760a
Status: Downloaded newer image for python:2.7-alpine
 ---> 8579e446340f
Step 2/7 : WORKDIR /app

Successfully built f76c34f84f91
Successfully tagged example-voting-app_vote:latest
```

```
Building result
Step 1/9 : FROM node:10-slim
 ---> 9cc5a0c00120
Step 2/9 : WORKDIR /app

Successfully built 16de1e51be42
Successfully tagged example-voting-app_result:latest
```

全てのイメージが作成されたあと、コンテナ起動
```
Creating redis                       ... done
Creating example-voting-app_result_1 ... done
Creating example-voting-app_vote_1   ... done
Creating db                          ... done
Creating example-voting-app_worker_1 ... done
Attaching to db, redis, example-voting-app_vote_1, example-voting-app_result_1, example-voting-app_worker_1
```

起動中のコンテナをチェック
```
$ docker ps

CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                                          NAMES
04d6abf7954a        example-voting-app_worker   "/bin/sh -c 'dotnet …"   56 seconds ago      Up 53 seconds                                                      example-voting-app_worker_1
8833d0bacc62        postgres:9.4                "docker-entrypoint.s…"   58 seconds ago      Up 56 seconds       5432/tcp                                       db
05c5a894a332        example-voting-app_vote     "python app.py"          58 seconds ago      Up 55 seconds       0.0.0.0:5000->80/tcp                           example-voting-app_vote_1
11c0c854e4d6        example-voting-app_result   "docker-entrypoint.s…"   58 seconds ago      Up 54 seconds       0.0.0.0:5858->5858/tcp, 0.0.0.0:5001->80/tcp   example-voting-app_result_1
89c80411ca12        redis:alpine                "docker-entrypoint.s…"   58 seconds ago      Up 55 seconds       0.0.0.0:32768->6379/tcp                        redis
```

`example-voting-app_vote_1`が`0.0.0.0:5000`でリスニング中なのがわかる
```
05c5a894a332        example-voting-app_vote     "python app.py"          58 seconds ago      Up 55 seconds       0.0.0.0:5000->80/tcp                           example-voting-app_vote_1
```

ブラウザーで`http://localhost:5000/`をチェック


`example-voting-app_result_1`が`0.0.0.0:5001`でリスニング中なのがわかる
```
11c0c854e4d6        example-voting-app_result   "docker-entrypoint.s…"   58 seconds ago      Up 54 seconds       0.0.0.0:5858->5858/tcp, 0.0.0.0:5001->80/tcp   example-voting-app_result_1
```

ブラウザーで`http://localhost:5001/`をチェック


## 4.　[example-voting-app/result/Dockerfile](example-voting-app/result/Dockerfile)をチェック
```
FROM node:10-slim

# コンテナが起動した時のWORKDIRを指定
WORKDIR /app

RUN npm install -g nodemon

# package.jsonのみ最初にコピーし、パッケージがインストールされたキャッシュレイヤーを作る
COPY package*.json ./

RUN npm ci 
RUN npm cache clean --force 
RUN mv /app/node_modules /node_modules

# 最後に全てのファイルをにコピー
COPY . .

# 環境変数を設定
ENV PORT 80

EXPOSE 80

# コンテナが起動した時のコマンドを指定
CMD ["node", "server.js"]
```


## 5.1 本番ベストプラクティス: rootユーザーでコンテナを起動するべからず
```
cd example-voting-app/result
```

`result`コンテナを`node`ユーザーで起動する

[example-voting-app/result/Dockerfile.nonroot_user.v2](example-voting-app/result/Dockerfile.nonroot_user.v2)
```
# /appのパミッションとOwnerを設定  <---- changed
RUN chown -R node:node /app && \
    chmod -R 744 /app && \
    chown -R node:node /node_modules && \
    chmod -R 777 /node_modules

# コンテナを起動した時のプロセスのOwnerを指定  <---- changed
USER node
```

v2イメージの作成
```
docker build -t result_nonroot_user_v2 --file Dockerfile.nonroot_user.v2 .
```

コンテナをバックグラウンドで起動
```
docker run -d --name result \
    -p 5002:80 \
    -p 5859:5858 \
    --rm -it result_nonroot_user_v2 sleep 500
```

シェルでコンテナ内に入り、OwnershipとPermissionをチェック。
```
docker exec -it result sh

$ ls -al
total 80
drwxr--r-- 1 node node  4096 May 23 12:45 .
drwxr-xr-x 1 root   root    4096 May 23 13:00 ..
-rwxr--r-- 1 node node    14 May 23 10:38 .dockerignore
drwxr--r-- 1 node node  4096 May 23 10:38 .vscode
-rwxr--r-- 1 node node   570 May 23 12:20 Dockerfile
-rwxr--r-- 1 node node   813 May 23 12:45 Dockerfile.nonroot_user.v2
-rwxr--r-- 1 node node   872 May 23 10:38 docker-compose.test.yml
drwxr--r-- 1 node node  4096 May 23 10:38 dotnet
-rwxr--r-- 1 node node 32705 May 23 10:38 package-lock.json
-rwxr--r-- 1 node node   445 May 23 10:38 package.json
-rwxr--r-- 1 node node  2329 May 23 10:38 server.js
drwxr--r-- 1 node node  4096 May 23 10:38 tests
drwxr--r-- 1 node rnode  4096 May 23 10:38 views
```

コンテナをフォアグランドで起動して作動確認
```
docker stop result

docker run --name result \
    -p 5002:80 \
    -p 5859:5858 \
    --rm -it result_nonroot_user_v2
```

Permissionエラーがログで見える
```
Emitted 'error' event on Server instance at:
    at emitErrorNT (net.js:1340:8)
    at processTicksAndRejections (internal/process/task_queues.js:84:21) {
  code: 'EACCES',
  errno: 'EACCES',
  syscall: 'listen',
  address: '0.0.0.0',
  port: 80
```

## 5.2 本番ベストプラクティス: non-rootユーザーは,ポート1024以上をリスニングするように設定

non-rootユーザーは、ポート1024以下のポートをオープンできないため、PORTを変更。

[example-voting-app/result/Dockerfile.port_above_1024.v3](example-voting-app/result/Dockerfile.port_above_1024.v3)
```
# 環境変数を設定   <---- changed
# Non-privileged user (not root) can't open a listening socket on ports below 1024
# ref: https://stackoverflow.com/questions/9164915/node-js-eacces-error-when-listening-on-most-ports
ENV PORT 8080

#   <---- changed
EXPOSE 8080  
```


v３イメージの作成
```
docker build -t result_port8080_v3 --file Dockerfile.port_above_1024.v3 .
```

コンテナをフォアグランドで起動して作動確認。エラーは消えたのが確認できる 
```
$ docker run --name result     -p 5002:80     -p 5859:5858     --rm -it result_port8080_v3

body-parser deprecated bodyParser: use individual json/urlencoded middlewares server.js:73:9
body-parser deprecated undefined extended: provide extended option ../node_modules/body-parser/index.js:105:29
App running on port 8080
Waiting for db
Waiting for db
```

しかし、イメージサイズの確認するとv１より大きくなっているのがわかる
```
$ docker images

result_port8080_v3                                                       latest              0055b1712cef        3 seconds ago       159MB
result_nonroot_user_v2                                                   latest              f0e343921530        3 seconds ago        159MB                                                 latest              f0e343921530  
example-voting-app_result                                                latest              16de1e51be42        2 hours ago          146MB
```  

イメージのレイヤー履歴を表示
```
$ docker history result_port8080_v3

IMAGE               CREATED              CREATED BY                                      SIZE                COMMENT
5698597b5fed        11 seconds ago       /bin/sh -c #(nop)  CMD ["node" "server.js"]     0B                  
228653381947        11 seconds ago       /bin/sh -c #(nop)  USER node                    0B                  
4381bf365877        11 seconds ago       /bin/sh -c chown -R node:node /app &&     ch…   5.84MB              
d4757a974d1a        15 seconds ago       /bin/sh -c #(nop) COPY dir:a6f3e2382e682898b…   473kB               
916ccf027932        About a minute ago   /bin/sh -c #(nop)  EXPOSE 8080                  0B                  
ca5a30d55dc4        About a minute ago   /bin/sh -c #(nop)  ENV PORT=8080                0B                  
d1ef68da43df        7 minutes ago        /bin/sh -c npm ci  && npm cache clean --forc…   5.36MB              
5e7447cbd1f1        7 minutes ago        /bin/sh -c #(nop) COPY multi:36e78ca4b899038…   33.1kB              
e9d5b97cd5ef        23 minutes ago       /bin/sh -c npm install -g nodemon               4.72MB   
```


`RUN chown -R node:node /app...`が`5.84MB`使っているのがわかる
```
4381bf365877        11 seconds ago       /bin/sh -c chown -R node:node /app &&     ch…   5.84MB 
```


## 5.3 本番ベストプラクティス: RUN chown && chmodはレイヤーサイズを倍増させるので使うべからず

代わりに`COPY --chown node:node`を使う

[example-voting-app/result/Dockerfile.copy_chown.v4](example-voting-app/result/Dockerfile.copy_chown.v4)
```
# 最後に全てのファイルをにコピー & /appのパミッションとOwnerを設定
# chown & chmod はレイヤーサイズを倍にするので使うべからず！代わりにCOPY --chownを使う    <---- changed
COPY --chown=node:node . .
```

v４イメージの作成
```
docker build -t result_copy_chown_v4 --file Dockerfile.copy_chown.v4 .
```

イメージサイズの確認すると`v3`より`v4`が`6MB`小さくなっているのがわかる
```
$ docker images

result_copy_chown_v4                                                     latest              2f66932a2338        About a minute ago   153MB
result_port8080_v3                                                       latest              0055b1712cef        3 seconds ago       159MB
result_nonroot_user_v2                                                   latest              f0e343921530        3 seconds ago        159MB
example-voting-app_result                                                latest              16de1e51be42        2 hours ago          146MB                                                 latest              f0e343921530  
```  

イメージのレイヤー履歴を表示
```
$ docker history result_copy_chown_v4

IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
28583ada8349        20 seconds ago      /bin/sh -c #(nop)  CMD ["node" "server.js"]     0B                  
2e716b3bb7f1        20 seconds ago      /bin/sh -c #(nop)  USER node                    0B                  
a8b20391f6c1        20 seconds ago      /bin/sh -c #(nop) COPY --chown=node:nodedir:…   474kB               
3ec847097acb        21 seconds ago      /bin/sh -c #(nop)  EXPOSE 8080                  0B                  
ab558d45b07b        21 seconds ago      /bin/sh -c #(nop)  ENV PORT=8080                0B                  
d641989f9617        22 seconds ago      /bin/sh -c npm ci  && npm cache clean --forc…   5.36MB              
1b6161492c54        31 seconds ago      /bin/sh -c #(nop) COPY --chown=node:nodemult…   33.1kB              
e9d5b97cd5ef        31 minutes ago      /bin/sh -c npm install -g nodemon               4.72MB  
```

v4の`COPY --chown=node:node`が`474kB`のみ使用していて、明らかにサイズ縮小の理由であることがわかる
```
/bin/sh -c #(nop) COPY --chown=node:nodedir:…   474kB
```

v3の`chown -R node:node /app`が`5.84MB`使用
```
/bin/sh -c chown -R node:node /app &&     ch…   5.84MB
```


## 5.4 本番ベストプラクティス: 複数のRUNコマンド”RUN CMD1 RUN CMD2”よりも”RUN &&"で数珠つなぎにすべし

Add below to [example-voting-app/result/Dockerfile.chain_run_command.v5](example-voting-app/result/Dockerfile.chain_run_command.v5)

v4
```
RUN npm ci 
RUN npm cache clean --force 
RUN mv /app/node_modules /node_modules
```

v5
```
# RUN && でレイヤー数を圧縮  <---- changed
RUN npm ci \
 && npm cache clean --force \
 && mv /app/node_modules /node_modules
```

v5イメージの作成
```
docker build -t result_chain_run_command_v5 --file Dockerfile.chain_run_command.v5 .
```


イメージサイズの確認すると`v4`より`v5`が`7MB`小さくなっているのがわかる
```
result_chain_run_command_v5                                              latest              a2239748ddd7        10 seconds ago       146MB
result_copy_chown_v4                                                     latest              2f66932a2338        About a minute ago   153MB
result_port8080_v3                                                       latest              0055b1712cef        3 seconds ago       159MB
result_nonroot_user_v2                                                   latest              f0e343921530        3 seconds ago        159MB
example-voting-app_result                                                latest              16de1e51be42        2 hours ago          146MB
```


## 5.5 本番ベストプラクティス: CMDコマンドはBackgroundで起動するべからず
ドッカーのRootプロセスは、バックグランドで起動されるとコンテナは停止してしまいます。

- ubuntuイメージからコンテナ起動
```
docker run -d --name test1 ubuntu 

＃コンテナが停止している
docker ps -a

＃イメージをインスペクトすると開始時のCMDが”CMD ["/bin/bash"]”なのがわかr
docker inspect ubuntu
```

- ubuntuイメージからコンテナ起動し、sleep 500 コマンドを/bin/bashのArgumentとしてPassする
```
docker run -d --name test2 ubuntu /bin/sh -c "sleep 500"
```

- ubuntuイメージからコンテナ起動し、sleep 500 コマンドをバックグランドで起動
```
docker run -d --name test3 ubuntu /bin/sh -c "sleep 500 &"

＃コンテナが停止している
docker ps -a
```

## Cleanup
```
docker rm test{1..3} 
docker system prune
docker ps -a
```
