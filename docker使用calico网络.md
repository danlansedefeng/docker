## <center>docker使用calico网络<center/>

#### calico简单简介

```
他是一个纯三层的方法，使用虚拟路由代替虚拟交换，每一台虚拟路由通过BGP协议传播可达信息(路由)到剩余数据中心。
```

---

![calico架构](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1576743264785&di=1c505d4b4605a8c730c528bf2b78250c&imgtype=jpg&src=http%3A%2F%2Fimg1.imgtn.bdimg.com%2Fit%2Fu%3D3257302456%2C2292096233%26fm%3D214%26gp%3D0.jpg)

```
结合上面这张图，我们来过一遍 Calico 的核心组件：

Felix，Calico agent，跑在每台需要运行 workload 的节点上，主要负责配置路由及 ACLs 等信息来确保 endpoint 的连通状态；

etcd，分布式键值存储，主要负责网络元数据一致性，确保 Calico 网络状态的准确性；

BGP Client(BIRD), 主要负责把 Felix 写入 kernel 的路由信息分发到当前 Calico 网络，确保 workload 间的通信的有效性；

BGP Route Reflector(BIRD), 大规模部署时使用，摒弃所有节点互联的 mesh 模式，通过一个或者多个BGP Route Reflector来完成集中式的路由分发；

通过将整个互联网的可扩展 IP 网络原则压缩到数据中心级别，Calico 在每一个计算节点利用Linux kernel实现了一个高效的vRouter来负责数据转发而每个vRouter通过BGP 
协议负责把自己上运行的 workload 的路由信息像整个 Calico 网络内传播 － 小规模部署可以直接互联，大规模下可通过指定的 
BGP route reflector 来完成。

这样保证最终所有的 workload 之间的数据流量都是通过 IP 包的方式完成互联的。
```

#### 环境准备
``` bash
122.114.139.26 ceph01
122.114.139.27 ceph02
122.114.139.28 ceph03
```

* 软件版本

``` bash
calicoctl（version v1.6.5） etcdctl（version: 3.2.22） docker（version：18.09.0-ce）
```

* 安装etcd,所有节点上执行

``` bash
yum -y install etcd
```

* 修改etcd 配置文件

* ceph01节点

``` bash
ETCD_NAME="ceph01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_INITIAL_CLUSTER="ceph01=http://122.114.139.26:2380,ceph02=http://122.114.139.27:2380,ceph03=http://122.114.139.28:2380"
ETCD_INITIAL_CLUSTER_STATE=new

ETCD_INITIAL_ADVERTISE_PEER_URLS=http://122.114.139.26:2380
ETCD_ADVERTISE_CLIENT_URLS=http://122.114.139.26:2379

ETCD_LISTEN_PEER_URLS=http://122.114.139.26:2380
ETCD_LISTEN_CLIENT_URLS="http://122.114.139.26:2379"

#[proxy]
ETCD_PROXY="off"
```

* ceph02节点

``` bash
ETCD_NAME="ceph02"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_INITIAL_CLUSTER="ceph01=http://122.114.139.26:2380,ceph02=http://122.114.139.27:2380,ceph03=http://122.114.139.28:2380"
ETCD_INITIAL_CLUSTER_STATE=new

ETCD_INITIAL_ADVERTISE_PEER_URLS=http://122.114.139.27:2380
ETCD_ADVERTISE_CLIENT_URLS=http://122.114.139.27:2379

ETCD_LISTEN_PEER_URLS=http://122.114.139.27:2380
ETCD_LISTEN_CLIENT_URLS="http://122.114.139.27:2379"

#[proxy]
ETCD_PROXY="off"
```

* ceph03节点

``` bash
ETCD_NAME="ceph03"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_INITIAL_CLUSTER="ceph01=http://122.114.139.26:2380,ceph02=http://122.114.139.27:2380,ceph03=http://122.114.139.28:2380"
ETCD_INITIAL_CLUSTER_STATE=new

ETCD_INITIAL_ADVERTISE_PEER_URLS=http://122.114.139.28:2380
ETCD_ADVERTISE_CLIENT_URLS=http://122.114.139.28:2379

ETCD_LISTEN_PEER_URLS=http://122.114.139.28:2380
ETCD_LISTEN_CLIENT_URLS="http://122.114.139.28:2379"

#[proxy]
ETCD_PROXY="off"
```

* 启动服务

``` bash
systemctl enable etcd && systemctl start etcd
```

* 安装docker

``` bash
yum install -y yum-utils epel-release device-mapper-persistent-data  lvm2
yum-config-manager  --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum -y install docker-ce docker-ce-cli containerd.io

```

