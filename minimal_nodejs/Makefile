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
