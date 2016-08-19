#Docker Swarm for MySQL Cluster & WordPress
建立 MySQL Galera 叢集，用 Swarm Cluster 快速建立 Master/Master DB 架構
此範例將上個練習 [Docker Compose for MySQL Cluster & WordPress](https://philipz.github.io/tech/2016/08/19/compose_galera_wp.html) ，改成 Docker new Swarm 方式，部署到多台伺服器，與前面練習相同，再配合 WordPress，達到動態配置和負載平衡的分散式架構，延伸 [Docker-Compose & Service Discovery](https://philipz.github.io/tech/2016/08/02/compose_dns.html) 課程，練習使用 docker swarm 及 docker service 等等新版指令，可動態擴展服務，訓練未來實際營運環境之使用經驗。

#New Swarm Mode Cluster
##建立 Swarm 叢集
先確認版本是否為 1.12?! `docker -v`{{execute}} 
，看一下這新 Swarm Mode 有哪些指令。 `docker swarm --help`{{execute HOST1}} 

接下來，初始化 Docker Swarm ，執行 `docker swarm init`{{execute HOST1}} 。
便會出現 Docker swarm join 的 Manager Token 和 Worker Token ，請先複製下來，在另一台伺服器加入叢集時需用到。

請選擇加入成為 Manager 或 Worker 的 Token：

Manager - `token=$(docker -H [[HOST_IP]]:2345 swarm join-token -q manager) && echo $token`{{execute HOST2}}

Worker - `token=$(docker -H [[HOST_IP]]:2345 swarm join-token -q worker) && echo $token`{{execute HOST2}}

加入到 Docker Swarm 叢集， `docker swarm join [[HOST_IP]]:2377 --token $token `{{execute HOST2}}。

叢集節點相關指令說明，`docker node -h`{{execute HOST1}}，
列出所有節點 `docker node ls`{{execute HOST1}}。

#建立 Galera Cluster
##新增 Docker Network
查看 Docekr Network 相關指令，`docker network -h`{{execute HOST1}}，
執行 `docker network create -d overlay galera`{{execute HOST1}}，
新增名為 galera 的層疊網路。

與上個練習中的 `docker network create mysql` 指令不同，須指定 overlay 層疊網路之驅動器，才能在 Swarm Cluster 中跨伺服器。

確認內容 `docker network ls`{{execute HOST1}} ，
若在上一步驟，以不同身分(Manager 和 Worker)加入，所產生的結果會有所不同，當以 Manager 身分加入，會立即同步其 Docker Network 的設定，因為其 Manager 的 K/V 是立即同步，若以 Worker 身分加入，則在服務部署到其伺服器時，才會生效。查看另一台 `docker network ls`{{execute HOST2}}，
詳細說明，請參閱[Understand Docker container networks](https://docs.docker.com/engine/userguide/networking/)。

##啟動 Galera 相關服務
###Master
所有的 Swarm 服務都必須在 Manager 上操作，而 Manager 數量建議為三台以上，原因請見[拜占庭將軍問題](http://www.twword.com/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%87%E8%BB%8D%E5%95%8F%E9%A1%8C)，由 [Leslie Lamport](http://news.sciencenet.cn/htmlnews/2015/12/333114.shtm) 所提出。
執行 `docker service create --name master --network galera erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://`{{execute HOST1}}

查看服務清單，`docker service ls`{{execute HOST1}}，
確認 Master 服務狀況 `docker service ps master`{{execute HOST1}}。

###Galera
`docker service create --name galera --network galera erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://master`{{execute HOST1}} 

檢查其狀況 `docker service ps galera`{{execute HOST1}} 、
`docker service ls`{{execute HOST1}} 

##確認 Galera Cluster 數量

查看容器狀況 `docker ps`{{execute HOST1}}，
登入到 Master 容器中，`docker exec -ti XXXXX bash`，
執行 `mysql -e 'show status like "wsrep_cluster_size"'`{{execute HOST1}}，
確認 DB 叢集狀況。

同樣的步驟，亦在另一台伺服器上執行。`docker ps`{{execute HOST2}}，
`docker exec -ti XXXXX bash`，`mysql -e 'show status like "wsrep_cluster_size"'`{{execute HOST2}}，
確認後離開`exit`{{execute HOST2}}。

#建立 Wordpress 服務
##準備資料庫 
再來設定 Wordpress 所需的資料庫和帳號權限。 
執行 `mysql`{{execute HOST1}}， 
資料庫：`CREATE DATABASE wordpress;`{{execute HOST1}}， 
帳號：`CREATE USER 'wordpress'@'%' IDENTIFIED BY 'PASSWORD';`{{execute HOST1}}， 
權限：`GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'%' WITH GRANT OPTION;`{{execute HOST1}}， 
最後跳出 mysql 程式 `exit`{{execute HOST1}}， 
並登出容器 `exit`{{execute HOST1}}。 

##啟動 Wordpress 服務
只需執行 `docker service create --name wordpress --network galera -p 80:80 -e WORDPRESS_DB_HOST=galera:3306 -e WORDPRESS_DB_USER=wordpress -e WORDPRESS_DB_PASSWORD=PASSWORD wordpress:4.5`{{execute HOST1}} 
 
查看服務清單，`docker service ls`{{execute HOST1}}， 
確認 Wordpress 服務狀況 `docker service ps wordpress`{{execute HOST1}}。
請點選上方的 + 符號，選擇「Web Preview host port 80」，就可看到 Wordpress 安裝畫面。

#Swarm 服務擴展與升級
##擴展 Wordpress 容器數量 
 與 Docker Compose 指令類似，只需 `docker service scale wordpress=2`{{execute HOST1}}，
就完成橫向擴展，查看一下服務狀況 `docker service ps wordpress`{{execute HOST1}} 。
 
請點選另一台伺服器上方的 + 符號，選擇「Web Preview host port 80」，亦可看到 Wordpress 安裝畫面。

如果要確認 Swarm 叢集的自動負載平衡，可以登入到 Wordpress 其中一個容器，新增一個 test.html
，`echo "Test LB!" > test.html`，再用瀏覽器查看 test.html 的結果，就可發現，更新網頁兩次就會出現一次，即便在同一台伺服器上。

##捲動式更新服務(Rolling updates)
 新版 Swarm 還提供[捲動更新功能](https://docs.docker.com/engine/swarm/swarm-tutorial/rolling-update/)，只需要執行 
``` 
docker service update \ 
  --name wordpress \ 
  --image wordpress:latest \ 
  --update-delay 10s \ 
  --update-parallelism 1 \ 
  wordpress 
```{{execute HOST1}} 

就可將 Wordpress 更新至最新版。

查看 Wordpress 服務更新狀況， `docker service ps wordpress`{{execute HOST1}}。

#SUMMARY
此示範將上一個只適用於開發測試環境的範例，使用 Docker Swarm 容器協同運作管理工具，部署到多台虛擬主機上。

亦善用這 overlay 層疊網路，很輕鬆就可配置出可動態擴展的架構，並且無須額外設定，即可達到動態配置和負載平衡。

各位可嘗試將[投票微服務範例](https://editor.katacoda.com/editor/docker/courses/docker2016/1)改成 Swarm 服務，而目前 Docker Compose 定義檔部署到 Swarm Cluster，需先轉成 [Distributed Application Bundles(DAB)](https://blog.docker.com/2016/06/docker-app-bundle/) 檔案格式，且仍處於[實驗階段](https://github.com/docker/docker/releases)。不過 Docker Compose 已經可以產生出 DAB 檔案， **docker-compose bundle -h**。
