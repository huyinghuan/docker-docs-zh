## Docker 网络详解之概览

-----------

该系列文章使用的docker版本: `Docker version 18.09.5, build e8ff056dbc`

本文适合读者:  刚接触docker的新手，并已经掌握基本的使用，如 `docker container ls`, `docker container rm`等。

docker 具有5种网络驱动。分别是: `bridge`, `overlay`, `host`, `macvlan`, `none`

### bridge

`bridge` 为docker使用的默认网络模式，也是经常使用的网络模式。

如果你创建`container`时，没有通过 `--network`指定自定义网络，那么将docker将为容器创建`bridge`网络驱动。

如果在同一台物理机上，你运行在容器`container`的`应用`或者`服务` 需要同其他容器里面的`服务`进行通信，(如:你的应用需要访问一个`container`里面的`mysql`或者`redis`),那么你可以使用`bridge`模式。 

`bridge`模式可以提供了一个隔离网络，使运行的容器不干扰物理机网络。也可以使不同服务的网络隔离开，使其相互不干扰。

以下情况，要手动指定隔离网络网段：

    如果你的物理机有多个网络，如办公网络，内部vpn网络等网络时，可能需要手动设置或指定`bridge`的网段
    
    否则自动创建的随机隔离网络网段 可能导致物理机的网络出现问题。



### host

`host`意味着没有网络隔离，将直接使用物理机的网络。如，你的服务监听的是80端口，那么你的物理机的80端口就会被该服务进行占用。`host`模式仅在 `docker`版本`17.06`以后才可以使用，也仅支持`linux`系统，不支持`mac`,`windows`以及`windows server`的`docker EE`企业版。

如果你是`mac`或者 `windows`想要实验`host`模式，那么好的办法是通过`linux`虚拟机(如，virtualbox)或者 [docker-machine](https://docs.docker.com/machine/)创建虚拟机


### overlay

一般用于`docker`集群。`overlay`网络将多个Docker守护程序连接在一起，并使群集[不同物理机上的docker进程]服务能够相互通信。`docker`集群介绍见[docker swarm](https://docs.docker.com/engine/swarm/) 

### macvlan

macvlan 模式【不常用】，是给应用或服务运行的`container`生成一个mac地址，将`container`模拟成一台同该宿主物理机同一网络的一台'物理机'。对于需要连接到物理网络的传统应用程序使用macvlan驱动是最佳选择

### none

none的意义很明显，就是完全禁止以上网络。通常与自定义网络驱动程序一起使用。


### Network plugins

也可以安装和使用第三方插件。 插件可以在[docker hub](https://hub.docker.com/search/?category=network&q=&type=plugin) 找到