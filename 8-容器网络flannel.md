
### 概念

flannel为每个host分配一个subnet，容器从此subnet中分配ip，这些ip可以在host间路由，容器间必须NAT和post mapping就可以跨主机通信

每个subnet都是从一个更大的ip池中划分的，flannel会在每个主机上运行一个叫flanneld的agent，其职责就是从池子中分配subnet。

为了在各个主机间共享信息，flannel用etcd（与consul类似的key-value分布式数据库）存放网络配置、已分配的subnet、host的ip等信息

数据包如何在主机间转发是由backend实现的，flannel提供了多种backend，常用的有vxlan和host-gw

### 安装配置etcd

参考官网安装
https://github.com/etcd-io/etcd/releases

把执行文件etcdctl、etcd拷贝到 /bin


### 将flannel网络的配置信息保存到etcd

```bash
# 启动etcd
[root@tk-tmp01:~]# nohup etcd --listen-client-urls "http://172.24.152.190:2379,http://127.0.0.1:2379" --advertise-client-urls "http://172.24.152.190:2379" &

# 设置key到etcd中
[root@tk-tmp01:~]# etcdctl --endpoints="http://172.24.152.190:2379" set /docker-test/network/config < flannel/flannel-config.json

# 获取key
[root@tk-tmp01:~]# etcdctl --endpoints="http://172.24.152.190:2379" get /docker-test/network/config
{ "Network": "10.2.0.0/16", "SubnetLen": 24, "Backend": { "Type": "vxlan" }}

# 查看已经分配的网段
[root@tk-tmp01:~]# etcdctl --endpoints="http://172.24.152.190:2379" ls /docker-test/network/subnets
/docker-test/network/subnets/10.2.46.0-24
/docker-test/network/subnets/10.2.28.0-24

```


### 在host01、host02启动flannel

```
# 在host01启动falnneld客户端服务
[root@host01 ~]# flanneld -etcd-endpoints=http://172.24.152.190:2379 -iface=eth0 -etcd-prefix=/docker-test/network

# 查看网卡接口信息
[root@host01 ~]# ip addr show flannel.1
52: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether f2:79:d1:59:ac:77 brd ff:ff:ff:ff:ff:ff
    inet 10.2.46.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever

# 查看路由信息
[root@host01 ~]# ip route
default via 172.24.159.253 dev eth0
10.2.28.0/24 via 10.2.28.0 dev flannel.1 onlink
169.254.0.0/16 dev eth0 scope link metric 1002
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
172.18.0.0/16 dev docker_gwbridge proto kernel scope link src 172.18.0.1
172.24.144.0/20 dev eth0 proto kernel scope link src 172.24.152.191

```

### 配置docker连接finnel




