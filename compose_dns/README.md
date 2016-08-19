#Docker-Compose & Service Discovery
##透過 Docker 1.12 解釋其 Docker Network 機制和 DNS-based Service Discovery 

此操作範例是介紹 Docker Compose 的使用方式及其指令，並解釋 Docker 1.12 版新增的 DNS-based Service Discovery 機制，以及 Docker Network 的 VLAN 容器隔離功能，希望能幫忙您撰寫自己的 *docker-compose.yml* 檔案，並設計所需系統架構。

#準備環境
##Install Docker 1.12
執行 `sudo apt-get -y update`{{execute}} 
更新套件，接著安裝新版 Docker， `sudo apt-get install -y docker-engine`{{execute}} 
，選擇保留原本設定，按 `N`{{execute}} ，完成後，執行 `docker -v` 確認版本。
##Install Docker Compose 1.8.0
依照 [Compose 文件](https://github.com/docker/compose/releases)，執行 ```curl -L https://github.com/docker/compose/releases/download/1.8.0/docker-compose-`uname -s`-`uname -m` > ./docker-compose```{{execute}} 
和 `chmod +x ./docker-compose && sudo mv ./docker-compose /usr/local/bin/docker-compose`{{execute}} 
，完成 docker-compose 安裝，一樣確認版本 `docker-compose -v`{{execute}}  。

#使用 Docker Compose
##複製範例庫
`git clone https://github.com/philipz/compose_dns`{{execute}} ，
接著 `cd compose_dns`{{execute}} 。

##docker-compose.yml
此範例的 *docker-compose.yml* 定義檔內容如下：
```
version: '2' 
 
services: 
   webapp: 
       image: philipz/minimal 
 
   webproxy: 
       build: webproxy 
       ports: 
        - "80:80"
```
由兩個服務所構成，*webapp* 是直接使用之前建置的 **philipz/minimal** 映像檔，而 *webproxy* 則是透過 webproxy 目錄底下的內容來建置所需映像檔。
##webproxy
*webproxy* 目錄包含兩個檔案，Nginx 所需要的設定檔 *proxy.conf* 和建置映像檔的 *Dockerfile* 。其中 *proxy.conf* 設定檔內容為：
```
server {
    listen 80;

    location / {
        proxy_pass http://webapp:8000;
    }
}
```
透過 proxy_pass 功能將 HTTP request 轉送到後端的 webapp 容器的 8000 port ，單純提供 reverse proxy 功能。

而 *Dockerfile* 是以 nignx:alpine 映像檔為基礎，將上述的 *proxy.conf* 覆蓋原本的配置設定。
```
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/*
COPY proxy.conf /etc/nginx/conf.d/
```
最後先建置此範例 `docker-compose build`{{execute}}
，再執行 `docker-compose up -d`{{execute}} 以背景模式運行。

#Service Discovery
請點選上方的 + 符號，選擇「Web Preview host port 80」，就可看到執行結果。
##Docker Network
啟動後，我們使用 `docker-compose ps`{{execute}} 查看這容器服務的運作狀況，必須在 *docker-compose.yml* 檔案相同的目錄才能正常執行。

接下來使用 `docker network ls`{{execute}} 
指令可看到 Docker Compose 為這範例建立了 *composedns_default* 這橋接器模式的虛擬網路，兩個容器服務都歸到相同的網段，若直接使用 `docker run -ti --rm nginx:alpine sh`{{execute}} 
所啟動的容器，其 IP 網段將與這範例不同， `ifconfig`{{execute}}。這是 Docker 針對容器所作的網路隔離特性。
##DNS
先登出 *nginx:alpine* `exit`{{execute}} 
，改進入到 *webproxy* 容器中， `docker exec -ti composedns_webproxy_1 sh`{{execute}}，
執行 `ifconfig`{{execute}} 
可看出其 IP 網段就與上面容器不同，接著查看 DNS 設定， `cat /etc/resolv.conf`{{execute}} ，
那 **127.0.0.11** 是 Docker 所內建的 DNS 伺服器，目的是為了服務探索所需要的容器名稱與 IP 對應，執行 `nslookup webapp`{{execute}} ，
就可顯示出 webapp 容器所使用的 IP ，登出此容器 `exit`{{execute}}。

接著增加 *webapp* 容器數量到 3，`docker-compose scale webapp=3`{{execute}} ，
再 `docker-compose ps`{{execute}} 
查看服務的啟動情況。再登入到 *webproxy* 容器中， `docker exec -ti composedns_webproxy_1 sh`{{execute}}，
一樣輸入 `nslookup webapp`{{execute}} ，
看看這次 webapp 的 DNS 名稱有何差異？可看出，這 DNS 紀錄增加到三個。執行 `nginx -s reload`{{execute}} ，
重新載入主機位置，使 *webproxy* 能正確對應，此刻點選上方的 + 符號，選擇「Web Preview host port 80」，重新整理可看到每次都是不同樣的結果，是以 DNS round-robin 方式循環。而這樣的功能也具備負載平衡的好處。

#Summary
利用這樣 DNS-based Service Discovery 機制，可免除掉架構上很多瑣碎的設定，直接以服務名稱就可查詢，讓 Nignx 原本 reverse proxy 功能還同時具備了 load balancing 作用。而這個操作情境，還介紹了 docker-compose 常用的指令，若需要更詳細的使用方法，建議直接使用 `docker-compose -h`{{execute}} 查詢，或閱讀 [Compose 官方文件](https://docs.docker.com/compose/)。
