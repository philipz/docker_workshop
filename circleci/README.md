#Docker Compose & CircleCI
利用投票微服務範例，來示範如何使用 Github 結合  CircleCI 自動化建構微服務架構中的單一容器

示範 Docker 在 CircleCI 的持續整合(CI)、持續交付(CD)之應用，以[之前的投票微服務課程](https://philipz.github.io/tech/2016/08/08/vote_microservice.html)中的 Vote 服務，修改其 Python 程式來操作，在 CircleCI 建置出 Docker 映像檔，並整合 Docker Compose 建置出開發環境，撰寫 Test Script (docker run、curl，或是其他測試軟體等) ，自動測試並驗證其程式正確性，再發布到 Docker Hub 上。

#複製程式碼

##下載範例程式
首先複製投票範例程式，`git clone https://github.com/docker/example-voting-app`{{execute}}，
切換到 *example-voting-app* 目錄，`cd example-voting-app`{{execute}}，
查看是否包含 Python 語言寫的 vote 目錄，`ls -l`{{execute}}，
回到 HOME 目錄，`cd ~`{{execute}}。

##建立 Git Repo.
登入到 Github ，在 Repositories 頁面，新增一個 Repo. ，如下圖：

![git_new1](https://cloud.githubusercontent.com/assets/664465/17506273/0fefed38-5e3a-11e6-91f5-b16e0ca998a2.png)

輸入專案名稱，**circlecidemo**，再按下 *Create repository* 按鈕。

![git_name](https://cloud.githubusercontent.com/assets/664465/17505759/f1645bf0-5e35-11e6-9c99-7907a844cd7c.png)

這樣便建立好一個空的 git 儲存庫，接著複製到本地端， `git clone https://github.com/philipz/circlecitest`{{execute}}，
進入到 circlecitest 目錄， `cd circlecitest`{{execute}}。

##複製 vote 程式
將 *example-voting-app* 底下 *vote* 目錄中的所有內容，複製到 *circlecitest*，`cp -R ~/example-voting-app/vote/* ./`{{execute}} ，
便完成程式碼儲存庫的設定和配置。

#撰寫 Docker Compose 定義檔

因為這投票微服務範例，包含五個容器服務元件，若不在 CI 環境上做系統整體測試，只做單元測試可能會有所疏漏，所以我們亦在 CircleCI 上使用 Docker Compose ，建置出一模一樣的環境，免除這樣的測試漏洞。所以複製 *example-voting-app* 目錄下的 **docker-compose.yml** 到 *circlecitest* 目錄中，`cp ~/example-voting-app/docker-compose.yml ./`{{execute}}。  
   
這範例主要是針對 Python 程式部分來建置 
vote 服務的 *philipz/votingapp_vote* 映像檔，所以改寫這 **docker-compose.yml** 檔案，將非 vote 服務的四個容器都採用映像檔方式，改寫成下面內容： 
```  
version: "2" 
 
services: 
  vote: 
    image: philipz/votingapp_vote 
    command: python app.py 
    ports: 
      - "5000:80" 
 
  redis: 
    image: redis:alpine 
    ports: ["6379"] 
 
  worker: 
    image: philipz/votingapp_worker 
 
  db: 
    image: postgres:9.4 
 
  result: 
    image: philipz/votingapp_result 
    ports: 
      - "80:80" 
      - "5858:5858" 
```  
可直接複製現成檔案，`wget https://raw.githubusercontent.com/philipz/circlecitest/master/docker-compose.yml`{{execute}} 。

#準備測試腳本

在 [example-voting-app/result](https://github.com/docker/example-voting-app/tree/master/result) 目錄中，其實有 Node.js 的測試程式，主要是透過 [PhantomJS](http://phantomjs.org/) 工具來測試，跟之前一樣複製到 *circlecitest* 目錄中，`cp -R ~/example-voting-app/result/tests ./`{{execute}}，
而原本的測試腳本是在本地端使用，需改寫成適合 Circle CI 環境，首先移除掉 while 這行 `while ! timeout 1 bash -c "echo > /dev/tcp/vote/80"; do sleep 1; done`，接著將 *http://vote* 改成 *http://localhost:5000* ，對應上一個步驟 **docker-compose.yml** 檔案中 vote 服務的 *ports:  - "5000:80"*，再將 *http://result* 改成 *http://localhost* ，正好對應 result 服務的 *ports:  - "80:80"*，改寫之後的結果如下：
```
#!/bin/sh 
curl -sS -X POST --data "vote=b" http://localhost:5000 > /dev/null 
sleep 10 
if phantomjs render.js http://localhost | grep -q '1 vote'; then 
  echo -e "\e[42m------------" 
  echo -e "\e[92mTests passed" 
  echo -e "\e[42m------------" 
  exit 0 
fi 
  echo -e "\e[41m------------" 
  echo -e "\e[91mTests failed" 
  echo -e "\e[41m------------" 
  exit 1
```
可直接複製現成檔案，`wget 
https://raw.githubusercontent.com/philipz/circlecitest/master/tests/tests.sh`{{execute}} 。

#撰寫 Circle CI 設定檔

接下來，新增 *circleci.yml* 檔案，內容如下，可直接複製現成檔案。 `wget https://raw.githubusercontent.com/philipz/circlecitest/master/circle.yml`{{execute}}  
``` 
machine: 
  pre: 
    - curl -sSL https://s3.amazonaws.com/circle-downloads/install-circleci-docker.sh | bash -s -- 1.10.0 
  services: 
    - docker 
  node: 
    version: 0.10.22 
     
dependencies: 
  override: 
    - docker info 
    - docker -v 
    - docker build -t philipz/votingapp_vote . 
test: 
  override: 
    - curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > ./docker-compose 
    - chmod +x ./docker-compose 
    - ./docker-compose -v 
    - ./docker-compose up -d 
    - npm install -g phantomjs 
    - cd test && ./tests.sh 
deployment: 
  hub: 
    branch: master 
    commands: 
      - docker login -e $DOCKER_EMAIL -u $DOCKER_USER -p $DOCKER_PASS 
      - docker push philipz/votingapp_vote 
```  
那 Circle CI 已經支援 Docker 服務，為何又要重新安裝？因為 Docker Engine 要 1.10 版之後才支援 Docker Compose version 2，而 Circle CI 預設的 Docker Engine 版本太舊，所以必須自行安裝，因此仍舊需要使用 `service: - docker` 。  
 
再來配合上一個步驟的測試腳本，選擇 Node.js 環境，`node:  version: 0.10.22`。而建置程式過程都寫在 `dependencies:` 這區塊，其實只有一行 `docker build -t philipz/votingapp_vote .`。接著是測試區塊 `test:` ，先安裝新版 **Docker Compose** ，再把整個投票微服務系統執行起來， `
docker-compose up -d`，最後安裝 PhantomJS 和執行上一步驟所準備的 *tests.sh* 腳本。

##上傳到 Github
全部檔案都改寫好，就新增所有檔案， `git add *`{{execute}} ，
確認 `git commit -m "First init"`{{execute}} ，
最後上傳 `git push`{{execute}} 。

#使用 Circle CI 服務

最後一個步驟就是利用 [Circle CI](https://circleci.com/) 來建置並發佈到 Docker Hub 上。

##新增帳戶
按下那 *Sign Up* 按鈕：

![circle_signup](https://cloud.githubusercontent.com/assets/664465/17509832/91503752-5e4e-11e6-9879-48d3a76b0d44.png)

並直接使用 Github 帳號認證，直接按下 *Authorize with GitHub* 按鈕，便完成新增帳號。

##新增建置專案
在 [Circle CI 儀錶板](https://circleci.com/dashboard)上選擇， **ADD PROJECTS** ，點選 Github 選項。

![dashboard](https://cloud.githubusercontent.com/assets/664465/17509997/693844a2-5e4f-11e6-8e37-c70db5d31c95.png)

接著尋找 circlecitest 專案，按下 **Build project** ，就立即開始建置。
![circle_build](https://cloud.githubusercontent.com/assets/664465/17510281/df48780a-5e50-11e6-94b1-27e6e2eeee3c.png)

##發佈到 Docker Hub 設定
緊接著，若想在建置成功後，設定此建置專案的環境變數。按下專案旁的齒輪按鈕：

![circlecitest_setting](https://cloud.githubusercontent.com/assets/664465/17510772/de0beb8c-5e52-11e6-95cb-8b4602763264.png)

選擇環境變數， *Environment Variables* 

![env_var](https://cloud.githubusercontent.com/assets/664465/17510877/53b81270-5e53-11e6-9aa1-9f6fed21a687.png)

新增三組環境變數，分別是您的 Docker Hub 帳號名、 Docker Hub EMAIL 和密碼。
![dockerhub_var](https://cloud.githubusercontent.com/assets/664465/17510942/9fbb2bbc-5e53-11e6-8d49-aba64cb553fd.png)

最後就可看到建置成功並上傳的 **綠燈** 。

![green_light](https://cloud.githubusercontent.com/assets/664465/17511626/61416ec0-5e56-11e6-8b3b-dc1e37ca02cb.png)

#SUMMARY

這練習利用之前的 [Docker 官方 Web 投票微服務範例](https://philipz.github.io/tech/2016/08/08/vote_microservice.html)，來示範如何使用 [Github](https://github.com/) 和 [CircleCI](https://circleci.com/) 
透過 Docker 容器達到軟體[持續整合](https://zh.wikipedia.org/wiki/%E6%8C%81%E7%BA%8C%E6%95%B4%E5%90%88)(Continuous Integration))和[持續交付](https://zh.wikipedia.org/wiki/%E6%8C%81%E7%BA%8C%E4%BA%A4%E4%BB%98)(Continuous Delivery)，
讓整個微服務架構中的單一容器元件可以自動化建置、測試後，上傳到企業內部的 [Docker registry server](https://github.com/docker/distribution/blob/master/docs/deploying.md) 存放映像檔，甚至呼叫 Webhook API 來自動部署營運環境。

延伸閱讀：
1. [此範例的 Github 專案](https://github.com/philipz/circlecitest)
2. [CircleCI 建置結果](https://circleci.com/gh/philipz/circlecitest)
3. [Continuous Delivery中文版](http://www.books.com.tw/products/0010653820)
4. [ThoughtWorks 的 CI 介紹](https://www.thoughtworks.com/continuous-integration )
5. [ThoughtWorks 的 CD 介紹](https://www.thoughtworks.com/continuous-delivery)
