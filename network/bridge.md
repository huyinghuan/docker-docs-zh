# Docker 网络之 Bridge 模式

------------

以下就通过多个实例来说明bridge网络。

在一台未部署任何服务的docker主机上运行 `docker network ls`查看当前docker的网络列表

```
NETWORK ID          NAME                DRIVER              SCOPE
xxxxxxxxxxx1        bridge              bridge              local
xxxxxxxxxxx2        host                host                local
xxxxxxxxxxx3        none                null                local
```

其中 `none`使预留网络。 `host`是使用`host`模式时使用的预建网络。`bridge`是默认的桥接网络。该篇文章仅讨论 `bridge` 模式


## 默认桥接网络

通过`docker network inspect $NETWORKName`, 查看具体的网络详情。 这里 `$NETWORKName` 请使用实际的 network name,
如：查看上面的名为`bridge`的 网络则是 `docker network inspect bridge`,

得到如下信息：
```
[
    {
        "Name": "bridge",
        "Id": "xxxxx",
        "Created": "2019-04-29T08:05:35.348925866Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

根据以上信息，可以看到在当前这个物理机上，docker默认的`bridge`网段为`172.17.0.0/16`,【不同机器可能生成的默认网段不同】 当创建一个服务没有指定网络时，那么该服务就会随机使用这个网段的一个地址。 接下来验证一下:

```
#先拉一个nginx的镜像
docker image pull nginx:latest
#运行该镜像
docker run --name=test -d nginx:latest 

