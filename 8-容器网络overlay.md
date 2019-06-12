
### 简介
为支持容器跨主机通信，docker提供overlay driver，使用户可以创建基于VxLAN的overlay网络。
VxLAN可将二层数据封装到UDP进行传输，VxLAN提供与VLAN相同的以太网二层服务，但是拥有更强的扩展性和灵活性。


### 实验环境描述

##### consul运行

```bash
# 容器方式运行Consul
[root@tk-tmp01:~]# docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap
```
web页面打开地址：http://tk-tmp01外网ip:8500


##### 修改节点host01，host02

增加内容--cluster-store=consul://172.24.152.190:8500 --cluster-advertise=eth0:2376
--cluster-store=consul://172.24.152.190:8500 指定consul的地址
--cluster-advertise=eth0:2376 告知consul自己的链接地址

```bash
# 在host01和host02修改docker daemon的配置文件
[root@host01 ~]# cat /etc/systemd/system/docker.service.d/10-machine.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock --storage-driver overlay2 --tlsverify --tlscacert /etc/docker/ca.pem --tlscert /etc/docker/server.pem --tlskey /etc/docker/server-key.pem --label provider=generic --cluster-store=consul://172.24.152.190:8500 --cluster-advertise=eth0:2376
Environment=
# 重启docker daemon
[root@host01 ~]# systemctl daemon-reload
[root@host01 ~]# systemctl restart docker.service

```

### 创建overlay网络

```bash
# 在host01创建
[root@host01 ~]# docker network create -d overlay ov_net1
cc6f2cb10d939c83456b09b47a59924c7e382492f5ff777333acce7a2125f202

# 在host01查看
[root@host01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
106070dae414        bridge              bridge              local
d8b4cf5bd66e        host                host                local
8dc7aa13590e        none                null                local
cc6f2cb10d93        ov_net1             overlay             global

# 在host02查看network，也能看到 ov_net1，因为host01将overlay信息存入consul了
[root@host02 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
7e9e0643fca0        bridge              bridge              local
c10c03324445        host                host                local
318fbabb67bc        none                null                local
cc6f2cb10d93        ov_net1             overlay             global

```

### 在overlay中运行容器

```bash
# 运行一个busybox容器，并连接到ov_net1
[root@host01 ~]# docker run -itd --name bbox1 --network ov_net1 busybox

# 查看网络配置
[root@host01 ~]# docker exec bbox1 ip r
default via 172.18.0.1 dev eth1
10.0.0.0/24 dev eth0 scope link  src 10.0.0.2
172.18.0.0/16 dev eth1 scope link  src 172.18.0.2

```

bbox1有2个网络接口eth0和eth1；
eth0为10.0.0.2，连接ov_net1
eth1为172.18.0.2，连接bridge

```bash
# 查看网络配置
[root@host01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
106070dae414        bridge              bridge              local
53ae4ad6dcd5        docker_gwbridge     bridge              local
d8b4cf5bd66e        host                host                local
8dc7aa13590e        none                null                local
cc6f2cb10d93        ov_net1             overlay             global
afea471bd08c        ov_net2             overlay             global

```

通过`docker network inspect docker_gwbridge`可以看出，是bbox1中172网段地址的网关
bbox1是通过docker_gwbridge访问外网的

```bash
# 外网要访问容器的话，主机端口映射实现
[root@host01 ~]# docker run -p 80:80 -d --net ov_net1 --name web1 httpd

```

### overlay网络连通性
```bash
# 在host02创建一个bbox2的容器，指定overlay网络
[root@host02 ~]# docker run -itd --name bbox2 --network ov_net1 busybox

# 查看bbox2的网络路由
[root@host02 ~]# docker exec bbox2 ip r
default via 172.18.0.1 dev eth1
10.0.0.0/24 dev eth0 scope link  src 10.0.0.4
172.18.0.0/16 dev eth1 scope link  src 172.18.0.2
[root@host02 ~]#

# bbox2直接pingbbox1
[root@host02 ~]# docker exec bbox2 ping -c 2 bbox1
PING bbox1 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=1.952 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.360 ms

```

结论：overlay网络中的容器可以直接通信，同时docker实现了DNS服务

实现过程：
docker为每隔overlay网络创建一个独立的network namespace，其中会有一个linux bridge br0，endpoint还是由veth pair 实现，一端连接到容器中（即eth0），另一端连接到namespace的br0上；
br0除了连接所有的endpoint，还会连接一个vxlan，用于和其他host简历vxlan tunnel，容器之间的数据就是通过这个tunnel通信的；


### 查看overlay网络的namespace
```bash
ln -s /var/run/docker/netns /var/run/netns

# 查看namespace列表
[root@host01 ~]# ip netns
dfea0ee457de (id: 5)
1-afea471bd0 (id: 4)
825df6295f79 (id: 2)
c45108a4d5d3 (id: 3)
4f3fc60851d4 (id: 1)
1-cc6f2cb10d (id: 0)

# 安装brctl工具
[root@host01 ~]# yum -y install brctl
[root@host01 ~]# ip netns exec 1-cc6f2cb10d brctl show
bridge name bridge id       STP enabled interfaces
br0     8000.3a8caadb2ae3   no      veth0
                            veth1
                            vxlan0
# 查看名字空间的vxlan设备
[root@host01 ~]# ip netns exec 1-cc6f2cb10d brctl show
bridge name bridge id       STP enabled interfaces
br0     8000.3a8caadb2ae3   no      veth0
                            veth1
                            veth2
                            vxlan0
# 查看vxlan0的具体配置信息
[root@host01 ~]# ip netns exec 1-cc6f2cb10d ip -d l show vxlan0

```

### overlay网络隔离

不同的overlay网络使相互隔离的

```bash
# 创建一个容器bbox3指定在ov_net2下
[root@host01 ~]# docker run -itd --name bbox3 --network ov_net2 busybox
da0d59cd1ece2bfb459001d617e9e5abcd4a06fd1e2a35bd19245e8f86a9d17a
[root@host01 ~]#

# bbox3去pingbbox1的ip，不通
[root@host01 ~]# docker exec -it bbox3 ping -c 2 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes

--- 10.0.0.2 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

# 查看bbox1的信息
[root@host01 ~]# docker exec -it bbox1 ip r
default via 172.18.0.1 dev eth1
10.0.0.0/24 dev eth0 scope link  src 10.0.0.2
172.18.0.0/16 dev eth1 scope link  src 172.18.0.2

# 如果想要打通bbox1和bbox3网络，就把bbox3关联到ov_net1上
[root@host01 ~]# docker network connect ov_net1 bbox3

# 再次ping
[root@host01 ~]# docker exec -it bbox3 ping -c 2 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.169 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.085 ms


```









