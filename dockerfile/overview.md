Dockerfile
---------

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

```
RUN <command> (命令运行在shell中) linux默认是: /bin/sh -c  windows默认是: cmd /S /C 
RUN ["executable", "param1", "param2"] 
```
`RUN` 执行操作后造成的影响(文件修改等)，将会保存在镜像中，影响Dockerfile 接下来的步骤。

比如 创建一个配置

```
...
RUN echo "{\"db\":\"127.0.0.1\"}" > /config.json
RUN xxx -c config.json
```

那么xxx 就可以使用`该文件`


## CMD

CMD指令用于运行镜像包含的软件以及配置参数

```
CMD ["nginx", "-s", "start"]
```

虽然`RUN`和`CMD`都可以执行命令，但是不同支持在于：

`RUN` 的命令在镜像构建时执行，这个操作将保留在镜像的提交历史中， 而`CMD`仅在容器运行时执行，它不会包含在提交历史中。
`CMD`用来配置容器启动完成后，运行的服务。


`CMD` 也可以给容器设置启动参数的，和 `ENTRYPOINT`不同的是，`CMD`可以在运行时被替换覆盖，而`ENTERYPOINT`不能被覆盖。
`CMD`可以作为 `ENTRYPOINT`的补充。

在有一个 `Dockerfile`里面，至少包含`CMD`和`ENTRYPOINT`中的一个，如果两个都没有，在镜像构建时会报错。

```
CMD ["executable","param1","param2"] (exec form, this is the preferred form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
CMD command param1 param2 (shell form)
```

## ENTRYPOINT

配置容器启动后运行的程序。

```
ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred) 推荐使用
ENTRYPOINT command param1 param2 (shell form)
```

如果你的容器是作为一个工具使用【启动运行某些操作后直接退出删除，如es6的编译等】那么需要使用 `ENTRYPOINT`.

`docker`运行时，如果指定了容器参数，那么这个参数将加在 `ENTRYPOINT`可配置项后面。

注意：如果想要在运行时覆盖ENTRYPOINT配置的参数，那么可以将需要覆盖的参数写入到`CMD`指令众.

`ENTRYPOINT`和`CMD`同时存在时，`CMD`的配置项加作为参数加在`ENTRYPOINT`配置项后面。

来看例子:

Dockerfile:

```Dockerfile
FROM huyinghuan/hello-tool:latest
ENTRYPOINT ["/tool/app"]
```
构建

```
docker build -t t:1 .
```
执行

```
docker run --rm t:1
```
可以看到响应为`I'm a tool, I get this args:`，增一个容器参数

```
docker run --rm t:1 -n "test"
```

响应输出`I'm a test, I get this args:`

更新`Dockerfile`

```
FROM huyinghuan/hello-tool:latest
ENTRYPOINT ["/tool/app", "-n", "t2"],
```
构建
```
docker build -t t:2 .
```
分别运行下面的命令
```
docker run --rm t:2
docker run --rm t:2 -t
```
可以看到第二个执行的实际命令是 `/toot/app -n t2 -t`

更新 `Dockerfile`

```
FROM huyinghuan/hello-tool:latest
ENTRYPOINT ["/tool/app", "-n", "t3"],
CMD ["-t"]
```

构建

```
docker build -t t:3 .
```

分别运行下面的命令
```
docker run --rm t:3
docker run --rm t:3 -m "haha convery"
```
可以看到 `-t`参数已经被 `-m`参数覆盖了

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

- 如果`src`是个归档文件（压缩文件, `tar`），则docker会自动帮解压。

## COPY

copy和add功能的简化版，但是只支持文件拷贝。除非确定要拷贝url,解压文件之类的功能，那么才使用`ADD`,否则推荐`COPY`

```
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

`copy`参数要求如下：

- `src`必须是基于`Dockerfile`的相对路径而且必须是子目录，不能是`../`，`dest`必须时绝对路径
- 如果`src`是目录，则复制目录的全部内容，包括文件系统元数据,目录本身不是复制，只会复制它的子文件和子文件夹
- 如果直接或由于使用通配符指定了多个`src`资源，则`dest`必须是目录，并且必须以斜杠/结尾。
- 如果`dest`不以尾部斜杠结束，则将其视为常规文件，`src`的内容将写入`dest`
- 如果`dest`不存在，则会在其路径中创建所有缺少的目录


## VOLUME

默认情况下，在容器内发生的写操作会在容器移除后而消除，要想保留容器写记录，那么需要把读写操作放到`VOLUME`指定的目录中,或者在运行时通过指定 `-v`参数，设置挂载。

`VOLUME`配置会在docker的主机中生成一个随机目录，进行写操作的保存。

`VOLUME`的声明必须放在`Dockerfile`所有文件更改操作的后面，否在在`VOLUME`的声明之后的所有文件操作将会被舍弃。

```
FROM ubuntu:latest
RUN mkdir /data
WORKDIR /data
RUN touch hello  #保存
VOLUME [ "/data" ]
RUN touch world #舍弃
ENTRYPOINT [ "ls", "-al" ]
```

container运行后，可以通过

```
docker container inspect CONTAINER_ID -f "{{json .Mounts}}"
```
查看`VOLUMN`配置详情

## USER

```
USER <user>[:<group>] or
USER <UID>[:<GID>]
```

设置容器运行时所用的用户组。


## WORKDIR

