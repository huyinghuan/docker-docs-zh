# Docker 网络之 overlay 模式

------------

overlay网络用于在多个Docker守护程序主机之间创建分布式网络，一般用于 docker `swarm`集群。

ps: 如有错误之处敬请谅解，如有时间，可以在issue中指出。

## swarm初始化

在初始化 swarm或者加入swarm集群时，会自动在主机上创建两个网络:

1. 一个`overlay`网络，名称叫`ingress`
2. 一个`bridge` 网络，名称叫`docker_gwbridge`

### ingress

`ingress` 控制处理与集群相关的服务和数据流量，如集群流量的负载均衡。在创建集群时，不管有没有指定自定义的`overlay`网络，
服务都会连接到此网络.  【TODO】`ingress`不提供服务发现功能。【两个不同的service不能通过service name进行流量通信】

### docker_gwbridge

`docker_gwbridge` 用于连接处理各个加入了swarm集群的docker守护进程的网络流量。

```
#查看 docker 网络，没有 ingress 和 docker_gwbridge网络
docker network ls
#初始化 swarm 网络 eth1为网卡名称，根据实际情况选择
docker swarm init  --advertise-addr eth1
#增加了ingress 和 docker_gwbridge
docker network ls
```

### 注意

为了在一台机器上模式测试`swarm`网络,使用了`docker-machine`工具创建具有`docker`环境的虚拟机。【使用`docker-machine`建议内存大小在8G以上，标配16G,最好32G】

`docker-machine`需要本机上装有`virtualbox`. 关于`virtualbox`下载安装见: https://www.virtualbox.org/wiki/Downloads

更多`docker-machine`相关信息见: https://docs.docker.com/machine/ 

如果你有多台物理机一样的可以进行测试。
注意事项如下:

1. 打开 TCP端口 2377, 该端口用于集群的管理通信
2. 打开 TCP和UDP端口 7946，该端口用于节点通信
3. 打开UDP端口 4789， 该端口用于`overlay`的网络流量 

## swarn集群搭建

上面的内容中，已经在当前主机初始化了一个`swarm`集群。现在用`docker-machine`从头开始模拟具有两个节点的swarm集群。
更多`swarm`介绍请阅读 https://docs.docker.com/engine/swarm/

如果没有翻墙的话，这一步很慢，因为需要从github下载一个iso系统镜像，<br>
可以更加运行时终端提示，将相关镜像用迅雷下载到相关目录中去。<br>
创建一个名为 master的虚拟机

```
docker-machine create -d "virtualbox" master
```

连接到创建完成的虚拟机 master 
```
docker-machine ssh master
```
1.初始化swarm集群
```
docker swarm init --advertise-addr eth1
```
2.初始化完成后，master节点就变成了集群的管理节点，管理节点可以查看当前集群节点列表
```
docker node ls
```
3.查看节点以非管理节点身份加入集群的命令 <br>
【也做作为管理候选节点的身份加入集群， 如果主节点挂了，那么候选节点会自动成功为主节点，进行节点服务管理,
相关命令为 `docker swarm join-token manager`】
```
docker swarm join-token worker
```

复制提示，类似:
```
 docker swarm join --token SWMTKN-1-53jepeqm8u868bc2lqob0po1kup1j8993edtqs4ythxtq7pmrh-9cg0o2atvfu8d5ykhhffv06t6 192.168.99.104:2377
```
退出当前虚拟机
```
exit
```
创建一个名为 node01 的虚拟机
```
docker-machine create -d "virtualbox" node01
```
登录虚拟机node01
```
docker-machine ssh node01
```
4.加入集群，粘贴刚才通过`docker swarm join-token worker` 得到的命令
```
docker swarm join --token SWMTKN-1-53jepeqm8u868bc2lqob0po1kup1j8993edtqs4ythxtq7pmrh-9cg0o2atvfu8d5ykhhffv06t6 192.168.99.104:2377
```
查看下当前docker网络,发现有默认的ingress和docker_gwbridge网络生成
```
docker network ls
```
退出当前虚拟机
```
exit
```
重新连接到master
```
docker-machine ssh master
```
5.查看集群节点数量
```
docker node ls
```
查看ingress网络状态，可以看到各个节点的数量
```
docker network inspect ingress -f "{{.Peers}}"
```
如果有两台物理机，那么执行上面的`1,2,3,4,5`点即可<br>
以上介绍了`swarm`集群的默认网络。

上面已经创建了具有两个节点的集群，那么接下来，将部署具体的服务到集群中，测试使用`overlay`网络

## swarm 集群服务部署

以 `huyinghuan/helloworld`和 `huyinghuan/gethelloworld`为例进行网络说明

具体源码见 https://github.com/huyinghuan/docker-docs-resource

在master节点内执行【第一次执行可能有点慢，因为需要在每个节点内下载docker image】
```
docker service create --mode replicated --replicas 2  --name helloworld -p 8080:8080 --endpoint-mode vip huyinghuan/helloworld:latest
```
重复执行多次，可以看到来自不同的节点的响应，可以看到负载均衡模式
```
curl -w "\n" localhost:8080
```

## 自定义overlay网络

```
docker network create -d overlay helloNet
```
也可以使用其他一些参数指定子网
```
docker network create -d overlay --subnet 172.19.0.1/24 --gateway 172.19.0.1 helloNet
```

### 服务发现 【 service discovery 】
先启动一个 service 看下网络情况