* 设置docker配置文件连接etcd，所有机器上执行

``` bash
{
  "registry-mirrors": ["http://f1361db2.m.daocloud.io"],
  "cluster-store": "etcd://122.114.139.26:2379"
}
```

* 启动服务查看配置是否正确

``` bash
systemctl enable docker && systemctl start docker && docker info|grep etcd
```

#### 配置calico

``` bash
wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v1.6.5/calicoctl
chmod +x /usr/local/bin/calicoctl
/usr/local/bin/calicoctl --version
```

* 创建calico容器
首先创建calico配置文件,每台机器不一样，注意修改ip

``` bash
mkdir /etc/calico
cat > /etc/calico/calico.cfg << 'EOF'
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "http://122.114.139.26:2379"
EOF
```

* 分别在两个节点上创建calico容器,执行后会自动下载calico镜像并运行容器，下载镜像可能较慢，建议配置镜像加速，启动过程如下：

``` bash
calicoctl node run --node-image=quay.io/calico/node:v2.6.12  -c /etc/calico/calico.cfg  --ip=122.114.139.26
```

* 创建calico网络

``` bash
docker network create --driver calico --ipam-driver calico-ipam cal_net1
```

`–driver calico 指定使用 calico 的 libnetwork CNM driver。`
`–ipam-driver calico-ipam 指定使用 calico 的 IPAM driver 管理 IP`


* 在 ceph01 中运行容器 bbox1 并连接到 cal_net1：

``` bash
docker container run --net cal_net1 --name bbox1 -tid busybox
```

* 查看bbox1的配置

``` bash
[root@ceph01 ~]# docker exec bbox1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: cali0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff
    inet 192.168.127.0/32 brd 192.168.127.0 scope global cali0
       valid_lft forever preferred_lft forever
[root@ceph01 ~]# docker exec bbox1 ip route sh
default via 169.254.1.1 dev cali0
169.254.1.1 dev cali0 scope link
```

*  将作为 router 负责转发目的地址为 bbox1 的数据包。

``` bash
[root@ceph01 ~]# ip route sh
default via 122.114.139.1 dev eth0
10.0.0.0/8 via 10.136.0.1 dev eth1
10.136.0.0/16 dev eth1 proto kernel scope link src 10.136.1.241
122.114.139.0/24 dev eth0 proto kernel scope link src 122.114.139.26
169.254.0.0/16 dev eth0 scope link metric 1002
169.254.0.0/16 dev eth1 scope link metric 1003
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.127.0 dev cali384dd2a4262 scope link
blackhole 192.168.127.0/26 proto bird
```

`所有发送到 bbox1 的数据都会发给caliea27309bc46，因为caliea27309bc46 与 cali0 是一对 veth pair，bbox1 能够接收到数据。`
`接下来我们在 ceph02 中运行容器 bbox2，也连接到 cal_net1：`

``` bash
docker container run --net cal_net1 --name bbox2 -tid busybox
```
` ip地址为 192.168.157.64`

* 测试一下 bbox1 与 bbox2 的连通性：

```bash
[root@ceph01 ~]# docker exec bbox1 ping -c 2 bbox2
PING bbox2 (192.168.157.64): 56 data bytes
64 bytes from 192.168.157.64: seq=0 ttl=62 time=0.279 ms
64 bytes from 192.168.157.64: seq=1 ttl=62 time=0.200 ms

--- bbox2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.200/0.239/0.279 ms
```

* 数据包经过
1. 根据 bbox1 的路由表，将数据包从 cal0 发出。
``` bash
[root@ceph01 ~]# docker exec bbox1 ip route sh
default via 169.254.1.1 dev cali0
169.254.1.1 dev cali0 scope link
```
2. 数据经过 veth pair 到达 ceph01，查看路由表，数据由 eth0发给 ceph02（122.114.139.27）
``` bash
192.168.157.64/26 via 122.114.139.27 dev eth0 proto bird
```
3. ceph02收到数据包，根据路由表送给 cali120795c955b，进而通过 veth pair cali0 到达 bbox2


#### 遇到的问题

* 下载的calico最新版本3.0,按照你的教程配置无法启动，总是提示“no etcd endpoints specified”，明明配置了etcd，并且用etcdctl也可以成功连接的！

``` 
后面看了calico官网进行配置可以成功连接，留给后面有同样问题的同学一个参考吧！
添加一个新文件/etc/calico/calicoctl.cfg，配置内容如下：
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "http://122.114.139.26:2379"
```

---

* Error response from daemon: plugin "calico" not found

``` 
calico3.0 开始不支持docker部署，使用calico1.6版本，并使用对应的容器
```
