#Minimal Node.js Docker Image
##Use Enclose to compile the binary execute file.
在這個實驗中，了解 Docker 映像檔的建立，及其檔案系統。
##scratch
**[scratch](https://hub.docker.com/_/scratch/)** 是 Docker 映像檔之母，所有基礎映像檔(Base Image)都是由 scratch 所建置而成。它其實是一個空的映像檔，把所需要的環境將其檔案系統匯入到其中，使得依賴這樣檔案系統的程式可以正確運作。
##相關文件
1. [Create a base image](https://docs.docker.com/engine/userguide/eng-image/baseimages/)
2. [從映像檔產生 Dockerfile](https://philipzheng.gitbooks.io/docker_practice/content/dockerfile/file_from_image.html)

#Just do "make"
##複製範例程式
`git clone https://github.com/philipz/node_minimal`{{execute}}
##安裝 Make 套件
`sudo apt-get install make `{{execute}}
##開始建置
先 `cd node_minimal`{{execute}} ，
接著只需要 `make`{{execute}} ，
執行完後查看是否有建置好的 Docker 映像檔 `docker images`{{execute}} ，這樣就建置好最小 Node.js 應用程式的 Docker 映像檔。
##確認
執行這最小 Node.js 應用程式的 Docker 映像檔， `docker run -d -p 80:8000 philipz/minimal`{{execute}} ，啟動後，請點選上方的 **+** 符號，選擇「Web Preview host port 80」，就可看到執行結果。

透過 `docker inspect philipz/minimal | more`{{execute}} 可以查看這映像檔的詳細資訊。

#How it works? Makefile
就來逐一解說，整個建置過程的每一個環節。
## Makefile
首先是利用 **GNU Make** 來執行定義好的 *Makefile* ，其中 *Makefile* 內容如下：
```
NAME = minimal 
APP_NAME = index.js 
NODE_VERSION = 6.0.0 
 
PWD := $(shell pwd) 
 
.PHONY: all 
 
all: clean build 
 
build: 
        docker run -ti --rm -v $(PWD):/app node:$(NODE_VERSION) /app/compile.sh $(APP_NAME) $(NAME) 
        echo 'FROM scratch\nADD build.tar.gz /\nCMD ["/app/$(NAME)"]' > Dockerfile 
        docker build -t philipz/$(NAME) . 
 
clean: clean-exe clean-build 
 
clean-docker: 
        docker rmi philipz/$(NAME) 
clean-exe: 
        sudo rm -f $(NAME) 
clean-build: 
        sudo rm -rf build  
        sudo rm -f build.tar.gz
```

前面三行定義變數，可依需求修改， *NAME* 是這程式名稱，也是編譯出來的執行檔名稱，而 *APP_NAME* 是 Node.js 主程式名稱，*NODE_VERSION* 則是欲使用的 Node.js 版本，根據這版號當作 Docker 標籤(tag)來建置其映像檔。

最主要是 `docker run -ti --rm -v $(PWD):/app node:$(NODE_VERSION) /app/compile.sh $(APP_NAME) $(NAME)` ，啟動官方 node 映像檔，並掛載目前所在目錄到容器的 */app* 目錄，再執行 *compile.sh* 這 bash script 程式。下一步驟便介紹 *compile.sh* 的內容。

#How it works? compile.sh
## compile.sh
而 *compile.sh* script 程式，內容如下：
``` 
#!/bin/bash 
cd /app 
npm install 
npm install -g enclose 
enclose --loglevel info -o ./$2 ./$1 
./copy_ldd.sh $2 build 
cd build && tar zcf build.tar.gz * && mv build.tar.gz ../ && cd /app && rm -rf build 
exit 
```

單純只是執行 npm 及 [enclose](http://enclosejs.com/) 這套專門用來將 Node.js 編譯成執行檔的工具。 `npm install -g enclose ` 安裝好 enclose ，便用它來輸出成 $2 參數二， $(NAME) 執行檔，而 $1 參數一，則為 $(APP_NAME) Node.js 主程式名稱。

#How it works? copy_ldd.sh
##copy_ldd.sh 
再下來執行 `copy_ldd.sh` 來分析編譯好的執行檔，由於 `copy_ldd.sh` 檔案內容較長，請自行研讀，主要目的就是將編譯好的 Node.js 程式執行檔(*minimal *)，用 **ldd** 指令分析其有關的函示庫，把相關函式庫，放到 build 目錄中，讓 `compile.sh` 可以壓縮打包成 tar.gz 檔案，便於讓 Dockerfile 以 `ADD` 指令建置出 Docker 映像檔。  
##參考文件 
[Export a linux binary with its lib dependencies to a chroot or initramfs, …](http://www.metashock.de/2012/11/export-binary-with-lib-dependencies/)

#SUMMARY
透過這個範例，就可理解 Docker 映像檔，其實只是一個符合程式所需的檔案系統結構，也可視為執行環境所需的檔案系統。而 Docekr 容器本身應該以單一個 **process** 來區分，而非是一整個 VM 的角度來看待容器，而這最小化的 Node.js 程式映像檔，就只是提供必要的相關函示庫（放置於所需的目錄位置）和其程式，未包含其他不必要的檔案和作業系統的指令，所以這最小化映像檔，也無法使用 `docker run -ti philipz/minimal bash`{{execute}} 登入到容器之中。

希望這個範例，能讓各位更理解容器的運作。
