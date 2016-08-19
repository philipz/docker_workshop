#Serverless 架構 & Docker
目前服務於 Docker 公司的 Ben Firshman 於 DockerCon 2016 Hackathon 之作品

原 fig 作者(被 Docker 併購，後來變成 Docker Compose) - Ben Firshman ，在 DockerCon 2016 Hackathon 分享他個人的小作品，[相關內容](https://github.com/bfirsh/serverless-docker)，[無伺服器架構](https://read01.com/RRQMLQ.html)，其中 [Docker 官方部落格](https://blog.docker.com/2016/06/building-serverless-apps-with-docker/)有解說整個實作內容，[簡體翻譯](https://linux.cn/article-7525-1.html)。是將第一個課程官方 Web 投票微服務範例改寫成 serverless 。

#準備環境
##Install Docker Compose 1.8.0
**請注意**，Docker Composer V2 需要配合 **Docker 1.10** 之後版本，
依照 [Compose 文件](https://github.com/docker/compose/releases)，執行 ```curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > ./docker-compose```{{execute}} 
和 `chmod +x ./docker-compose && sudo mv ./docker-compose /usr/local/bin/docker-compose`{{execute}} 
，完成 docker-compose 安裝，確認版本 `docker-compose -v`{{execute}} 。

##Install Docker 1.12
執行 `sudo apt-get -y update`{{execute}} 
更新套件，接著安裝新版 Docker， `sudo apt-get install -y docker-engine`{{execute}} 
，選擇保留原本設定，按 `N`{{execute}} ，完成後，執行 `docker -v` 確認版本。

##複製範例程式庫
從 Github 複製 Docker 無伺服器範例，執行 `git clone https://github.com/bfirsh/serverless-docker-voting-app`{{execute}}。

##啟動範例
切換到範例目錄， `cd serverless-docker-voting-app`{{execute}}，
接著需輸入 `docker-compose -f docker-compose.build.yml build && docker-compose build && docker pull postgres:9.4`{{execute}}，即開始建置。

#啟動無伺服器範例
##耐心等待是必要的 T_T
![Pokemon Go BEN Screen](https://cloud.githubusercontent.com/assets/664465/17803186/6b435504-6627-11e6-8694-4291bafb03c3.PNG)

##啟動範例
Build 好映像檔，就執行 `docker-compose up -d`{{execute}}。

![shia-labeouf-magic](https://cloud.githubusercontent.com/assets/664465/17802850/3e967362-6625-11e6-9214-703da1bc1826.gif)

請點選上方的 + 符號，選擇「Web Preview host port 80」，網址後加上 **vote/** 可看到投票畫面，加上 **result/** 可看到結果畫面。

輸入 `docker ps`{{execute}} 查看運作過程。

#SUMMARY
[AWS Lambda](http://www.ithome.com.tw/news/93920)服務的推出，進化到無需自行管理維護伺服器，並在流量大時自動擴展規模，而[淺析 serverless 架構與實作](http://abalone0204.github.io/2016/05/22/serverless-simple-crud/)文章有更深入介紹，近期更有針對此架構所發展的 [Serverless Framework](http://serverless.com/
)。
延伸閱讀：
[Martin Fowler - Serverless](http://martinfowler.com/bliki/Serverless.html)
[Martin Fowler - Serverless Architectures](http://martinfowler.com/articles/serverless.html)
