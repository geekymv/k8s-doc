### Docker hub

https://hub.docker.com/
```shell
交互式方式运行
docker run --name tomcat1 -it -p 8081:8080 tomcat:8.5.49

ctrl + p + q 从交互式中出来

守护进程方式运行
dokcer run --name tomcat2 -d -p 8082:8080 tomcat:8.5.49

查看帮助
docker run --help

查看镜像分层
docker history 

管理docker命令
docker system --help

查看docker磁盘使用情况
docker system df 
```


### Dockerfile 指令
https://docs.docker.com/engine/reference/builder/
```shell
FROM 继承基础镜像
MAINTAINER #镜像制作者信息，一般是姓名和邮箱，该指令官方已不建议使用，而是使用LABEL代替
LABEL # <key>=<value> <key>=<value> ... 通过 docker inspect 可以查看 LABEL 和 MAINTAINER 信息
ENV <key> <value># 配置环境变量，这些环境变量可以被RUN指令使用，容器运行起来之后，也可以在容器中获取这些环境变量
ENV <key1>=<value1> <key2>=<value2> # 多个环境变量

ADD 复制文件到容器
COPY 复制文件到容器

WORKDIR # 设置容器的工作目录，容器打开后默认进入的目录，

RUN <command> # 用来执行shell命令，docker build 执行过程中，会使用 shell 运行指定的 command
RUN ["EXECUTABLE","PARAM1","PARAM2"] # docker build 执行过程中，会调用第一个参数指定的应用程序运行，并使用后面的参数作为应用程序的运行参数

CMD 启动容器默认执行的命令
CMD <command> shell 命令
CMD ["EXECUTABLE","PARAM1","PARAM2"] 

ENTRYPOINT 启动容器真正执行的命令

EXPOSE # 指定容器准备对外暴露端口号，但该端口号并不会对外真正的暴露，只有在执行 docker run 命令时使用 -p 来指定要真正暴露的端口号

VOLUME 创建挂载点

USER 容器使用的用户
```

注意：
- CMD 和 ENTRYPOINT 同时存在时，CMD 是作为 ENTRYPOINT 的参数
- ADD 复制文件会解压，COPY不会解压

#### 将C语言编写的程序构建镜像
hello-world 的 Dockerfile https://hub.docker.com/_/hello-world
https://github.com/docker-library/hello-world/blob/3fb6ebca4163bf5b9cc496ac3e8f11cb1e754aee/amd64/hello-world/Dockerfile

yum install -y gcc gcc-c++ glibc-static

```c
#include<stdio.h>
void main()
{
  printf("hello\n");
}
```
gcc --static -o hello hello.c

vi Dockerfile
```shell
FROM scratch
COPY hello /
CMD ["/hello"]
```
docker build -t hello:v0.1 .

docker run --name hello1 hello:v0.1

#### 在CentOS7的基础上构建自己的镜像
docker run --name mycentos -it centos:7

mkdir centos7
cd centos7
vi Dockerfile
```shell
FROM centos:7
LABEL \
    auther:"geekymv" \
    email:"ym2011678@foxmail.com" \
    version="1.0" \
    description="this is a custom centos7 image"
ENV WORK_HOME=/opt/app
RUN mkdir -p "${WORK_HOME}"    
WORKDIR ${WORK_HOME}

RUN yum install -y wget vim net-tools

CMD ["/bin/bash"]
```
docker build -t mycentos:1.0 .


/var/lib/doker

### 制作Dockerfile

mkdir dockerfiles

cd dockerfiles

- 创建 Dockerfile 文件

  vi Dockerfile

  ```shell
  FROM centos:7
  LABEL maintainer="my centos"
  LABEL test=dockerfile
  RUN mkdir /opt/geekymv
  CMD ["sh", "-c", "echo 1"]
  ```

  - 制作镜像
    docker build -t mycentos:7 .
    或
    docker build -t mycentos:7 -f Dockerfile .

  - 运行镜像（--rm运行完后删除）
    docker run -it --rm mycentos:7
    docker run -it --rm mycentos:7 bash （覆盖CMD）

    cd /opt/geekymv

  - 查看镜像
    docker images

