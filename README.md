# docker-docs-zh
docker 中文笔记

## 目录

### 网络

- 1.  [概览](https://github.com/huyinghuan/docker-docs-zh/blob/master/network/overview.md)
  - 1.1 [Bridge](https://github.com/huyinghuan/docker-docs-zh/blob/master/network/bridge.md)
  - 1.2 [Host And MacVlan](https://github.com/huyinghuan/docker-docs-zh/blob/master/network/HostAndMacVlan.md)
  - 1.3 [Overlay](https://github.com/huyinghuan/docker-docs-zh/blob/master/network/overlay.md)
  - 1.4 [iptables](https://github.com/huyinghuan/docker-docs-zh/blob/master/network/iptables.md)
  - 1.5 [容器访问宿主服务](#container-connect-host)
  
  
  
#### <a href="#container-connect-host"></a> 容器访问宿主网络
具体见：
https://stackoverflow.com/questions/24319662/from-inside-of-a-docker-container-how-do-i-connect-to-the-localhost-of-the-mach

总结就是：
如果在 mac或者windows 并且使用的版本在 18.03 以上,那么可以使用 `host.docker.internal` 来表示宿主机。
linux下面，可能需要等到 docker 20.04 版本才会出现。 目前使用 docker0 网卡的ip地址 【存疑，docker-compose里面好像访问不到，直接用docker run可以访问到？】
