# Docker-learn
在YT上找到一篇厲害的[Docker教學](https://www.youtube.com/watch?v=3c-iBn73dDE&list=RDCMUCdngmbVKX1Tgre699-XLlUA&start_radio=1&t=4832s&ab_channel=TechWorldwithNana)講解其基礎並實作一應用範例，教學範圍和整體的Workflow如以下所示:

1. Docker指令集：以`MongoDB`和`Mongo-express`為例，進行容器設定、網域連接等
2. 將寫好的JavaScript App寫成`Dockerfile`包成一個image
3. 將JavaScript App、MongoDB和Mongo-Express用三個容器包裝起來，再以`Docker-compose.yaml`設定微服務架構
4. 把image上傳至private repo（在此以AWS ECR為例），使用者可以到該repo把image給pull下來做開發環境部署

註：Docker是一個容器的服務，主要處理的是Application layer，而VM主要是處理OS layer和Application layer，所以VM通常耗費的記憶體容量較大，而由於Docker容器是利用被部署機器的OS，因此如果有調用到OS相關套件的話二者須相互對應。

<div align=center><img src="https://user-images.githubusercontent.com/100120881/175194419-3d201360-b5fc-4b37-b891-578f5f68bfe4.png"></div>

## 以Docker建立MongoDB+Mongo-Express

可先利用`docker images`先列出本機上所有image做檢查，接著直接在terminal中執行以下指令：

```
docker run mongodb
docker run mongo-expres
```

若想要建立特定網域（Network）讓所有容器服務都跑在同一個網路環境下，以命名`mongo-network`作為範例：

```
docker network create mongo-network
```

假設目前沒有`mongodb`和`mongo-expres`這二個image，就會自動到Docker hub上pull下來。我們也可以針對跑起來的服務做一些設定，以`mongodb`為例：

```
docker run -p 27017:27017 -d -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_PASSWORD=password --name mongodb --net mongo-network
```

其中，參數解釋如下：
* `-d`：Detach mode
* `-p`：Ports，用來設定容器對外和對內的通訊連接埠，格式為`HOST_PORT:CONTAINER_PORT`，前者代表Host上要取用容器內服務需要連接的通道、後者代表容器內部服務互相溝通需要連接的通道
* `-e`：Environment variable，代表該服務可設定的環境變數，`mongodb`就有`MONGO_INITDB_ROOT_USERNAME`、`MONGO_INITDB_PASSWORD`...等
* `--name`：容器識別名，後續如果在容器內的服務需要互相溝通，可以直接用識別名而不需要用連接埠
* `--net`：網域名，用於設定那些容器處在同一個網域下可相互通訊

同樣的，`mongo-express`同樣也有一些設定：

```
docker run -d -p 8081:8081 -e ME_CONFIG_MONGODB_ADMINUSERNAME=admin -e ME_CONFIG_MONGODB_ADMINPASSWORD=password -e ME_CONFIG_MONGODB_SERVER=mongodb --name mongo-express --net mongo-network
```

`mongo-express`是操作`mongodb`的GUI，其中的`ME_CONFIG_MONGODB_SERVER`就是要連接的MongoDB容器名，這裡可以用`docker ps`去查（其實就是`mongodb`這個識別名）。

把二個容器跑起來後，可以在terminal輸入`mongo`和`mongo-express`，會輸出二個容器的獨特ID，可以拿它來查詢目前容器的啟動、執行狀況：

```
docker logs <id>
```

例如：放入新資料到MongoDB中，我們也可以選擇不同的模式去監控容器狀態：

```
docker logs <id> | tail // 僅看最後一次操作
docker logs <id> -f // 即時監控的stream mode
```

## 以`docker-compose`建立多個服務
有時實作的App可能涵蓋多種服務內容，因此必須設定多組容器和環境，這時候如果再用`docker run`一個一個跑起來和設定就比較沒效率。取而代之的，可以寫`docker-compose.yaml`一次跑起多組服務，這也是「Configuration as Code」強大之處，僅用程式碼就能架構所需要的設定。

在這裡只需要將各個容器列在services下，而參數得像解釋則可以參考[Kafka-learn](https://github.com/urjjj0909/kafka-learn)。需要注意的是`mongo-express`必須等到`mongodb`啟動完成才能夠連線，所以在`docker-compose.yaml`中可以在`mongo-express`這邊加入`depends_on`參數，或是加入`restart`也可以：

```
version: '3'
services:
  mongodb:
    image: mongo
    ports:
      - 27017:27017
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=password
    volumes:
      - mongo-data:/data/db
  mongo-express:
    image: mongo-express
    depends_on: [ mongodb ]
    // restart: always // fixes MongoNetworkError when mongodb is not ready when mongo-express starts
    ports:
      - 8080:8081
    environment:
      - ME_CONFIG_MONGODB_ADMINUSERNAME=admin
      - ME_CONFIG_MONGODB_ADMINPASSWORD=password
      - ME_CONFIG_MONGODB_SERVER=mongodb
```

使用`docker-compose.yaml`把容器都跑起來會預設一個網域，如果想要特別指定每個服務的網域，則可以自行在每個服務下加入`networks`參數。

## 建立NodeJS的Docker image
首先，撰寫`Dockerfile`將`server.js`包裝成Docker image：

```
FROM node:13-alpine

ENV MONGO_DB_USERNAME=admin \
    MONGO_DB_PWD=password

RUN mkdir -p /home/app

COPY ./app /home/app

# set default dir so that next commands executes in /home/app dir
WORKDIR /home/app

# will execute npm install in /home/app because of WORKDIR
RUN npm install

# no need for /home/app/server.js because of WORKDIR
CMD ["node", "server.js"]
```

* `RUN`指令是跑在容器內部，所以若想要複製Host的檔案到容器內部要使用`COPY`而非`RUN`
* 通常在Dockerfile內也可以指定要跑這個App的相關環境變數`ENV`，但通常環境變數都會到後來的`docker-compose.yaml`裡面寫，原因在於如果在這裡就把環境變數寫死，後續在Runtime環境想要更改設定，這份Dockerfile就要重新build起來（大準則就是僅設定必要參數，在後續啟動可能會改變的參數都不要寫進去）
* `CMD`這行執行的是`node server.js`指令（前提是前面有`FROM`先把node環境給建起來）。`CMD`這行指令又可以稱為`server.js`的entrypoint command，當然也可以用`RUN`去取代，但就比較不符合整個Docker文件的結構。所以通常一份Dockerfile可以有很多`RUN`、`COPY`，但只會有一行`CMD`放在最後

寫完Dockerfile後就可以用`docker build -t my-app:1.0 .`把它build起來。`-t`的意思就是tag，代表這個image的標籤、而`.`的意思表示目前所在的目錄位置，這個位置就是Dockerfile所在的路徑。在跑完build後用`docker images`就可以看到`my-app`已經建立完成，另外，要注意每當Dockerfile有修改就要重新build一次。

如果要把image刪除，需要按照「停止容器、刪除容器、刪除image」這樣的順序執行：

```
docker ps -a | grep my-app // 得到my-app容器詳細資訊
docker stop <id> // 指定id停止該容器
docker rm <id> // 指定id刪除該容器
docker images // 得到image詳細資訊
docker rmi <id> // 指定id刪除該image
docker images // 檢查是否刪除成功
```

成功把`server.js`容器跑起來後，可以直接到該容器內的terminal去看執行狀況：

```
docker ps
docker exec -it <id> /bin/bash // 或是用docker exec -it <id> /bin/sh
```

其中，`-it`是代表interactive terminal的意思，而有些容器環境內是安裝bash、有些是安裝shell，所以都可以試試看。而在進入terminal後可以用`env`去檢查該容器內的環境變數。
