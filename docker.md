#### Docker hub

```shell
https://hub.docker.com/

```



#### Dockerfile 指令

```shell
FROM 继承基础镜像
MAINTAINER 镜像制作者信息
RUN 用来执行shell命令
EXPOSE 暴露端口号
CMD 启动容器默认执行的命令
ENTRYPOINT 启动容器真正执行的命令
VOLUME 创建挂载点
ENV 配置环境变量
ADD 复制文件到容器
COPY 复制文件到容器
WORKDIR 设置容器的工作目录
USER 容器使用的用户
```

注意：

- CMD 和 ENTRYPOINT 同时存在时，CMD 是作为 ENTRYPOINT 的参数

- ADD 复制文件会解压，COPY不会解压

  

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



#### VirtualBox 扩容 /dev/mapper/centos-root

https://blog.0xzhang.com/posts/VirtualBox%E8%99%9A%E6%8B%9F%E6%9C%BACentOS7%E7%A3%81%E7%9B%98%E6%89%A9%E5%AE%B9.html



