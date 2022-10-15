---
title: "docker网络"
author: 一张狗
lastmod: 2019-07-06 12:07:29
date: 2018-02-05 22:23:00
tags: []
---



## 一、docker网络技术基础和名词

docker的网络基于linux network namespace提供了网络资源的隔离，包括网络设备、端口、/proc/net目录、/sys/class/net目录等。

linux bridge：是Linux 上用来做TCP/IP 二层协议交换的设备，与现实世界中的交换机功能相似。


## 二、docker网络模式

1. bridge模式  
 docker进程启动时，会在主机中构建一个docker0的虚拟网桥，类似于交换机，Docker 容器与外部的通信都是通过 iptable 来实现的。主机上所有容器连到docker0这个交换机上，docker0子网分配一个ip给各个容器使用，bridge模式是默认模式。docker run起容器时，创建一对虚拟网卡，一个放在容器，一个在宿主机Network namespace，并且从私有IP段分配ip给网桥，与各容器之间构成一个子网络，并在iptables加DNAT规则实现转发。![](http://wiki.baidu.com/download/attachments/452613663/image2016-8-1015_16_3.png?version=2&modificationDate=1515655438000&api=v2)
2. host模式  
 docker不单独使用网络命名空间，使用宿主机的网络。
3. container模式  
 与已经存在的容器共享一个网络命名空间。
4. none模式  
 拥有自己的网络命名空间，但是宿主机不为该容器创建任何网络配置，即没有网卡、IP、路由，需要自行配置。在该模式下容器没有对外网络，本机只有一个回路地址。
5. Overlay模式  
 Docker 目前原生的跨主机多子网模型，主要是通过 vxlan 技术来实现的



## 三、bridge模式的实现细节：

1. 在一台宿主机上启动docker： 
```
[work@your_machine ~]$ ifconfig
br-54bef593d492: flags=4099<UP,BROADCAST,MULTICAST> mtu 1500
	inet 172.18.0.1 netmask 255.255.0.0 broadcast 0.0.0.0
	inet6 fe80::42:8dff:fe84:c527 prefixlen 64 scopeid 0x20<link>
docker0: flags=4099<UP,BROADCAST,MULTICAST> mtu 1500
	inet 172.17.0.1 netmask 255.255.0.0 broadcast 0.0.0.0
	inet6 fe80::42:d2ff:fe8a:a9c9 prefixlen 64 scopeid 0x20<link>
ens11: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
	inet 10.103.179.40 netmask 255.255.255.0 broadcast 10.103.179.255
	inet6 fe80::566e:ac37:c006:4c6d prefixlen 64 scopeid 0x20<link>
lo: flags=73<UP,LOOPBACK,RUNNING> mtu 65536
	inet 127.0.0.1 netmask 255.0.0.0
	inet6 ::1 prefixlen 128 scopeid 0x10<host>
virbr0: flags=4099<UP,BROADCAST,MULTICAST> mtu 1500
	inet 192.168.122.1 netmask 255.255.255.0 broadcast 192.168.122.255
[work@your_machine ~]$ brctl show
bridge name bridge id STP enabled interfaces
br-54bef593d492 8000.02428d84c527 no
docker0 8000.0242d28aa9c9 no
virbr0 8000.525400a48be1 yes virbr0-nic
```

可以看到docker0是个网桥
2. 进入容器之后，在主机ifconfig：  
 可以看到多了veth2a129e8
```
br-54bef593d492: flags=4099<UP,BROADCAST,MULTICAST> mtu 1500
	inet 172.18.0.1 netmask 255.255.0.0 broadcast 0.0.0.0
	inet6 fe80::42:8dff:fe84:c527 prefixlen 64 scopeid 0x20<link>
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
	inet 172.17.0.1 netmask 255.255.0.0 broadcast 0.0.0.0
	inet6 fe80::42:d2ff:fe8a:a9c9 prefixlen 64 scopeid 0x20<link>
ens11: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
	inet 10.103.179.40 netmask 255.255.255.0 broadcast 10.103.179.255
	inet6 fe80::566e:ac37:c006:4c6d prefixlen 64 scopeid 0x20<link>
lo: flags=73<UP,LOOPBACK,RUNNING> mtu 65536
	inet 127.0.0.1 netmask 255.0.0.0
	inet6 ::1 prefixlen 128 scopeid 0x10<host>
veth2a129e8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
	inet6 fe80::2470:3eff:fe47:29e8 prefixlen 64 scopeid 0x20<link>
virbr0: flags=4099<UP,BROADCAST,MULTICAST> mtu 1500
	inet 192.168.122.1 netmask 255.255.255.0 broadcast 192.168.122.255
```
3. 进入容器ifconfig 
```
[root@59e246cbf2b7 /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1500
	inet 172.17.0.2 netmask 255.255.0.0 broadcast 0.0.0.0
	inet6 fe80::42:acff:fe11:2 prefixlen 64 scopeid 0x20<link>
	ether 02:42:ac:11:00:02 txqueuelen 0 (Ethernet)
lo: flags=73<UP,LOOPBACK,RUNNING> mtu 65536
	inet 127.0.0.1 netmask 255.0.0.0
	inet6 ::1 prefixlen 128 scopeid 0x10<host>
```

4. 回去宿主机： 
```
[root@your_machine ~]# route
Kernel IP routing table
Destination Gateway Genmask Flags Metric Ref Use Iface
default gateway 0.0.0.0 UG 100 0 0 ens11
10.103.179.0 0.0.0.0 255.255.255.0 U 100 0 0 ens11
172.17.0.0 0.0.0.0 255.255.0.0 U 0 0 0 docker0
172.18.0.0 0.0.0.0 255.255.0.0 U 0 0 0 br-54bef593d492
192.168.122.0 0.0.0.0 255.255.255.0 U 0 0 0 virbr0
```

可以看到发往172.17.0.2的数据包路由到了docker0网桥。

```
[root@your_machine ~]# brctl show
bridge name bridge id STP enabled interfaces
br-54bef593d492 8000.02428d84c527 no
docker0 8000.0242d28aa9c9 no veth2a129e8
virbr0 8000.525400a48be1 yes virbr0-nic
[root@your_machine ~]# arp -n
Address HWtype HWaddress Flags Mask Iface
172.18.176.53 (incomplete) br-54bef593d492
10.103.179.1 ether 54:ab:3a:ca:93:b5 C ens11
10.103.179.24 ether 7c:d3:0a:c8:40:b6 C ens11
172.18.0.3 ether 02:42:ac:12:00:03 C br-54bef593d492
172.17.0.5 ether 02:42:ac:11:00:05 C docker0
172.17.0.2 ether 02:42:ac:11:00:02 C docker0
10.103.179.22 ether 7c:d3:0a:c8:40:a2 C ens11
172.18.180.83 (incomplete) br-54bef593d492
172.17.0.4 ether 02:42:ac:11:00:04 C docker0
172.18.0.4 ether 02:42:ac:12:00:04 C br-54bef593d492
172.17.0.3 ether 02:42:ac:11:00:03 C docker0
```
综合ifconfig可以看到新增了veth2a129e8这个虚拟网卡，通过宿主机上arp -n 发现172.17.0.2绑定到了mac地址 02:42:ac:11:00:02。veth2a129e8其peer_ifindex在这个例子里是395：
```
[root@your_machine ~]# ethtool -S veth2a129e8
NIC statistics:
peer_ifindex: 395
```
5. 在容器内： 
```
[root@59e246cbf2b7 /]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
	link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
395: eth0@if396: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
	link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
可以看到395路由到了本容器的eth0。
6. 总结：  
 每当新启动一个容器，主机就会增加一对VETH设备，把一个连接到docker0上，另一个挂载到容器内部的eth0里。


## 四、bridge模式下端口映射

在bridge模式下启动端口映射的容器，其实是通过NAT与外界互联。在通过-p参数启动一个容器后，iptables中加规则，以8038端口为例：
```
Chain DOCKER (5 references)
target prot opt source destination
ACCEPT tcp -- 0.0.0.0/0 172.17.0.4 tcp dpt:8038
Chain POSTROUTING (policy ACCEPT 96 packets, 6206 bytes)
pkts bytes target prot opt in out source destination
0 0 MASQUERADE tcp -- * * 172.17.0.4 172.17.0.4 tcp dpt:8038
Chain DOCKER (2 references)
pkts bytes target prot opt in out source destination
0 0 RETURN all -- br-fdb572d43390 * 0.0.0.0/0 0.0.0.0/0
0 0 RETURN all -- docker0 * 0.0.0.0/0 0.0.0.0/0
0 0 RETURN all -- br-f8aba8ff1d1b * 0.0.0.0/0 0.0.0.0/0
0 0 RETURN all -- br-54bef593d492 * 0.0.0.0/0 0.0.0.0/0
0 0 RETURN all -- docker_gwbridge * 0.0.0.0/0 0.0.0.0/0
114 6840 DNAT tcp -- !docker0 * 0.0.0.0/0 0.0.0.0/0 tcp dpt:8500 to:172.17.0.2:8500
0 0 DNAT tcp -- !docker0 * 0.0.0.0/0 127.0.0.1 tcp dpt:8038 to:172.17.0.4:8038
```
## 五、容器互联

1. 如果容器使用宿主机网络（–net=host）则容器间互联跟宿主机互联的方式一样。下面讨论不是用宿主机网络的情况。
2. 同个宿主机间不同容器互联： 使用命令 `docker network create <your_network_name>`，创建之后docker network ls：
```
[root@your_machine docker]# docker network ls
NETWORK ID NAME DRIVER SCOPE
f8aba8ff1d1b <your_network_name> bridge local
5e04bc31c042 bridge bridge local
cae6189e0ca7 host host local
26d10f20625e none null local
54bef593d492 test bridge local
```
docker run起容器的时候指定 –net=<your_network_name>，以及容器name，在另外一个容器内根据name和端口就可以访问。

宿主机要访问容器端口的话需要在docker run的时候指定端口映射 -p
3. 跨主机：简单的少数机器的跨主机通信方案可以通过把对方的docker网段加到本机路由表以实现互联。多台机器或者跨子网的通信方案有二层VLAN网络和Overlay网络。二层VLAN使用二层网络设备把原先的网络架构改造为互通的大二层网络，通过网络设备直接路由。Overlay网络把二层保温封装在IP保温上，利用IP路由协议进程数据分发。overlay网络不需要硬件支持，并采用扩展的隔离标识位数，能够突破VLAN的4000数量限制支持高达16M的用户，并在必要时可将广播流量转化为组播流量，避免广播数据泛滥，是目前主流的容器跨界点数据传输和路由方案。还可以使用各种第三方工具进行网络互连，使用的是自己的网络协议，包括Weave、Kubernetes、CoreOS, Flannel、Pipework以及SocketPlane等。docker 1.12把overlay网络集成进来，下面实际操作加路由表、overlay互联以及第三方工具三种手段：
    3.1 overlay网络虚拟出一个网络比如10.0.2.3这个ip地址。在这个overlay网络模式里面，有一个类似于服务网关的地址，然后把这个包转发到物理服务器地址，最终通过路由和交换，到达另一个服务器的ip地址。![](http://wiki.baidu.com/download/attachments/452613663/2b2709b5a9ca5f069cd7ce1307759cda.png?version=1&modificationDate=1515655455000&api=v2)
    3.2 要实现overlay网络，我们需要一个服务发现程序，比如consul，此外还需要定义一个ip地址池，比如10.0.2.0/24。容器的ip地址从这个地址池中获取，获取之后通过本机网卡（比如ens33）来进行通信，以实现跨主机的通信。overlay网络有二种模式，一个是swarm模式，一个是非swarm模式，在非swarm模式下使用则需要借助服务发现第三方组件。下面介绍非swarm模式的使用，swarm模式在只需要在swarm集群上运行类似的操作即可。 
![](http://wiki.baidu.com/download/attachments/452613663/3c32b090307c0d85d592586a59e8c619.png?version=1&modificationDate=1515655524000&api=v2)  
    3.3 overlay方案非swarm模式实践（[http://tonybai.com/2016/02/15/understanding-docker-multi-host-networking/](http://tonybai.com/2016/02/15/understanding-docker-multi-host-networking/)）  
 服务发现为consul，docker版本为1.12.6，机器为nodeA和nodeB；

        3.3.1 修改要连接的所有宿主机的docker配置文件（centos7上面是/usr/lib/systemd/system/docker.service）在ExecStart最后面加上

```
-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock \
--cluster-store=consul://<consul_ip>:8500 --cluster-advertise=<网卡>:2375
```
这两行，以让docker daemon监听tcp协议2375端口，consul_ip是consul服务的ip，<网卡>是当前节点的ip地址对应的网卡，在这次试验中为ens11。

–cluster-store= 参数指向docker daemon所使用key value service的地址
–cluster-advertise= 参数决定了所使用网卡以及docker daemon端口信息

        3.3.2 重启docker daemon（root用户）
```
systemctl daemon-reload
systemctl restart docker
```

        3.3.3 在一台机器上启动consul服务，可以用docker起

`docker run -d -p 8500:8500 -h consul --name consul progrium/consul -server -bootstrap`

        3.3.4 在同一台机器上创建类型为overlay的network

`docker network create -d overlay ov_net2`

        3.3.5 在两台机器上分别启动容器并设置网络为ov_net2（上一步创建的docker network，–network=ov_net2）

启动之后进去容器中，用名字就可以ping通了
```
[root@92f59872b5ff /]# hostname -i
10.0.0.3 172.18.0.2
[root@92f59872b5ff /]# ping test-net3
PING test-net3 (10.0.0.2) 56(84) bytes of data.
64 bytes from test-net3.ov_net2 (10.0.0.2): icmp_seq=1 ttl=64 time=0.233 ms
64 bytes from test-net3.ov_net2 (10.0.0.2): icmp_seq=2 ttl=64 time=0.167 ms
```

在创建完一个overlay网络之后，通过docker network ls可以看到网络中不仅多了一个我们创建的 ov_net2 （类型为overlay、scope为global），还能看到一个名为 docker_gwbridge （类型为bridge、scope为local）。这其实就是 overlay 网络的工作原理所在。通过brctl show可以看出，每创建一个网络类型为overlay的容器，则docker_gwbridge下都会挂载一个vethxxx，这说明确实overlay容器是通过此网桥进行对外连接的。简单的说 overlay 网络数据还是从 bridge 网络docker_gwbridge出去的，但是由于consul的作用（记录了overlay网络的endpoint、sandbox、network等信息），使得docker知道了此网络是 overlay 类型的，这样此overlay网络下的不同主机之间就能够相互访问，但其实出口还是在docker_gwbridge网桥。

    3.4 weave方式互联（第三方工具）

        3.4.1 机器A、B安装weave：
```
wget -O /usr/local/bin/weave https://raw.githubusercontent.com/zettio/weave/master/weave
chmod a+x /usr/local/bin/weave
```

        3.4.2 机器A：启动weave并启动容器加入该weave
```
weave launch
docker run -itd --name=my-test1 centos /bin/bash
weave attach 10.0.0.1/24 my-test1
```
        3.4.3.机器B：启动weave连到机器A
```
weave launch <A_ip>
docker run -itd --name=my-test2 centos /bin/bash
weave attach 10.0.0.2/24 my-test2
```
        3.4.4.分别进入容器，可以ping通对方ip  
     3.5 把对方网段加入本机路由表：  
 A、B机器：
    - A:192.168.124.51
    - B:192.168.124.52
更改虚拟机docker0网段，修改为并重启服务

    - A：172.17.1.1/24
    - B：172.17.2.1/24

```
#A 
ifconfig docker0 172.17.1.1 netmask 255.255.255.0 
#B 
ifconfig docker0 172.17.2.1 netmask 255.255.255.0
```

然后在A，B上把对方的docker0网段加入到自己的路由表中

```
#A 
route add -net 172.17.2.0 netmask 255.255.255.0 gw 192.168.124.52 
iptables -t nat -F POSTROUTING 
iptables -t nat -A POSTROUTING -s 172.17.1.0/24 ! -d 172.17.0.0/16 -j MASQUERADE 
#B
route add -net 172.17.1.0 netmask 255.255.255.0 gw 192.168.124.51 
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -s 172.17.2.0/24 ! -d 172.17.0.0/16 -j MASQUERADE
```

然后A、B上创建容器，在容器中就可以通过对方IP地址互联了。


## 五、网络损耗

1. Overlay网络  
 overlay网络的性能损耗取决于其实现方式，其中flannel(vxlan模式)、swarm overlay实现的损耗几乎与端口映射持平，但是 docker 1.12版本新加入的 swarm overlay 实现性能损耗高达60%(swarm overlay代码实现质量不高)。因此，在产生环境下不建议使用swarm overlay方案。
2. Calico和Weave  
 这两种实现的方式跟overlay不太一样，它会把每台宿主机都当成一个路由器使用，数据包在各个主机之间流动最终被投递到目标主机。为了让主机支持路由功能，它们会向路由表中写入大量记录，因些如果集群中的节点太多，路由表记录数过高(超过1万)时会出现性能问题。  
 虽然实现原理一样，但它们的性能区别还是很大的。Calico因为使用的是内核特性，能做到在内核态完成路由，因此性能与原生网络非常接近(90%以上)，而Weave则是在用户态转发数据包，性能较差，损耗高达70%以上。
3. bridge模式、host模式  
host模式是用主机的网络，基本没有性能损失，但是牺牲隔离性。
bridge在非满载下，没什么性能损失，在满载负荷下，桥接存在15%左右的性能损失，即支持并发数是原来的85%。
4. k8s  
 使用Flannel的vxlan模式，会比宿主机直接互联的网络性能损耗30%～40%。而flannel的host-gw模式比起宿主机互连的网络性能损耗大约是10%。


