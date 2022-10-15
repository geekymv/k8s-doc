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

CMD 和 ENTRYPOINT 同时存在时，CMD 作为 ENTRYPOINT 的参数。



### 制作Dockerfile

mkdir dockerfiles

cd dockerfiles

vi Dockerfile

```shell
FROM centos:7
LABEL maintainer="my centos"
LABEL test=dockerfile
RUN useradd geekymv
RUN mkdir /opt/geekymv
CMD ["sh", "-c", "echo 1"]
```

- 制作镜像

docker build -t mycentos:7 .

或

docker build -t mycentos:7 -f dockerfiles/Dockerfile

- 运行镜像（--rm运行完后删除）

docker run -it --rm mycentos:7

docker run -it --rm mycentos:7 bash （覆盖CMD）

查看镜像

docker images