docker container ls
``` 
可以看到正在运行的服务：

```
16d0b12fa58e   nginx:latest  3 minutes ago   Up 3 minutes         test
```

这个时候看下docker网络 `docker network ls`,发现没有任何网络配置增加，看下`bridge`网络详情：`docker network inspect bridge`, 发现详情里面增加了类似信息：

```
"Containers": {
            "16d0b12fa58ee5e15f7851dc8203469c6bcb2035e1c2ba3f8b055b9cf2291423": {
                "Name": "test",
                "EndpointID": "42e03c7f0239433dcb8e3e6ef34f3b510d6da63dd7067f24014c88560d5acd3f",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
```

可以在这里看到 刚才创建的 `test`服务，使用了`172.17.0.2/16`网络地址，属于`bridge`默认网段。

继续创建一个`test2`的服务，继续查看网络相关信息。

```
docker run --name=test2 -d amouat/network-utils:latest /usr/bin/yes
docker network inspect bridge
```
    amouat/network-utils 是一个包含网络工具的镜像
    /usr/bin/yes 是由于该镜像没有服务运行，为了保持该镜像运行，启用该命令


可以看到增加的 `test2`服务使用了`172.17.0.3/16`地址。

使用一下命令：
```
docker exec test2 curl "http://172.17.0.2"
```
可以看到 nginx的响应消息。证明这两个镜像属于同一个可以互通的网络。


### 总结

未指定运行网络的应用容器使用默认桥接网络，在该桥接网络内，各个服务网络互通。


## 自定义bridge模式

### 创建自动分配网段的bridge

创建一个自定义bridge网络，名为`custom`

```
docker network create -d bridge custom
```

可以通过`docker network ls`查看到该网络已经生成，通过`docker network inspect custom` 可以看到该网络的详细信息。

该网络的网段是docker自动生成的。

### 创建自定义网段的bridge

如果有需要，可以手动指定`bridge`网段。该网段必须要是局域网网段,网段必须在这个范围内：

```
C类：192.168.0.0-192.168.255.255
B类：172.16.0.0-172.31.255.255
A类：10.0.0.0-10.255.255.255
```
以下命令生成了一个 `172.20.1.1-172.20.1.254` 可用地址为254个局域网网段


```
docker network create -d bridge --subnet 172.20.1.0/24 --gateway 172.20.1.1 custom2
```

创建一个使用`custom2`网络的服务。

```
docker run --name=custom-nginx --network=custom2 -d nginx:latest 
```

通过`docker inspect custom-nginx`,可以看到docker分配给该容器的网络地址。

可以通过 `--ip` 指定该容器在该网段的ip地址

```
#先移除正在运行的容器
docker rm -f custom-nginx 
#指定ip地址
docker run --name=custom-nginx --network=custom2  --ip=172.20.1.199 -d nginx:latest
#校验分配的地址 
docker inspect custom-nginx
```

### 不同bridge之间的网络是隔离的

使用不同bridge网络的容器无法相互访问。

```
docker network create -d bridge testisolation
docker network create -d bridge testisolation2
docker run --name=nginx --network=testisolation -d nginx:latest 
#使用不同的bridge网络报错，无法获取到nginx首页内容,链接超时
docker run --network=testisolation2 --rm amouat/network-utils:latest curl  -w "\n" -s --max-time 3 "http://$(docker inspect nginx -f {{.NetworkSettings.Networks.testisolation.IPAddress}})"
#使用相同的bridge网络，获取到nginx首页内容
docker run --network=testisolation --rm amouat/network-utils:latest curl  -w "\n" -s --max-time 3 "http://$(docker inspect nginx -f {{.NetworkSettings.Networks.testisolation.IPAddress}})"
```

解释:

```
命令 `docker inspect nginx -f {{.NetworkSettings.Networks.testisolation.IPAddress}}`
获取 container name为 nginx的容器信息，-f 将内容格式化，
获取指定字段`NetworkSettings.Networks.testisolation.IPAddress`的值。 即得到了 nginx容器的ip地址。
将该命令包裹在$() 即可将命令的结果作为参数传给curl命令
```

## 自定义bridge模式 和 默认bridge模式 区别

### 自定义桥接网络在容器之间提供自动DNS解析。

自定义`bridge`网络:

```
docker network create -d bridge testportopen
docker run --name=helloworld --network=testportopen -d huyinghuan/helloworld:latest 
#正常输出 hello docker
docker run --network=testportopen --rm amouat/network-utils:latest curl -w "\n" -s --max-time 3 "http://helloworld:8080"
```

最后的`curl`命令使用`helloworld`来访问正在运行的容器服务。而不是用IP，自定义`bridge`网络提供了DNS解析，将`container name`解析到了对应的容器ip，这个使用起来很方便，比如运行的http服务里面有mysql之类的依赖,可以将mysql的`container name`作为连接地址


默认的bridge网络并不直接提供DNS解析功能: 

```
docker run --name=helloworlddf -e SERVER_PORT=3306 -d huyinghuan/helloworld:latest 
#解析失败
docker run --rm amouat/network-utils:latest curl -w "\n" --max-time 3 http://helloworlddf:3306
```
目前还有一个方式可以提供解析功能，即使用`--link`参数，不过该选项以后会被删除弃用，因此不做说明

### 容器可以在运行中与自定义桥接网络进行连接和分离。

从以下列子可以看到，可以不停止容器而进行容器的网络连接和分离

```
docker network create -d bridge fly
docker network create -d bridge fly2
#启动时连接fly网络
docker run --name=helloworld --network=fly -d huyinghuan/helloworld:latest
#访问测试,预期正常
docker run --network=fly --rm amouat/network-utils:latest curl -w "\n" --max-time 3 http://helloworld:8080
#从网络fly中分离
docker network disconnect fly helloworld
#访问测试,预期失败
docker run --network=fly --rm amouat/network-utils:latest curl -w "\n" --max-time 3 http://helloworld:8080
#连接网络fly2
docker network connect fly2 helloworld
#访问测试,预期正常
docker run --network=fly2 --rm amouat/network-utils:latest curl -w "\n" --max-time 3 http://helloworld:8080
```

## 结束

以上的示例都是在`bridge`内部网络容器内进行， 如果你想要外部网络访问 `bridge`里面的应用服务，你需要在运行时使用`-p`选项将端口发布出来。如:

```
#helloworld 程序监听的时8080端口
docker run --name=helloworld -p 8080:8080 -d huyinghuan/helloworld:latest
curl -w "\n" http://localhost:8080
#如果你的8080端口已经被其他程序占用，可以指定将`8080`端口发布到主机的其他端口，如:
docker run --name=helloworld2 -p 9000:8080 -d huyinghuan/helloworld:latest
curl -w "\n" http://localhost:9000
```