- 验证CMD 和 ENTRYPOINT 同时存在时，CMD 是作为 ENTRYPOINT 的参数
  vi Dockerfile2
  ```shell
  FROM centos:7
  LABEL maintainer="my centos"
  LABEL test=dockerfile
  ENTRYPOINT ["echo"]
  CMD ["hi"]
  ```
  docker build -t epcentos:7 -f Dockerfile2 .
  运行容器
  docker run -it --rm epcentos:7
  指定参数
  docker run -it --rm epcentos:7 hello

- ENV
  vi Dockerfile3
  ```shell
  FROM centos:7
  LABEL maintainer="my centos"
  LABEL test=dockerfile
  ENV k1 v1
  ENV k2 v2
  CMD echo "${k1} ${k2}"
  ```
  docker build -t envcentos:7 -f Dockerfile3 .
  运行容器
  docker run -it --rm envcentos:7

- ADD 和 COPY
  tar -cvf hello.tar.gz hello.txt
  vi Dockerfile4
  ```shell
  FROM centos:7
  LABEL maintainer="my centos"
  LABEL test=dockerfile
  RUN mkdir /opt/geekymv
  COPY ./hello.tar.gz /opt/geekymv
  ```

  docker build -t accentos:7 -f Dockerfile4 .
  运行容器
  docker run -it --rm accentos:7 bash
  ls /opt/geekymv
- WORKDIR
  vi Dockerfile5
  ```shell
  FROM centos:7
  LABEL maintainer="my centos"
  LABEL test=dockerfile
  RUN mkdir /opt/geekymv
  WORKDIR /opt/geekymv
  CMD pwd
  ```
  docker build -t wdcentos:7 -f Dockerfile5 .
  运行容器
  docker run -it --rm wdcentos:7

- USER
  vi Dockerfile6
  ```shell
  FROM centos:7
  LABEL maintainer="my centos"
  LABEL test=dockerfile
  
  RUN useradd -ms /bin/bash geekymv
  USER geekymv
  WORKDIR /usr/local/bin/geekymv
  ```
  docker build -t usercentos:7 -f Dockerfile6 .
  运行容器
  docker run -it --rm  usercentos:7
  pwd
- 

- 

- 

### Docker compose
https://docs.docker.com/compose/
```shell
# CentOS 安装 https://docs.docker.com/compose/install/linux/#install-using-the-repository
sudo yum update
sudo yum install docker-compose-plugin

# 查看版本
docker compose version
```



#### 安装GitLab

```shell
安装文档
https://docs.gitlab.com/ee/install/docker.html#install-gitlab-using-docker-compose

mkdir -p /usr/local/docker/gitlab_docker
cd /usr/local/docker/gitlab_docker

# 创建 docker-compose.yml
vi docker-compose.yml

version: '3.6'
services:
  web:
    image: 'gitlab/gitlab-ee:latest'
    container_name: gitlab
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://192.168.56.101:8929'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ports:
      - '8929:8929'
      - '2224:22'
    volumes:
      - '$GITLAB_HOME/config:/etc/gitlab'
      - '$GITLAB_HOME/logs:/var/log/gitlab'
      - '$GITLAB_HOME/data:/var/opt/gitlab'
    shm_size: '256m'


# 启动
export GITLAB_HOME=/usr/local/docker/gitlab_docker

docker compose up -d

查看密码
sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password

```
### 安装MySQL
```md
docker run --name mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-v /usr/local/mysql/data:/var/lib/mysql \
-v /usr/local/mysql/log:/var/log/mysql \
-v /usr/local/mysql/conf:/etc/mysql/conf.d \
-d -p 3306:3306 \
mysql:5.7
```
增加配置文件
```md
cd /usr/local/mysql/conf
vi my.cnf
配置如下：
[client]
default_character_set=utf8mb4
[mysql]
default_character_set=utf8mb4
[mysqld]
collation_server=utf8mb4_general_ci
character_set_server=utf8mb4
```
重启MySQL容器
```md
docker restart mysql
```




#### VirtualBox 扩容 /dev/mapper/centos-root

https://blog.0xzhang.com/posts/VirtualBox%E8%99%9A%E6%8B%9F%E6%9C%BACentOS7%E7%A3%81%E7%9B%98%E6%89%A9%E5%AE%B9.html



