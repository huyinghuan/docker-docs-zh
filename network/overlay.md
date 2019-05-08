# Docker 网络之 overlay 模式

------------

overlay网络用于在多个Docker守护程序主机之间创建分布式网络，一般用于 docker `swarm`集群。

## swarm初始化

在初始化 swarm时，会自动在主机上创建两个网络:

1. 一个`overlay`网络，名称叫`ingress`
2. 一个`bridge` 网络，名称叫`docker_gwbridge`

### ingress

`ingress` 控制处理与集群相关的服务和数据流量，如集群流量的负载均衡。如果在创建集群时，没有指定自定义的`overlay`网络，
那么将默认使用此网络

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

```
#如果没有翻墙的话，这一步很慢，因为需要从github下载一个iso系统镜像，
#可以更加运行时终端提示，将相关镜像用迅雷下载到相关目录中去。
#创建一个名为 master的虚拟机
docker-machine create -d "virtualbox" master
#连接到创建完成的虚拟机 master
docker-machine ssh master
#1.初始化swarm集群
docker swarm init --advertise-addr eth1
#2.初始化完成后，master节点就变成了集群的管理节点，管理节点可以查看当前集群节点列表
docker node ls
#3.查看节点以非管理节点身份加入集群的命令 
#【也做作为管理候选节点的身份加入集群， 如果主节点挂了，那么候选节点会自动成功为主节点，进行节点服务管理，】
#【相关命令为 docker swarm join-token manager】
docker swarm join-token worker
#复制提示，类似:
#
# docker swarm join --token SWMTKN-1-53jepeqm8u868bc2lqob0po1kup1j8993edtqs4ythxtq7pmrh-9cg0o2atvfu8d5ykhhffv06t6 192.168.99.104:2377
#
#
#退出当前虚拟机
exit
#创建一个名为 node01 的虚拟机
docker-machine create -d "virtualbox" node01
#登录虚拟机node01
docker-machine ssh node01
#4.加入集群，粘贴刚才通过 docker swarm join-token worker 得到的命令
docker swarm join --token SWMTKN-1-53jepeqm8u868bc2lqob0po1kup1j8993edtqs4ythxtq7pmrh-9cg0o2atvfu8d5ykhhffv06t6 192.168.99.104:2377
#查看下当前docker网络,发现有默认的ingress和docker_gwbridge网络生成
docker network ls
#退出当前虚拟机
exit
#重新连接到master
docker-machine ssh master
#5.查看集群节点数量
docker node ls
#查看ingress网络状态，可以看到各个节点的数量
docker network inspect ingress -f "{{.Peers}}"
```
如果有两台物理机，那么执行上面的`1,2,3,4,5`点即可

以上介绍了`swarm`集群的默认网络。

上面已经创建了具有两个节点的集群，那么接下来，将部署具体的服务到集群中，测试使用`overlay`网络

## swarm 集群服务部署

以 `huyinghuan/helloworld`为例进行网络说明，

```
#在master节点内执行
#第一次执行可能有点慢，因为需要在每个节点内下载docker image
docker service create --mode replicated --replicas 2  --name helloworld -p 8080:8080 --endpoint-mode vip huyinghuan/helloworld:latest
#重复执行多次，可以看到来自不同的节点的响应，可以看到负载均衡模式
curl -w "\n" localhost:8080
```

## 自定义overlay网络

```
docker network create -d overlay helloNet
#也可以使用其他一些参数指定子网
docker network create -d overlay --subnet 172.19.0.1/24 --gateway 172.19.0.1 helloNew
#更新服务
```