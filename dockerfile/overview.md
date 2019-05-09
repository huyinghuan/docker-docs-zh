# Dockerfile

docker image 的构建文件。 如果需要将一个服务运行在docker里面，那么就要把这个服务打包成`image`， `Dockerfile`就是服务打包一系列到命令集合。

## FROM
用法
```
FROM <image>[@<digest>] [AS <name>]
```

指定服务运行需要的一个带有运行时环境的系统镜像。


比如`java`应用需要`jvm`环境，`node`需要 `nodejs`的环境。

你可以基于基本linux系统镜像来搭建你需要到环境，比如`ubuntu`,`centos`等

或者使用官方发布的具有相关环境系统镜像。比如`nodejs`等。

官方推荐的最基本linux镜像为 `alpine`,因为它是很小[5MB] 并且是完整的linux发型版。

如
```
FROM ubuntu:latest
....
```
或者

```
FROM nodejs:latest
...
```

## Label

可以为镜像添加标签，以帮助按项目组织图像，记录许可信息，辅助自动化等。对于每个标签，添加以LABEL开头并带有一个或多个键值对的行。

```
FROM ubuntu:latest
LABEL LICENSE="MIT"
LABEL AUTHOR="HYH" EMAIL="ec.huyinghuan@gmail"
...
```

## RUN

在基础镜像中运行的命令.

```
FROM ubuntu:latest
LABEL LICENSE="MIT"
LABEL AUTHOR="HYH" EMAIL="ec.huyinghuan@gmail"
RUN apt-get update
RUN ap-get install nginx
...
```

以及在构建过程中，将先更新ubuntu的仓库信息，然后安装`nginx`软件


## CMD

CMD指令用于运行镜像包含的软件以及任何参数

```
CMD ["nginx", "-s", "start"]
```

虽然`RUN`和`CMD`都可以执行命令，但是不同支持在于：

`RUN` 的命令在镜像构建时执行，这个操作将保留在镜像的提交历史中， 而`CMD`仅在容器运行时执行，它不会包含在提交历史中。
`CMD`用来配置容器启动完成后，运行的服务。


## EXPOSE

EXPOSE指令通知Docker容器在运行时侦听指定的网络端口。您可以指定端口是侦听TCP还是UDP，如果未指定协议，则默认为TCP.
EXPOSE指令实际上不会发布端口。它在构建映像的人和运行容器的人之间起到一种文档的作用，告诉使用的人这个镜像里面的服务运行在哪些端口。

```
FROM ubuntu:latest
LABEL LICENSE="MIT"
LABEL AUTHOR="HYH" EMAIL="ec.huyinghuan@gmail"
RUN apt-get update
RUN ap-get install nginx
EXPOSE 80
#EXPOSE 80/tcp
#EXPOSE 80/udp
```

## ENV

配置镜像的默认环境变量，如果你服务需要环境变量中读取数据那可以通过`ENV`来配置。在 `docker run`时使用 `-e`来增加或者修改。

```
ENV <key> <value>
ENV <key>=<value> ...
```

## ADD

向镜像中添加文件

```
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

比如需要将静态文件添加到镜像中，作为服务的一部分就可以使用此指令

```
ADD static /var/www/test.com
```

`ADD`指令支持文件通配符，具体规则见 https://golang.org/pkg/path/filepath/#Match

### src规则
- `src`必须是基于`Dockerfile`的相对路径而且必须是子目录，不能是`../`，`dest`必须时绝对路径

- 如果`src`是URL且`dest`不以`/`结尾，则从URL下载文件并将其复制到`dest`。

- 如果`src`是一个URL，`dest`以`/`结尾，然后从URL推断文件名，文件下载到`dest/filename`

- 如果`src`是目录，则复制目录的全部内容，包括文件系统元数据,目录本身不是复制，只会复制它的子文件和子文件夹

## 