# 介绍
该文章介绍如何通过VPN翻墙的同时又能使用本地线路访问国内网络。另外对比其他类似方案通常只是基于ip而不考虑端口，造成p2p流量也走vpn，该文章介绍的方法同时使用目标端口做限定条件，p2p流量即使发往国外，也走本地线路。


# 前期准备
 * 一台Openwrt路由器
 * 安装 pptp 客户端 (opkg install ppp-mod-pptp) (10.03旧版请用 opkg install pptp)
 * 安装 iproute2 (opkg install ip)
 * 安装 ipset (opkg install ipset)
 * 安装 iptables 的 MARK 模块 (opkg install iptables-mod-ipopt)


# 原理
通过策略路由根据目标/源ip及目标端口来决定走vpn线路还是本地线路。将中国ip加入特定的 ipset 中， 在数据包通过 iptables mangle 表时根据源/目标ip及目标端口判断是否走vpn，并打上mark。使用ip rule设定规则，不同的 mark 走不同的路由表，从而实现访问国内ip使用本地线路，访问外国网站使用vpn线路。同时因为使用了 ip-up ip-down 脚本，当VPN断线时会自动切换至本地线路。


# 解决方法
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

替换上面的 server,username,password 为vpn服务器地址、用户名及密码，另外注意上面defaultroute设为0，因为之后会通过脚本添加路由，所以这里不开启默认路由

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

 * **下载中国的ip列表并存到 /etc/config/cn.zone**

```
wget -O /etc/config/cn.zone http://www.ipdeny.com/ipblocks/data/countries/cn.zone
```

注：在写该文章时，这个地址下到的中国ip列表不全，建议用 apnic 的ip地址分配信息生成 cn.zone，或者下载我预生成的 [cn.zone](cn.zone)

 * **创建 /etc/config/ip.whitelist 把不需要翻墙的源ip或目标ip添入其中，例如**

```
192.168.1.129
65.55.58.201
```

上面表示所有从192.168.1.129发起的流量以及所有发往65.55.58.201的流量都强制走本地线路。如果不需要的话可以留空

 * **创建 /etc/ppp/ip-up.d/ip-up-wall ，将该文件设置成可执行，并输入如下内容**

```bash
ip_up_wall() {
        ip route del default via $PPP_REMOTE

        CHKIPSET=$(ipset -L china | wc -l)
        if [ "$CHKIPSET" == "0" ]; then
                ipset -N china nethash --hashsize 32768

                # for IP in $(wget -O - http://www.ipdeny.com/ipblocks/data/countries/cn.zone)
                for IP in $(cat /etc/config/cn.zone)
                do
                        ipset -A china $IP
                done

                ipset -A china 10.0.0.0/8
                ipset -A china 172.16.0.0/12
                ipset -A china 192.168.0.0/16
        fi

        CHKIPSET=$(ipset -L whiteip | wc -l)
        if [ "$CHKIPSET" == "0" ]; then
                ipset -N whiteip iphash --hashsize 128
                for IP in $(cat /etc/config/ip.whitelist)
                do
                        ipset -A whiteip $IP
                done
        fi

        iptables -t mangle -A PREROUTING -p tcp -m set --match-set ! whiteip src -m set --match-set ! whiteip dst -m set --match-set ! china dst -m multiport --dports 80,443,1935 -j MARK --set-mark 0xffff
        iptables -t mangle -A OUTPUT -p tcp -m set --match-set ! whiteip src -m set --match-set ! whiteip dst -m set --match-set ! china dst -m multiport --dports 80,443,1935 -j MARK --set-mark 0xffff
        iptables -t mangle -A FORWARD -p tcp -m set --match-set ! whiteip src -m set --match-set ! whiteip dst -m set --match-set ! china dst -m multiport --dports 80,443,1935 -j MARK --set-mark 0xffff

        CHKIPROUTE=$(grep wall /etc/iproute2/rt_tables)
        if [ -z "$CHKIPROUTE" ]; then
                echo "11 wall" >> /etc/iproute2/rt_tables
        fi

        ip route add table wall default via $PPP_LOCAL
        ip rule add fwmark 0xffff table wall priority 1
}

if [ "$PPP_IFACE" == "pptp-wall" ]; then
        ip_up_wall
fi
```

该脚本会在vpn建立时执行，建立必要的ipset、iptables、ip route及rule规则，只有目标端口为80,443,1935(部分youtube视频会用到这个端口)的流量才会走vpn，否则即使发往国外，也走本地线路。
**注：如果你没有根据 [AntiDNSPoisoning](AntiDNSPoisoning.md) 设置防 DNS 污染，则这里需要把 udp 53 端口加进来并使用国外DNS**

 * **创建 /etc/ppp/ip-down.d/ip-down-wall ，将该文件设置成可执行，并输入如下内容**

```bash
ip_down_wall() {
        iptables -t mangle -D PREROUTING -p tcp -m set --match-set ! whiteip src -m set --match-set ! whiteip dst -m set --match-set ! china dst -m multiport --dports 80,443,1935 -j MARK --set-mark 0xffff
        iptables -t mangle -D OUTPUT -p tcp -m set --match-set ! whiteip src -m set --match-set ! whiteip dst -m set --match-set ! china dst -m multiport --dports 80,443,1935 -j MARK --set-mark 0xffff
        iptables -t mangle -D FORWARD -p tcp -m set --match-set ! whiteip src -m set --match-set ! whiteip dst -m set --match-set ! china dst -m multiport --dports 80,443,1935 -j MARK --set-mark 0xffff

        ip route del table wall default
        ip rule del priority 1
}

if [ "$PPP_IFACE" == "pptp-wall" ]; then
        ip_down_wall
fi
```

该脚本会在vpn断开时执行，删除必要的iptables、ip route及rule规则


# 后记
 * 上述脚本在每个数据包通过时都会判断条件，也许修改成仅在连接建立的时候判断条件并使用 CONNMARK 来标记连接可以提高性能，不过该脚本在TP-Link WR841N上运行并没有遇到性能问题，因此也没有试过 CONNMARK 是否可行
 * **注：如果你同时还在使用multiwan的话可能需要修改上面的mark以 兼容multiwan**