```
docker service create  --name helloworld -p 8080:8080 huyinghuan/helloworld:latest
curl -w "\n" http://localhost:8080
```
可以看到响应结果中有两个`ip`地址，其中一个属于`ingress`，一个属于`docker_gwbridge`

再启动，测试是否具有`bridge`里面类似的自动`dns`发现功能
```
docker service create  --name gethello -e SERVER_TARGET=http://helloworld:8080  -p 8090:8080 huyinghuan/gethelloworld:latest
curl -w "\n" http://localhost:8090
```
可以看到结果，无法解析`helloworld`.

创建一个测试`overlay`网络

```
docker network create -d overlay --subnet 172.19.0.1/24 --gateway 172.19.0.1 helloTest
```

将两个服务`helloworld`和`gethello` 附加到`helloTest`网络上


```
docker service update --network-add helloTest helloworld
docker service update --network-add helloTest gethello
```
再次测试

```
curl -w "\n" http://localhost:8090
```

可以看到 `gethello2`已经发现了`helloworld`服务了

接着尝试下 独立的容器是否能发现服务

```
docker run --rm -d -p 9000:8080 -e SERVER_TARGET=http://helloworld:8080 --network helloTest --name standlone huyinghuan/gethelloworld:latest
```
可以看到没有权限附加到`helloTest`网络，如果需要使独立容器使用`overlay`网络那么需要在创建自定义网络时 使用`--attachable` 标志

创建一个`attachable`的`overlay`网络。

```
docker network create -d overlay --subnet 172.19.1.1/24 --gateway 172.19.1.1  --attachable helloTest2
```

附加`helloworld`到此网络

```
docker service update --network-add helloTest2 helloworld
```

再次运行独立容器，这次使用`helloTest2`网络

```
docker run --rm -d -p 9000:8080 -e SERVER_TARGET=http://helloworld:8080 --network helloTest2 --name standlone huyinghuan/gethelloworld:latest
```
成功运行,检查结果:
```
curl -w "\n" http://localhost:9000
```

### 清空测试用例

```
docker service rm  helloworld
docker service rm  gethello
docker container rm -f standlone
docker network prune
```

### 负载均衡之 VIP 

在`swarm`集群下创建服务，网络默认使用`VIP`模式, 创建服务时会生成一个虚拟IP地址，该IP会在所有节点轮流使用。

看下实例:

```
docker network create -d overlay --subnet 172.19.1.1/24 --gateway 172.19.1.1 --attachable helloNet
docker service create --name helloworld --network helloNet --replicas 2  huyinghuan/helloworld:latest
```

查看`helloworld`的`VIP`

```
docker service inspect helloworld -f "{{json .Endpoint}}"
```

可以看到一个`172.19.1.x`的虚拟IP，在本测试机上，显示的是`172.19.1.2/24`
查看两个服务副本的ID，
```
docker service ps -q helloworld
```
在本测试机上，显示的是 `ynuf7af5oiav9l102uxup0uny`和`dk1v4nlggaupyvqpe7j0w9fpk`

查看网络情况:

```
docker inspect ynuf7af5oiav9l102uxup0uny -f {{.NetworksAttachments}}
docker inspect dk1v4nlggaupyvqpe7j0w9fpk -f {{.NetworksAttachments}}
```
本测试机上可以看到`172.19.1.3/24`和`172.19.1.4/24`

重复运行,查看IP情况

```
docker run --network=testisolation --rm --network helloNet amouat/network-utils:latest curl -s -w "\n" "http://172.19.1.2:8080"
```

看下dns解析情况

```
docker run --network=testisolation --rm --network helloNet amouat/network-utils:latest dig helloworld
```

可以看到 dns将 `helloworld`总是解析到了VIP `172.19.1.2`上。

所以负载均衡流程是

```
                                             --> node 1                              
client -访问-> server name --> dns -解析-> vip 
                                             --> node 2
```

或者
```

                --> node 1                              
client -> vip 
                --> node 2
```

所以通过 `service name`或者 vip都可以进行负载均衡

清除实验代码:

```
docker service rm helloworld
```

### 负载均衡之 dnsrr 

当服务以 `dnsrr` 运行时， 相对于 `vip`模式， 将会缺少`vip`环节， dns直接将域名交替继续到不同节点上，如下:

```
                                --> node 1                              
client -> server name --> dns 
                                --> node 2
```

修改运行模式通过参数`--endpoint-mode`,以实例进行验证：

创建服务
```
docker service create --endpoint-mode dnsrr --name helloworld --network helloNet --replicas 2  huyinghuan/helloworld:latest
```
查看网络
```
docker service inspect helloworld -f "{{json .Endpoint}}"
```
发现没有`vip`了，通过下面命令可以看到 `dnsrr`模式

```
docker service inspect helloworld -f "{{json .Spec.EndpointSpec}}"
```

查看服务详情

```
docker service ps -q helloworld
```
获取container id ,在本测试机上是 `qbzciklktju9`和`uva6r7w8szca`
查看网络情况，得到ip地址为`172.19.1.26/24`和`172.19.1.27/24`

```
docker inspect qbzciklktju9 -f {{.NetworksAttachments}}
docker inspect uva6r7w8szca -f {{.NetworksAttachments}}
```

查看dns解析

```
docker run --network=testisolation --rm --network helloNet amouat/network-utils:latest dig helloworld
```

可以得到两个解析地址.