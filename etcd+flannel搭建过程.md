## <center>etcd+flannel网络搭建过程<center>


#### flannel网络环境

``` bash
122.114.139.26 ceph01
122.114.139.27 ceph02
122.114.139.28 ceph03
```
  
![etcd网络环境图](http://blog.gcalls.cn/images/packet-01.png)

```
  flannel首先创建了一个名为flannel0的网桥，而且这个网桥的一端连接docker0网桥，另一端连接一个叫做flanneld的服务进程
  
  flannel进程上连etcd，利用etcd来管理可分配的ip地址段资源，同时监控etcd中每个pod的实际地址，并在内存中简历了一个pod节点路由表，它下连docker0和物理网络，使用内存中的pod节点路由表，将docker0发给它的数据包包装起来，利用物理网络的连接将数据包投递到目标flanneld上，从而完成pod到pod之间的直接地址通信。

```

### etcd搭建过程


* 1.在所有主机上安装etcd

``` bash
yum -y install etcd
```

* 2.在所有主机上编辑etcd.conf配置文件

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

* 3.启动服务

``` bash
systemctl enable etcd
systemctl start etcd
```

* 4.测试

``` bash
etcdctl --endpoints "http://122.114.139.26:2379,http://122.114.139.27:2379,http://122.114.139.28:2379" cluster-health

```

* 5.采坑
  集群搭建，启动时候报错

```
解决办法：删除了etcd集群所有节点中的--data_dir的内容
分析： 因为集群搭建过程，单独启动过单一etcd,做为测试验证，集群内第一次启动其他etcd服务时候，是通过发现服务引导的，所以需要删除旧的成员信息
```
https://blog.51cto.com/1666898/2156165


### flannel搭建过程

* 安装软件，在每个节点执行

``` bash
yum -y install flannel
```

* 配置在etcd中设置flannel所使用的ip段:

``` bash
etcdctl --endpoints "http://122.114.139.26:2379,http://122.114.139.27:2379,http://122.114.139.28:2379" set /coreos.com/network/config '{"NetWork":"10.244.0.0/16"}'
```

* 每台执行:

``` bash
sed -i 's;^FLANNEL_ETCD_ENDPOINTS=.*;FLANNEL_ETCD_ENDPOINTS="http://122.114.139.26:2379,http://122.114.139.27:2379,http://122.114.139.28:2379";g' \
/etc/sysconfig/flanneld

sed -i 's;^FLANNEL_ETCD_PREFIX=.*;FLANNEL_ETCD_PREFIX="/coreos.com/network";g' \
/etc/sysconfig/flanneld

grep -v ^# /etc/sysconfig/flanneld
```

* 在service脚本中，会自动通过以下命令生成docker bip所需要的环境变量：

``` bash
/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker

[root@ceph01 ~]# cat /run/flannel/docker
DOCKER_OPT_BIP="--bip=10.244.7.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1472"
DOCKER_NETWORK_OPTIONS=" --bip=10.244.7.1/24 --ip-masq=true --mtu=1472"

```

* 修改docker网段：

``` bash
sed -i -e '/ExecStart=/iEnvironmentFile=/run/flannel/docker' -e 's;^ExecStart=/usr/bin/dockerd;ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS;g' \
/usr/lib/systemd/system/docker.service

#$ vim /usr/lib/systemd/system/docker.service
#EnvironmentFile=/run/flannel/docker
#ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS

#重启docker服务
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

* 配置flannel0

``` bash
source /run/flannel/subnet.env
ifconfig docker0 ${FLANNEL_SUBNET}
```

### 测试

* 创建机器，在每台机器上运行

``` bash
[root@ceph01 ~]# docker exec con1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1472 qdisc noqueue
    link/ether 02:42:0a:f4:07:02 brd ff:ff:ff:ff:ff:ff
    inet 10.244.7.2/24 brd 10.244.7.255 scope global eth0
       valid_lft forever preferred_lft forever
[root@ceph01 ~]# docker exec con1 ping -c 1 10.244.79.2
PING 10.244.79.2 (10.244.79.2): 56 data bytes
64 bytes from 10.244.79.2: seq=0 ttl=60 time=1.014 ms

--- 10.244.79.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.014/1.014/1.014 ms
[root@ceph01 ~]# docker exec con1 ping -c 1 10.244.38.2
PING 10.244.38.2 (10.244.38.2): 56 data bytes
64 bytes from 10.244.38.2: seq=0 ttl=60 time=1.584 ms

--- 10.244.38.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.584/1.584/1.584 ms
[root@ceph01 ~]#
```

### 遇到的问题

* flannel网关可以ping通，但是无法ping通容器内ip

``` bash
iptables -P FORWARD ACCEPT
```

* 参考教程

http://blog.sina.com.cn/s/blog_704836f40102x9qj.html

https://blog.51cto.com/1666898/2156165

http://blog.gcalls.cn/blog/2017/01/%E4%BD%BF%E7%94%A8Flannel%E6%90%AD%E5%BB%BAdocker%E7%BD%91%E7%BB%9C.html

https://zhizhebuyan.com/2017/08/07/flannel%E7%BD%91%E5%85%B3%E5%8F%AF%E4%BB%A5ping%E9%80%9A%EF%BC%8C%E4%BD%86%E6%98%AF%E6%97%A0%E6%B3%95ping%E9%80%9A%E5%AE%B9%E5%99%A8%E5%86%85ip%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/
