#Docker Compose for MySQL Cluster & WordPress
建立 MySQL Galera 叢集，用 Docker Compose 快速建立 Master/Master DB 架構

示範如何將書中 MySQL Galera 叢集，改寫 Compose YAML檔案及參數，建置出 DB Mater/Master 架構，再配合 WordPress，以及上一個練習，[Compose & Service Discovery](https://philipz.github.io/tech/2016/08/02/compose_dns.html) 所學習到的 DNS-based 負載平衡機制，架設 WP 的 HA （高可用性）架構。
##Galera Cluster 原出處：
* [Getting started Galera with Docker, part 1](http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-1/) 
* [Getting Started Galera with Docker, part 2](http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-2-2/)

#準備環境
##Install Docker Machine 0.8.0
為快速建立雲端虛擬機，請安裝 Docekr Machine ，執行 ```wget https://github.com/docker/machine/releases/download/v0.8.0/docker-machine-`uname -s`-`uname -m` && \ 
sudo mv docker-machine-`uname -s`-`uname -m` /usr/local/bin/docker-machine && \ 
sudo chmod +x /usr/local/bin/docker-machine```{{execute}}，
確認版本 `docker-machine -v`{{execute}} 。

##Install Docker 1.12
執行 `sudo apt-get -y update`{{execute}} 
更新套件，接著安裝新版 Docker， `sudo apt-get install -y docker-engine`{{execute}} 
，選擇保留原本設定，按 `N`{{execute}} ，完成後，執行 `docker -v` 確認版本。
##Install Docker Compose 1.8.0
Docker Composer V2 需要配合 Docker 1.10 之後版本，
依照 [Compose 文件](https://github.com/docker/compose/releases)，執行 ```curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > ./docker-compose```{{execute}} 
和 `chmod +x ./docker-compose && sudo mv ./docker-compose /usr/local/bin/docker-compose`{{execute}} 
，完成 docker-compose 安裝，一樣確認版本 `docker-compose -v`{{execute}} 。

##建立 Azure 虛擬機
為真實模擬實際營運環境，使用 Docker Machine 快速建立 VM ，執行 `docker-machine create -d azure --azure-subscription-id="XXXXX" --azure-location="westeurope" --azure-image canonical:ubuntuserver:16.04.0-LTS:16.04.201608150 --azure-size Standard_A1 --engine-install-url https://get.docker.com docker-00-node01`{{execute}}，
建立完成，利用 **docker-machine env** 指令將 Docker Engine Server 指定到遠端 Azure 主機， `eval $(docker-machine env docker-00-node01)`{{execute}}，
輸入 `docker info`{{execute}} 確認電腦資訊。

#解說 Galera 定義檔
##Docker Network
Docker Network 是從 Docker 1.9 所新增的功能，可用來建立一個跨多台 Docker 伺服器的層疊網路，其利用 VXLAN 通訊協定。層疊技術的好處在於它提供每個容器具有相同子網路的 IP 位址，並且也能透過變更每個容器的 /etc/hosts 來管理名稱解析，因此每個容器都啟動在相同層疊網路，用各自的容器名稱,就可連線到其他任何一個容器。

執行 `docker network -h`{{execute}}
，來查看有哪些指令。

##複製範例庫
請執行 `git clone https://github.com/philipz/compose_galera_wordpress`{{execute}}，
切換到此範例目錄，`cd compose_galera_wordpress`{{execute}}，
首先，來看 galera.yml 定義檔內容：
```
version: "2"

services:
  master:
    image: erkules/galera:basic
    container_name: master
    volumes:
      - /data:/var/lib/mysql
    networks:
      - mysql
    command: --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://

  slave:
    depends_on:
      - master
    image: erkules/galera:basic
    networks:
      - mysql
    command: --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://master

networks:
  mysql:
```
主要是由兩個服務所組成，分別是 Master 和 Slave ，但此 Galera Cluster 是 Master-Master 資料庫架構，此命名只是方便識別。
Master 服務先啟動，方便 Slave 利用那 *Command:* `--wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://master` 來連線到 Master，並且加上 *depends_on:* 設定，確保 Master 服務啟動好，才執行 Slave 服務，相關啟動順序細節，請參閱 [Controlling startup order in Compose](https://docs.docker.com/compose/startup-order/)。

最下面 *networks:* 則是建立 Docker Network ，與 `docker network create mysql` 指令相同。而 Master 和 Slave 服務都設定在 mysql 層疊網路上。

#建立 Galera Cluster
##啟動 Galera Cluster
瞭解了 Compose 定義檔內容，接著就是啟動這服務。執行 `docker-compose -f galera.yml up -d`{{execute}}，
等待 pull 那 Galera 映像檔後，就建置完成。

顯示服務的標準輸出結果，可執行 `docker-compose -f galera.yml logs slave`{{execute}}，
確認服務是否有錯誤訊息。

若其中有服務未完成啟動，可執行 `docker-compose -f galera.yml up -d slave`{{execute}}，
嘗試再一次啟動容器服務。

查看其服務的狀態， `docker-compose -f galera.yml ps`{{execute}}，
為何不直接執行 `docker-compose ps`{{execute}}，
反而加上 **-f galera.yml** 是因為，docker-compose 預設使用檔案名稱為 *docker-compose.yml* ，若不用此名稱，就需要透過 **-f** 指定。

確認啟動完成，再登入到容器中，確認資料庫叢集狀態，`docker exec -ti master bash`{{execute}}，
進入容器後，執行 `mysql -e 'show status like "wsrep_cluster_size"'`{{execute}}，
即顯示目前叢集數量。

#解說 Wordpress 定義檔
##準備資料庫
再來設定 Wordpress 所需的資料庫和帳號權限。
執行 `mysql`{{execute}}，
資料庫：`CREATE DATABASE wordpress;`{{execute}}，
帳號：`CREATE USER 'wordpress'@'%' IDENTIFIED BY 'PASSWORD';`{{execute}}，
權限：`GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'%' WITH GRANT OPTION;`{{execute}}，
最後跳出 mysql 程式 `exit`{{execute}}，
並登出容器 `exit`{{execute}}。

##Wordpress 定義檔
接著，看一下 *wordpress.yml* 內容：
```
version: "2"

services:
  wordpress:
    image: wordpress
    networks:
      - mysql
    environment:
      - WORDPRESS_DB_HOST=slave:3306
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=PASSWORD
    ports:
      - "80:80"

networks:
  mysql:
```
與之前 *galera.yml* 定義檔相同，都是使用 mysql 的層疊網路，直接利用官方 wordpress 映像檔，故需定義相關環境變數，請注意那 *WORDPRESS_DB_HOST=slave:3306* 是連線到 slave 服務容器。

#建立 Wordpress
##啟動 Wordpress
執行 `docker-compose -f wordpress.yml up -d`{{execute}}，
等待 pull 那 Wordpress 映像檔後，便建置完成。

啟動後，執行 `docker-machine ls`{{execute}}，
用瀏覽器開啟此 IP 就可看到 Wordpress 安裝畫面，並依指示完整安裝。

##停止所有服務
都操作完成後，便先停止 Wordpress 服務，`docker-compose -f wordpress.yml down`{{execute}}，
再來停止全部 Galera 服務，`docker-compose -f galera.yml down`{{execute}}。
完成後，確認目前容器狀況，`docker ps`{{execute}}。

##合併 Galera 和 Wordpress
最後，將兩個定義檔合併成一個，請見 *galera_wordpress.yml* 內容，接著一次啟動所有服務， `docker-compose -f galera_wordpress.yml up -d`{{execute}}，
確認所有服務的狀態， `docker-compose -f galera_wordpress.yml ps`{{execute}}，
用瀏覽器開啟此虛擬機 IP ，就可看到 Wordpress 已安裝好畫面。

##動態擴展 
之前查看過資料庫叢集數量，此時動態增加 Slave 的數量，執行 `docker-compose -f galera_wordpress.yml scale wordpress=2`{{execute}}， 
擴展完成，可登入到 wordpress 容器中，加上 test.html 來驗證負載平衡功能。

#SUMMARY
此操作，示範如何使用 Docker Compose 和 Docker Network 整合之功能，善用這 overlay 層疊網路，很輕鬆就可配置出可動態擴展的架構，並且無須額外設定，即可達到動態配置和負載平衡，不過此範例只適用於開發測試環境，若要應用到營運環境，則須配合 Docker Swarm 容器協同運作管理工具，部署到多台虛擬主機上。
