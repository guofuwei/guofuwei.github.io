---
title: iptables与docker网络
---

# iptables 与 docker 网络

## docker 默认网络

#### 创建 test1,test2 两个容器，加入 docker0 默认网络

```shell
docker run -itd --name test1 alpine
```

```shell
docker run -itd --name test2 alpine
```

查看 test1 网络

```shell
docker inspect test1 | jq '.[].NetworkSettings.Networks'
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 17.27.13.png"/>

默认网关是 172.17.0.1

同时这个 172.17.0.2 的 ip 不是和 test1 绑定的，并且不能通过`--ip`来绑定 ip 地址，也不能通过容器名来相互 ping 通

## docker 自定义网络

#### 创建自定义网络

```shell\
docker network create test_web -d bridge -o com.docker.network.bridge.name=test_web --subnet="172.18.0.0/16" --gateway="172.18.0.1"
```

#### 创建 test3，test4 容器加入网络

```shell
docker run -itd --name test3 --network test_web alpine
```

```shell
docker run -itd --name test4 --network test_web alpine
```

#### 现在有四个网络

```
docker ps
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 19.35.30.png"/>

```shell
brctl show
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 19.36.19.png"/>

#### 测试容器之间的网络互通性

```shell
# test1可以ping通test2
docker exec -it test1 ping 172.17.0.3
```

```shell
# test3可以ping通test4
docker exec -it test3 ping test4
```

```shell
# test1不能ping通test3
docker exec -it test1 ping 172.18.0.2
```

## iptables 完成 docker 网络之间的隔离性

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/20221029200229.png"/>

```shell
sudo iptables -t filter -nxvL
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.05.30.png"/>

所有的 FORWARD 链都必须先经过 DOCKER-USER，默认 DOCKER-USER 直接返回

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.06.54.png"/>

再经过 DOCKER-ISOLATION-STAGE-1

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.07.41.png"/>

在 DOCKER-ISOLATION-STAGE-1 链中，所有从**test_net 发往 test_net 以外的网卡，从 docker0 发往 docker0 以外的网卡** 都经过 DOCKER-ISOLATION-STAGE-2

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.07.47.png"/>

DOCKER-ISOLATION-STAGE-2 直接将发往 test_net 和 docker0 的包 DROP,完成容器隔离

**注意：docker 必须开启 ip_forward，如果关闭，docker 容器将无法上网**

```shell
sudo sysctl -w net.ipv4.ip_forward=0 # 关闭ip_forward
```

## iptables 完成 docker 容器的动态地址转换

```shell
sudo iptables -t nat -nvxL
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.14.17.png"/>

将从 docker 容器的发出的网络请求进行 MASQUERADE（动态 SNAT）

**注意：不需要对返回的包进行 DNAT，iptables 有 conntrack 连接追踪功能**

例如 `Container A` 通过 bridge 网络向 baidu.com 发起了 N 个连接，这时数据的处理流程如下：

- 例如 Container A 发出的数据包被 MASQUERADE 规则处理，将 src ip 替换成 eth0 的 ip，然后发送到物理网络 192.168.31.0/24。

* conntrack 系统记录此连接被 NAT 处理前后的状态信息，并将其状态设置为 NEW，表示这是新发起的一个连接

* 对端 baidu.com 返回数据包后，会首先到达 eth0 网卡

* conntrack 查表，发现返回数据包的连接已经记录在表中并且状态为 NEW，于是它将连接的状态修改为 ESTABLISHED，并且将 dst_ip 改为 172.17.0.2 然后发送出去（注意，这个和 tcp 的 ESTABLISHED 没任何关系）

* 经过路由匹配，数据包会进入到 docker0，然后匹配上 iptables 规则：`-t filter -A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`，数据直接被放行，数据经过 veth 后，最终进入到 `Container A` 中，交由容器的内核协议栈处理。

* 数据被 `Container A` 的内核协议栈发送到「发起连接的应用程序」。

## iptables 和 docker-proxy 共同完成容器端口映射

#### 先起一个 nginx 容器

```shell
docker run --name nginx-demo -itd --network test_net --ip 172.18.0.66 -p 8080:80 nginx:latest
```

#### 查看 iptables

```shell
sudo iptables -t nat -nxvL
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.26.01.png"/>

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.26.13.png"/>

iptables 对发往 172.18.0.66 的端口进行了 DNAT

#### docker-proxy 的存在

```shell
ps -aux | grep 'docker-proxy'
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.28.19.png"/>

当我们绕开 iptables 的 DNAT 规则时

```shell
sudo iptables -t nat -I DOCKER 1 -p tcp -j RETURN
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.30.20.png"/>

依然可以正常访问

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.30.57.png"/>

杀掉 docker-proxy 进程

```shell
sudo pkill docker-proxy
```

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.33.49.png"/>

无法正常访问

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/截屏2022-10-29 20.34.03.png"/>

## 总结

1.  docker 安装后会自动开启 ip_forward，默认桥接网络是 docker0
2.  docker 会在 filter 表 FORWARD 链中加入 DOCKER-USER，DOCKER-ISOLATION-STAGE-1,DOCKER-ISOLATION-STAGE-2，通过 DOCKER-ISOLATION-STAGE-1，DOCKER-ISOLATION-STAGE-2 完成容器之间的隔离
3.  每新建一个 docker network，docker 会在 nat 表 POSTROUTING 链中加入该网络的动态 SNAT（MASQUERADE），同时利用 conntrack 完成回应包的自动地址转换
4.  docker 容器和宿主机之间的端口映射是 iptables 和 docker-proxy 共同完成的。iptables 会在 nat 表的 PREROUTING 中自定义 DOCKER 链中加入 DNAT。docker-proxy 会为每个容器开启。
