---
title: "使用 CNI 为 Containerd 容器添加网络"
isCJKLanguage: true
date: 2022-10-10T17:29:16+08:00
#lastmod: 2022-04-28T15:13:38+08:00
#categories: [CSI]
#tags: [kubernetes, csi, storage, volume]
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**
---

# 前言
  在 [部署 Containerd](kubernetes/containerd/install-containerd/) 中通过 ctr run 了一个容器，该容器没有任何的网络能力，这里通过 containerd 与 cni 插件的结合，为容器加入基本的网络(bridge)能力

```shell
# ctr task exec -t --exec-id alpine alpine  sh
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
      valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
      valid_lft forever preferred_lft forever
/ # ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
/ # 
```

# CNI
[CNI](https://www.cni.dev/)(Container Network Interface) 是 CNCF 旗下的一个项目，由一个规范和编写插件的库组成，用于在 Linux 容器中配置网络接口，同时还有一些支持的插件。CNI 只关注容器的网络连接，以及在容器被删除时移除分配的资源

# 安装 CNI,ctrtool

安装目前最新的版 [cni-v1.1.1](https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz), cnitool 是一个执行和验证 CNI 配置的简单程序, 它可以在一个已经创建的网络命名空间中添加或删除接口

```shell
# mkdir -p /opt/cni/bin
# wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz -O - | tar -zxf - -C /opt/cni/bin
# go install github.com/containernetworking/cni/cnitool@latest
go: downloading github.com/containernetworking/cni v1.1.2
# tree /opt/cni/bin/
/opt/cni/bin/
├── bandwidth
├── bridge
├── dhcp
├── firewall
├── host-device
├── host-local
├── ipvlan
├── loopback
├── macvlan
├── portmap
├── ptp
├── sbr
├── static
├── tuning
├── vlan
└── vrf

0 directories, 16 files

# tree $GOPATH/bin/
/go/bin/
└── cnitool

0 directories, 1 file

```

# cni 的分类

[cni plugins 分类](https://www.cni.dev/plugins/current/#reference-plugins) (单纯的搬运工，详情请移步官网)

## Main: interface-creating

* `bridge`: Creates a bridge, adds the host and the container to it
* `ipvlan`: Adds an [ipvlan](https://www.kernel.org/doc/Documentation/networking/ipvlan.txt) interface in the container
* `macvlan`: Creates a new MAC address, forwards all traffic to that to the container
* `ptp`: Creates a veth pair
* `host-device`: Moves an already-existing device into a container
* `vlan`: Creates a vlan interface off a master

## Windows: windows specific

* `win-bridge`: Creates a bridge, adds the host and the container to it
* `win-overlay`: Creates an overlay interface to the container

## IPAM: IP address allocation
* `dhcp`: Runs a daemon on the host to make DHCP requests on behalf of a container
* `host-local`: Maintains a local database of allocated IPs
* `static`: Allocates static IPv4/IPv6 addresses to containers

## Meta: other plugins

* `tuning`: Changes sysctl parameters of an existing interface
* `portmap`: An iptables-based portmapping plugin. Maps ports from the host's address space to the container
* `bandwidth`: Allows bandwidth-limiting through use of traffic control tbf (ingress/egress)
* `sbr`: A plugin that configures source based routing for an interface (from which it is chained)
* `firewall`: A firewall plugin which uses iptables or firewalld to add rules to allow traffic to/from the container


# cni 配置
## 配置文件
这里简单配置、调试一下 [bridge plugin](https://greencloudvps.com/billing/clientarea.php) 方式, [cni v1.0.0](https://www.cni.dev/docs/spec/#version), [默认配置文件](https://www.cni.dev/docs/cnitool/#environment-variables)放在 /etc/cni/net.d
```shell
# mkdir -p /etc/cni/net.d
# cat <<-EOF | tee /etc/cni/net.d/mynet.conf
{
    "cniVersion": "1.0.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "mynet0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.10.0.0/16"
    }
}
EOF
```

## 创建 ns

通过 [cnitool](https://www.cni.dev/docs/cnitool/) 来添加网络接口,默认情况，没有多余的桥口， netns 是空的

```shell
# brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02422f9e43bb       no
# ip netns ls

```
- 创建一个名为 testing 的 network namespace
```shell
# ip netns add testing
# ip netns ls
testing
# tree /var/run/netns/
/var/run/netns/
└── testing

0 directories, 1 file
# stat /var/run/netns/testing
  File: /var/run/netns/testing
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: 4h/4d   Inode: 4026532738  Links: 1
Access: (0444/-r--r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2022-10-11 02:58:39.178583169 -0400
Modify: 2022-10-11 02:58:39.178583169 -0400
Change: 2022-10-11 02:58:39.178583169 -0400
 Birth: -

```

## 添加网络
```shell
### 目前的 ns 中除了 lo 是没有其他网络的
# ip -n testing addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# CNI_PATH=/opt/cni/bin cnitool add mynet /var/run/netns/testing
{
    "cniVersion": "1.0.0",
    "interfaces": [
        {   ### 宿主机上创建的桥接口
            "name": "mynet0",
            "mac": "e2:70:39:4d:80:0c"
        },
        {   ### 宿主机上的接口和 mac
            "name": "veth73bb31cb",
            "mac": "9e:ff:06:e1:9a:df"
        },
        {    ### ns 中的接口和 mac
            "name": "eth0",
            "mac": "7a:ba:f0:e7:76:fd",
            "sandbox": "/var/run/netns/testing"
        }
    ],
    "ips": [
        {   ### 分配的 ip 地址
            "interface": 2,
            "address": "10.10.0.2/16",
            "gateway": "10.10.0.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0",
            "gw": "10.10.0.1"
        }
    ],
    "dns": {}
}
```
## 验证网络
```shell
### 检查创建的结果是否正确
### 桥接口能对应上
# brctl show 
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02422f9e43bb       no
mynet0          8000.e270394d800c       no              veth73bb31cb

# ip addr show dev mynet0
30: mynet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether e2:70:39:4d:80:0c brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.1/16 brd 10.10.255.255 scope global mynet0
       valid_lft forever preferred_lft forever
    inet6 fe80::e070:39ff:fe4d:800c/64 scope link 
       valid_lft forever preferred_lft forever
# 宿主机上的接口和 mac 能够对应上
# ip link show dev veth73bb31cb
31: veth73bb31cb@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master mynet0 state UP mode DEFAULT group default 
    link/ether 2a:a0:79:c5:10:6c brd ff:ff:ff:ff:ff:ff link-netns testing
# ns 中的接口，mac 和 ip 能对应上
# ip -n testing addr show dev eth0
2: eth0@if31: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 7a:ba:f0:e7:76:fd brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.0.2/16 brd 10.10.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::78ba:f0ff:fee7:76fd/64 scope link 
       valid_lft forever preferred_lft forever
### ns 中 ping 网关
# ip netns exec testing ping -c 1  10.10.0.1
PING 10.10.0.1 (10.10.0.1) 56(84) bytes of data.
64 bytes from 10.10.0.1: icmp_seq=1 ttl=64 time=0.062 ms

--- 10.10.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.062/0.062/0.062/0.000 ms
### 宿主机中 ping ns
# ping -c 1 10.10.0.2
PING 10.10.0.2 (10.10.0.2) 56(84) bytes of data.
64 bytes from 10.10.0.2: icmp_seq=1 ttl=64 time=0.052 ms

--- 10.10.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.052/0.052/0.052/0.000 ms

```

## 新建 container 并加入 ns

```shell
# ctr run --with-ns=network:/var/run/netns/testing -d docker.io/library/alpine:3.14 alpine-net
# ctr task exec -t --exec-id alpine alpine-net sh
# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0@if31: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 7a:ba:f0:e7:76:fd brd ff:ff:ff:ff:ff:ff
    inet 10.10.0.2/16 brd 10.10.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::78ba:f0ff:fee7:76fd/64 scope link 
       valid_lft forever preferred_lft forever
# ip r
default via 10.10.0.1 dev eth0 
10.10.0.0/16 dev eth0 scope link  src 10.10.0.2 
# ping 10.10.0.1 -c 1
PING 10.10.0.1 (10.10.0.1): 56 data bytes
64 bytes from 10.10.0.1: seq=0 ttl=64 time=0.081 ms

--- 10.10.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.081/0.081/0.081 ms
```

## 清理
- 删除 container
```shell
# ctr task ls
TASK          PID       STATUS
alpine        106032    RUNNING
alpine-net    193076    RUNNING
# ctr task kill -s SIGKILL alpine-net
# ctr task kill -s SIGKILL alpine
# ctr task ls
TASK          PID       STATUS
alpine        106032    STOPPED
alpine-net    193076    STOPPED
# ctr task rm alpine
# ctr task rm alpine-net
# ctr container ls 
CONTAINER     IMAGE                            RUNTIME                  
alpine        docker.io/library/alpine:3.14    io.containerd.runc.v2    
alpine-net    docker.io/library/alpine:3.14    io.containerd.runc.v2    
# ctr container rm alpine alpine-net
# ctr container ls 
CONTAINER    IMAGE    RUNTIME    
```
- 清除 ns
```shell
# CNI_PATH=/opt/cni/bin cnitool del mynet /var/run/netns/testing
# ip netns del testing
# ip netns ls

```

# Thanks 
[CNI](https://www.cni.dev/)  
[cnitool](https://www.cni.dev/docs/cnitool/)  
[cni bridge plugin](https://www.cni.dev/plugins/current/main/bridge/)
[使用CNI为Containerd容器添加网络能力](https://blog.frognew.com/2021/04/relearning-container-03.html)  
[containerd](https://github.com/containerd/containerd)
