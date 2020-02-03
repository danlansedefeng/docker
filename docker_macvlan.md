## MacVLAN实现Docker跨宿主机互联

#### 1. 环境准备

|主机名|ip地址|软件环境|
|----|---|---|
|ceph-01|122.114.139.26|kernel>3.9,docker|
|ceph-02|122.114.139.27|kernel>3.9,docker|

#### 2.环境安装
```
# 内核升级
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install kernel-ml
grub2-set-default 'kernel-ml-5.4.2-1.el7.elrepo.x86_64'
reboot

# 软件安装
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io
systemctl enable docker
systemctl start docker

# 设置混杂模式
ip link set eth2  promisc on

```

#### 3. 环境部署

##### 创建macvlan网络
```
docker network create -d macvlan --subnet 122.114.139.0/24 --gateway 122.114.139.1 -o parent=eth2 -o macvlan_mode=bridge macnet
```

##### 查看macvlan是否创建成功
```
[root@ceph01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
07c93774866b        bridge              bridge              local
92ae16caf225        host                host                local
1c67c6609d02        macnet              macvlan             local
8a178527ca4b        none                null                local
```

##### 创建两个使用macvlan容器

创建容器c1
```
[root@ceph01 ~]# docker run -id --net macnet --ip 122.114.139.30 --name c1 busybox sh
fd4d78cc4004c4538375d876e3734531f76df508fcf7c37e9e93fd49914d89d5
[root@ceph01 ~]#
[root@ceph01 ~]#
[root@ceph01 ~]# docker exec c1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:7a:72:8b:1e brd ff:ff:ff:ff:ff:ff
    inet 122.114.139.30/24 brd 122.114.139.255 scope global eth0
       valid_lft forever preferred_lft forever
[root@ceph01 ~]# docker exec c1 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         122.114.139.1   0.0.0.0         UG    0      0        0 eth0
122.114.139.0   0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

创建容器c2

```
[root@ceph01 ~]# docker run -id --net macnet --ip 122.114.139.31 --name c2 busybox sh  
cd6bfb44442e97f2e90b688cf9b23b6e8cfa23546ad7c24321825f1e6f60e3c7
[root@ceph01 ~]#
[root@ceph01 ~]#
[root@ceph01 ~]# docker exec c2 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
9: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:7a:72:8b:1f brd ff:ff:ff:ff:ff:ff
    inet 122.114.139.31/24 brd 122.114.139.255 scope global eth0
       valid_lft forever preferred_lft forever
[root@ceph01 ~]# docker exec c2 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         122.114.139.1   0.0.0.0         UG    0      0        0 eth0
122.114.139.0   0.0.0.0         255.255.255.0   U     0      0        0 eth0
```


* node主机

##### 创建macvlan网络
```
docker network create -d macvlan --subnet 122.114.139.0/24 --gateway 122.114.139.1 -o parent=eth2 -o macvlan_mode=bridge macnet
```

创建两个使用macvlan的容器
```
[root@ceph02 ~]# docker run -id --net macnet --ip 122.114.139.32 --name c3 busybox sh
acba44d52599dcbf11704d47524a0747b2c2fdb858b788d3479539aebbb94c53
[root@ceph02 ~]#
[root@ceph02 ~]# docker run -id --net macnet --ip 122.114.139.33 --name c4 busybox sh
9bb734d48c035c484b2690ad0d6d2dfe24cf094c7fd7bdc9caea03d6840d4093
[root@ceph02 ~]#
```

##### https://www.cnblogs.com/luoahong/p/10289072.html
