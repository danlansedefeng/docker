# docker openvswitch网络跨主机互通

---
[TOC]

## 1. docker跨容器互通

![avatar](https://upload-images.jianshu.io/upload_images/11177530-8983891e1285cb1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/885/format/webp)

`https://www.jianshu.com/p/2e816843c80f`

## 2. 环境配置
### 2.1 122.114.139.22 配置
```
iptables -t nat -F
echo 0 > /proc/sys/net/ipv4/ip_forward
docker run -d --name con1 --net=none --privileged=true busybox top
docker run -d --name con2 --net=none --privileged=true busybox top
# 添加ovs网桥br0
ovs-vsctl add-br br0
# 为两个容器配置网络
ovs-docker add-port br0 eth0 con1 --ipaddress=192.168.0.1/16
ovs-docker add-port br0 eth0 con2 --ipaddress=192.168.0.2/16
# 建立gre tunnel
ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=122.114.139.5
```

执行结果

```
[root@k8snode02 ~]# ovs-vsctl show
e0dfe389-ed9e-4a5c-a9b1-9c57b64a2fc0
    Bridge "br0"
        Port "br0"
            Interface "br0"
                type: internal
        Port "9db563c66f414_l"
            Interface "9db563c66f414_l"
        Port "gre0"
            Interface "gre0"
                type: gre
                options: {remote_ip="122.114.139.5"}
        Port "4705e6d8639d4_l"
            Interface "4705e6d8639d4_l"
    ovs_version: "2.8.1"
```

### 2.2 122.114.139.5 环境配置
```
iptables -t nat -F
echo 0 > /proc/sys/net/ipv4/ip_forward
docker run -d --name con1 --net=none --privileged=true busybox top
docker run -d --name con2 --net=none --privileged=true busybox top
# 添加ovs网桥br0
ovs-vsctl add-br br0
# 为两个容器配置网络
ovs-docker add-port br0 eth0 con1 --ipaddress=192.168.1.1/16
ovs-docker add-port br0 eth0 con2 --ipaddress=192.168.1.2/16
# 建立gre tunnel
ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=122.114.139.22
```

## 3. 验证结果
```
[root@k8snode01 ~]# docker exec -it con1 ping -c 1 192.168.0.1
PING 192.168.0.1 (192.168.0.1): 56 data bytes
64 bytes from 192.168.0.1: seq=0 ttl=64 time=1.387 ms

--- 192.168.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.387/1.387/1.387 ms
[root@k8snode01 ~]# docker exec -it con1 ping -c 1 192.168.0.2
PING 192.168.0.2 (192.168.0.2): 56 data bytes
64 bytes from 192.168.0.2: seq=0 ttl=64 time=1.090 ms

--- 192.168.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.090/1.090/1.090 ms
[root@k8snode01 ~]# docker exec -it con1 ping -c 1 192.168.1.1
PING 192.168.1.1 (192.168.1.1): 56 data bytes
64 bytes from 192.168.1.1: seq=0 ttl=64 time=0.061 ms

--- 192.168.1.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.061/0.061/0.061 ms
[root@k8snode01 ~]# docker exec -it con1 ping -c 1 192.168.1.2
PING 192.168.1.2 (192.168.1.2): 56 data bytes
64 bytes from 192.168.1.2: seq=0 ttl=64 time=0.501 ms

--- 192.168.1.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.501/0.501/0.501 ms
```
