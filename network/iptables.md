## docker iptables

默认情况下  docker 暴露出来的端口，是绑定在 0.0.0.0 上的， 由于docker会修改`iptables`的 `FORWARD`配置， 导致宿主机自带的防火墙无法管理 docker暴露出来的端口，这些端口都是对公网可见的。


要屏蔽这些端口，需要进行iptables配置。

主要的参考文章有：
1. iptables 基本手册：https://linux.die.net/man/8/iptables
2. https://docs.docker.com/network/iptables/
3. https://serverfault.com/questions/704643/steps-for-limiting-outside-connections-to-docker-container-with-iptables


如果docker内无法访问宿主机

```
iptables -I INPUT -i docker0 -j ACCEPT
```




追踪 `iptables` 流程：`https://www.opsist.com/blog/2015/08/11/how-do-i-see-what-iptables-is-doing.html`

```
iptables -t raw -A PREROUTING -s 追踪指定来源的ip -p tcp -j TRACE
```

设置 追踪指定来源的ip 编译日志过滤与分析。

拒绝外网访问docker端口
```
iptables -R DOCKER-USER 2 -i bond4 -o docker0  -p tcp -m conntrack --ctorigdstport 8081 --ctdir ORIGINAL -j REJECT
```
