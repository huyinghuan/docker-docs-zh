# Docker 网络之 Host 模式
--------

在`Overview`已经介绍过`host`, 运行几个例子来看下实际效果.

```
docker run --name=helloworld --network=host -d huyinghuan/helloworld:latest
#查看端口状态，可以看到8080端口已经被占用了。
netstat -anpt
#访问下服务
curl -w "\n" http://localhost:8080
```

可以看到使用 host 模式,不需要`-p`导出端口，既可以在本机上使用

# Docker 网络之 macvlan 模式
--------
通常情况下不建议使用`macvlan`模式。

```
#首先看下网卡信息
ifconfig
#选择一个可以分配多个MAC地址的物理接口。请根据实际情况选用，这里选用eth1
docker network create -d macvlan --subnet=172.16.86.0/24 --gateway=172.16.86.1 -o parent=eth mactest
```

由于没有找到合适的程序，因此`macvlan`模式无法进行校验，本篇文章也到此为止。

更多相关信息可以参考: https://docs.docker.com/network/macvlan/