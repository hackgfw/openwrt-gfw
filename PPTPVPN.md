# 介绍
该文章介绍如何通过VPN翻墙的同时又能使用本地线路访问国内网络。另外对比其他类似方案通常只是基于ip而不考虑端口，造成p2p流量也走vpn，该文章介绍的方法同时使用目标端口做限定条件，p2p流量即使发往国外，也走本地线路。


# 原理
通过策略路由根据目标/源ip及目标端口来决定走vpn线路还是本地线路。将中国ip加入特定的 ipset 中， 在数据包通过 iptables mangle 表时根据源/目标ip及目标端口判断是否走vpn，并打上mark。使用ip rule设定规则，不同的 mark 走不同的路由表，从而实现访问国内ip使用本地线路，访问外国网站使用vpn线路。同时因为使用了 ip-up ip-down 脚本，当VPN断线时会自动切换至本地线路。

# 局限
本地进程发送数据包前需要先选择源IP地址并将其填到数据包中，然后经过netfilter并最终到达网卡，但是策略路由是发生在数据包经过netfilter时，因此之前选择的源IP地址是错误的，需通过NAT更正，但NAT的结果在每个连接的第一个数据包经过netfilter时就已经确定下来了，即使之后路由发生改变，其NAT之后的源IP地址也不会变，导致数据包发往新路由时使用了错误的源IP地址，FORWARD的数据包也有类似的问题。  
目前 gfw-vpn 在vpn连接建立后会清空所有udp连接信息，强制之后的udp包重新NAT，副作用就是即使不需要重新NAT的连接也会被重置，该行为可以通过flush_conntrack选项控制

# 解决方法
 * **使用预编译的 [gfw-vpn](packages) 或根据 [UsePackage](UsePackage.md) 自己编译安装**
 * **在 /etc/config/network 中添加vpn连接**

```
config interface 'wall'
        option proto 'pptp'
        option server 'vpn.example.com'
        option username 'username'
        option password 'password'
        option defaultroute '0'
        option auto '1'
```

替换上面的 server,username,password 为vpn服务器地址、用户名及密码，另外注意上面defaultroute设为0，因为之后会通过 ip-up-wall 脚本添加路由，所以这里不开启默认路由

 * **在 /etc/config/firewall 的wan区域中加入vpn接口，找到类似**

```
config zone
        option name             wan
        option network          'wan'
        option input            REJECT
        option output           ACCEPT
        option forward          REJECT
        option masq             1
        option mtu_fix          1
```

在 network 选项中加入 wall

```
config zone
        option name             wan
        option network          'wan wall'
        option input            REJECT
        option output           ACCEPT
        option forward          REJECT
        option masq             1
        option mtu_fix          1
```

注：如果你看到的是 list network 'wan' 的话，则再加一行 list network 'wall' 即可

 * 可以根据需要添加/修改/删除 /etc/config/gfw-vpn 中的rule，符合rule的数据包会走VPN，目前只支持tcp和udp协议。其中的interface选项为之前添加的VPN接口名称，不同的rule可以走不同的VPN接口（例如：上网走vpn1，游戏走vpn2）
 * 可以根据需要把不翻墙的源ip或目标ip加入 /etc/config/gfw-vpn.whiteip ，例如

```
192.168.1.129
65.55.58.201
```

上面表示所有从192.168.1.129发起的流量以及所有发往65.55.58.201的流量都强制走本地线路。如果不需要的话可以留空

# 后记
 * 上述脚本在每个数据包通过时都会判断条件，如果使用 CONNMARK 仅在连接建立的时候判断条件会导致vpn建立之前就已经存在的连接一直使用本地线路，达不到翻墙的效果
 * **注：如果你同时还在使用multiwan的话可能需要修改 ip-up-wall 脚本中的mark以兼容multiwan**
