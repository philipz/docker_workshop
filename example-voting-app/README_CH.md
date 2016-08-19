#Docker 官方 Web 投票微服務範例
利用 Docker Birthday #3 所提供的範例，來說明利用 Docker 建構微服務架構系統

出處：[Wiki](https://zh.wikipedia.org/wiki/%E5%BE%AE%E6%9C%8D%E5%8B%99)

微服務 (Microservices) 是一種軟體架構風格 (Software Architecture Style)，它是以專注於單一責任與功能的小型功能區塊 (Small Building Blocks) 為基礎，利用模組化的方式組合出複雜的大型應用程式，各功能區塊使用與語言無關 (Language-Independent/Language agnostic) 的 API 集相互通訊。

而 [Docker Birthday #3](https://github.com/docker/docker-birthday-3) 所提供的 Voting 範例就是以微服務方式所設計，結合 PHP、Redis、Jave、Node.js 和 Postgres ，五個單一責任的容器，組合出這一個 Web 投票系統。

#投票系統架構

整個微服務架構如下：

![bd3-architecture](bd3-architecture.png)

* Python Web 程式提供兩個選項來投票
* Redis 佇列來收集 PHP 所選的新選票
* Java worker 負責監聽 Redis 的投票佇列，再轉存到 Postgres
* Postgres 資料負責持久儲存，以 Docker volume 掛載
* Node.js Web 程式即時顯示所有投票結果

##Install Docker Compose 1.8.0
依照 [Compose 文件](https://github.com/docker/compose/releases)，執行 ```curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > ./docker-compose```{{execute}} 
和 `chmod +x ./docker-compose && sudo mv ./docker-compose /usr/local/bin/docker-compose`{{execute}} 
，完成 docker-compose 安裝，一樣確認版本 `docker-compose -v`{{execute}} 。

##複製範例庫
`git clone https://github.com/philipz/example-voting-app`{{execute}}

#使用 Docker Compose 建置

切換到 example-voting-app 目錄，`cd example-voting-app`{{execute}} ，
接著只要執行 `docker-compose up -d`{{execute}} 
就完成整個投票系統建置。執行 `docker-compose ps`{{execute}} 
確認整個系統狀況。

請點選上方的 + 符號，選擇「Web Preview host port 80」，就可看到投票執行網頁。再點選上方的 + 符號，選擇「Change Port」，會看到修改 Port 變更網頁，輸入 5000 ，按下 Display Port 就可看到投票網頁。畫面如下：

![change_port](https://cloud.githubusercontent.com/assets/664465/17457101/dc2b84ba-5c1f-11e6-8b50-19d5ed691dfd.png)

##解說 docker-compose.yml
```
version: "2"

services:
  vote:
    build: ./vote
    command: python app.py
    volumes:
     - ./vote:/app
    ports:
      - "5000:80"

  redis:
    image: redis:alpine
    ports: ["6379"]

  worker:
    build: ./worker

  db:
    image: postgres:9.4

  result:
    build: ./result
    command: nodemon --debug server.js
    volumes:
      - ./result:/app
    ports:
      - "80:80"
      - "5858:5858"

```
首先，宣告此 Compose 格式為 version 2 ，必須配合 Docker Engine 1.10 之後的版本使用。接下來定義如前面所描述的五個微服務元件。
1. Vote: 使用 vote 目錄來建置映像檔，其中目錄包含 Dockerfile 及其相關程式檔案。以 Python 執行 app.py，掛載 vote 到容器的 /app 目錄中，對應本機 5000 port 到容器的 80 port。
2. Redis: 直接使用 redis:alpine 映像檔，並隨機對應本機連接埠到容器 6379 port。 
3. Worker: 使用 worker 目錄建置，其目錄包含 Dockerfile 、 src 目錄程式和 pom.xml 函式庫定義檔。
4. DB: 直接使用 porstgres:9.4 映像檔。
5. Result: 使用 result 目錄建置，其目錄除了 Node.js 相關程式，還包含測試程式(tests目錄)，預設執行 `nodemon --debug server.js` ，並掛載 result 到容器的 /app 目錄中，對應本機 80 port 到容器的 80 port，以及 Remote debug port 5858。

#改用 .NET 版的 Worker

既然是微服務架構，便可方便抽換其中一個容器服務元件，因此我們將那 **Java** 版本的 *Worker*，替換成 **.NET** 所寫的*新 Worker*，新架構如下：

![architecture](https://cloud.githubusercontent.com/assets/664465/17456546/ba03d8a4-5c0d-11e6-94a5-31f5e1432edb.png)

##改寫 docker-compose.yml
先停止之前所啟動的容器，`docker-compose down`{{execute}}，
將原本 *docker-compose.yml* 檔案中的 worker 服務，改用成 .NET 的 worker。首先將原本內容從，
```
worker: 
    build: ./worker
```
改成
```
worker.net:
    build: ./worker.net
    depends_on: 
      - redis 
      - db
```
因 worker.net 啟動很快，若沒有加上相依容器服務，會找不到其主機位置而跳出容器，產生錯誤。

為方便操作，已修改好 *docker-compose.net.yml* 檔案，查看其內容，`cat docker-compose.net.yml`{{execute}} ，
為了讓此範例運作更順利，就直接使用建置好的映象檔 **philipz/worker.net**，若想了解 .NET Worker 建置內容，可自行在電腦上重新建置，*docker-compose build*。我們直接覆蓋原本檔案，執行 `mv docker-compose.net.yml docker-compose.yml`{{execute}} ，
再次執行 `docker-compose up -d`{{execute}} 
就完成整個投票系統的替換建置。

再次點選上方的 + 符號，選擇「Web Preview host port 80」，就可看到投票執行網頁。再點選上方的 + 符號，選擇「Change Port」，會看到修改 Port 變更網頁，輸入 5000 ，按下 Display Port 就可看到投票網頁。執行 `docker-compose ps`{{execute}}  
確認整個系統狀況。

#SUMMARY

本範例，示範了 Docker Compose 相關功能及操作，並且利用 Docker Birthday #3 所提供的範例，來說明如何利用 Docker 建構微服務架構系統，並且很方便就可替換架構中的容器服務元件，讓整個架構可以隨著組織業務成長來擴展，可以隨著技術演化來替換堆疊，靈活彈性的架構可應付不斷改變的需求和變化。
