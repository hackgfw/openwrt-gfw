# 介绍
通常VPN连接都是单线的，而中美线路经常抽风，如果VPN线路遇到丢包或断线会非常影响翻墙体验。该文章介绍如何通过两条冗余的VPN连接来避免断线及丢包


# 前期准备
 * 两台国外服务器，均需要有root权限以修改服务器设置。OpenVZ服务器即可，不过需要OpenVZ加载对应的 内核模块。另外还需要公网ip或位于Stateless NAT后，即只映射ip而不改变端口（例如ec2的NAT）
 * 一台Openwrt路由器且必须能取得公网IP (位于Stateless NAT后或许也可以，不过未测试过)
 * 安装 pptp 客户端 (opkg install pptp) (这里指的是 http://pptpclient.sourceforge.net/ ，新版Openwrt使用的是内核pptp，不确定是否同样适用)
 * 安装 iptables 的 u32 模块及对应的内核模块 (opkg install iptables-mod-u32 kmod-ipt-u32)
 * 安装 iptables 的 tee 模块及对应的内核模块 (opkg install iptables-mod-tee kmod-ipt-tee) (该模块是2.6.35版引入内核的，如果使用旧版内核的话也许可以用 http://xtables-addons.sourceforge.net/ 不过本人并未测试过)


# 原理
通常的VPN线路:
```
客户端----------VPN服务器---------互联网
```
由于客户端到VPN服务器的线路是跨国界的，经常遇到断线丢包，而VPN服务器到互联网的连接质量相比之下要好很多，所以重点就是如何保证客户端到VPN服务器的连接质量。该文的目标是通过建立冗余的客户端到VPN服务器线路以应对断线和丢包：
```
客户端------------主VPN服务器-----------互联网
  |                    |
  ----------------辅VPN服务器
```
主VPN服务器和辅VPN服务器的区别是谁来作为访问互联网的出口。客户端------主VPN服务器----互联网 这部分和普通的VPN连接是相同的，只是客户端还会建立一条到辅VPN服务器的VPN连接，并且把所有发往主VPN服务器的数据通过tee模块抄送一份到辅VPN服务器，而辅VPN服务器收到数据后，会通过主VPN服务器和辅VPN服务器之间预先建立好的VPN连接将数据包路由到主VPN服务器，这样主VPN服务器就会收到两份数据，同理客户端也会收到两份主VPN服务器的数据，最终客户端到互联网的延迟将是两条链路中延迟最低的那条，而丢包也将极大的改善。

具体来讲这是依靠pptp程序对于乱序数据包重组来实现的。在两条线路都没有丢包的情况下，或者只有延迟高的线路丢包，那么pptp会直接将后收到的重复数据包丢弃。但是如果延迟低的线路有丢包，比如现在pptp从延迟低的线路上收到了标号为1、2、4、5的数据包，但是没有收到3号数据包，pptp会把4、5号数据包已收到但3号包未收到这3条信息保存下来，如果之后还陆续收到6、7、8、9号包，那么需要保存的信息就越来越多，因此pptp默认只保存最近300ms的信息，如果3号包超过300ms后才从延迟高的线路上，则3号包将直接被丢弃。

举例来说吧（以下均假设两条线路的丢包率不相关，即丢包并非因为本地带宽不足或其他等因素）：
  1. 客户端到主VPN服务器的丢包率是10%，延迟是150ms，客户端到辅VPN服务器的丢包率是0%，延迟是250ms，那么最终客户端到互联网将没有丢包，延迟会在150ms到250ms之间波动。如果客户端到主VPN服务器的丢包率上升到100%即断线，那么延迟将稳定在250ms
  2. 客户端到主VPN服务器的丢包率是10%，延迟是150ms，客户端到辅VPN服务器的丢包率是10%，延迟是250ms,那么最终客户端到互联网的丢包率=1-10%*10%=1%，延迟会在150ms到250ms之间波动。
  3. 客户端到主VPN服务器的丢包率是10%，延迟是150ms，客户端到辅VPN服务器的丢包率是0%，延迟是550ms，最终客户端到互联网的丢包率将为10%，延迟将保持在150ms，除非客户端到主VPN服务器完全断线。之所以如此是因为pptp默认最多缓存300ms内收到的数据包信息。这个300ms是定义在pqueue.h中的

```c
/* wait this many seconds for missing packets before forgetting about them */
#define DEFAULT_PACKET_TIMEOUT 0.3
```
可以在调用pptp时指定 --timeout 参数以调整缓存时间

**注：新版的openwrt 12.09使用的是内核pptp，我没有查源码，不确定该文章是否依然适用，不过鉴于[wiki](http://wiki.openwrt.org/doc/uci/network#protocol.pptp.point-to-point.tunneling.protocol)中有提到buffering，因此我估计该文可能同样适用**


# 解决方法
注：双线VPN是去年为玩D3而建的，因此下面脚本均以D3所用端口为例，如需改成上网请参照 [PPTPVPN](PPTPVPN.md)。
 * **修改所有主机（主/辅VPN服务器和Openwrt）上的 /etc/sysctl.conf 加入**

```
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0
```
 * **在主VPN服务器上建立VPN服务，设定分配地址为10.66.6.0/24，**
 * **在主VPN服务器上建立到辅VPN拨号，创建 /etc/ppp/peers/bvpn 并输入**

```
pty "pptp secondary.example.com --nolaunchpppd"
noauth
require-mschap-v2
require-mppe-128
name sdual
nodefaultroute
maxfail 0
holdoff 30
persist
remotename bvpn
ipparam bvpn
file /etc/ppp/options.pptp
```
替换 secondary.example.com 为辅VPN的IP地址
 * **在主VPN服务器上设定自动拨号到辅VPN并预先建立iptables chain，在 /etc/rc.local 中加入**

```
iptables -t mangle -N dupgre
pon bvpn
```
 * **在主VPN服务器上建立VPN帐号，在 /etc/ppp/chap-secrets 中加入**

```
sdual           *       password                   *
client          *       password                   10.66.6.66
```
上面的password是密码,可以替换成其他值,不过需要保持和辅VPN及客户端的密码一致
 * **在主VPN服务器上创建 /etc/ppp/ip-up.d/ip-up-client ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

addduprule() {
        ifconfig $PPP_IFACE mtu 1380
        iptables -t mangle -I OUTPUT -d $PPP_IPPARAM -j dupgre
}

if [ "$PPP_REMOTE" == "10.66.6.66" ]; then
        addduprule
fi
```
 * **在主VPN服务器上创建 /etc/ppp/ip-down.d/ip-down-client ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

delduprule() {
        iptables -t mangle -D OUTPUT -d $PPP_IPPARAM -j dupgre
}

if [ "$PPP_REMOTE" == "10.66.6.66" ]; then
        delduprule
fi
```
 * **在主VPN服务器上创建 /etc/ppp/ip-up.d/ip-up-dupgre ，将该文件设置成可执行，并输入如下内容**

注：因为我用了ec2 ubuntu 12.04做主VPN，因此需要NAT内网和外网ip，把10.123.82.34替换成ec2的内网ip，把34.123.238.192替换成ec2的外网ip,另外还需要安装 xtable-addons 以便使用RAWSNAT和RAWDNAT，如果你的服务器是外网ip请把 "# fix ec2 internal address" 后面几行删除

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

dupgre() {
ifconfig $PPP_IFACE mtu 1448

# fix ec2 internal address
iptables -t raw -I OUTPUT -o $PPP_IFACE -s 10.123.82.34 -j RAWSNAT --to-source 34.123.238.192
iptables -t raw -I OUTPUT -o $PPP_IFACE -j NOTRACK
iptables -t raw -I PREROUTING -i $PPP_IFACE -d 34.123.238.192 -j RAWDNAT --to-destination 10.123.82.34
iptables -t raw -I PREROUTING -i $PPP_IFACE -j NOTRACK

iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8>>24=0xFD" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12>>24=0xFD" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12>>24=0xFD" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16>>24=0xFD" -j TEE --gateway $PPP_REMOTE

iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8=0xFF0300FD" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12=0xFF0300FD" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12=0xFF0300FD" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16=0xFF0300FD" -j TEE --gateway $PPP_REMOTE

iptables -t mangle -I dupgre -p tcp --sport 1723 -m u32 --u32 "0>>22&0x3C@12>>26&0x3C@2>>16=0x1 && 0>>22&0x3C@12>>26&0x3C@8>>16=0x5:0x6" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p tcp --dport 1723 -m u32 --u32 "0>>22&0x3C@12>>26&0x3C@2>>16=0x1 && 0>>22&0x3C@12>>26&0x3C@8>>16=0x5:0x6" -j TEE --gateway $PPP_REMOTE

iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE

iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8=0xFF03C021 && 0>>22&0x3C@12>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12=0xFF03C021 && 0>>22&0x3C@16>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12=0xFF03C021 && 0>>22&0x3C@16>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
iptables -t mangle -I dupgre -p 47 -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF && 0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16=0xFF03C021 && 0>>22&0x3C@20>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
}


if [ "$PPP_IPPARAM" == "bvpn" ]; then
        dupgre
fi
```
 * **在主VPN服务器上创建 /etc/ppp/ip-down.d/ip-down-dupgre ，将该文件设置成可执行，并输入如下内容**
**注：如果用ec2的话同样需要将下面的ip替换掉，否则请把 "# fix ec2 internal address" 后面几行删除**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

deldupgre() {
iptables -t mangle -F dupgre

# fix ec2 internal address
iptables -t raw -D OUTPUT -o $PPP_IFACE -s 10.123.82.34 -j RAWSNAT --to-source 34.123.238.192
iptables -t raw -D OUTPUT -o $PPP_IFACE -j NOTRACK
iptables -t raw -D PREROUTING -i $PPP_IFACE -d 34.123.238.192 -j RAWDNAT --to-destination 10.123.82.34
iptables -t raw -D PREROUTING -i $PPP_IFACE -j NOTRACK

}

if [ "$PPP_IPPARAM" == "bvpn" ]; then
        deldupgre
fi

```
 * **在辅VPN服务器上建立VPN服务，设定分配地址为10.66.4.0/24，**
 * **在辅VPN服务器上建立VPN帐号，在 /etc/ppp/chap-secrets 中加入**

```
sdual           *       password                   10.66.4.67
cdual           *       password                   10.66.4.65
```
上面的password是密码,可以替换成其他值,不过需要保持和主VPN及客户端的密码一致
 * **在辅VPN服务器上创建 /etc/ppp/ip-up.d/ip-up-sdual ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

sdual() {
        CHKIPROUTE=$(grep sdual /etc/iproute2/rt_tables)
        if [ -z "$CHKIPROUTE" ]; then
                echo "76 sdual" >> /etc/iproute2/rt_tables
        fi

        ifconfig $PPP_IFACE mtu 1448

        iptables -t mangle -I PREROUTING -i $PPP_IFACE -j MARK --set-mark 0xf223
        ip route add table sdual default via $PPP_LOCAL dev $PPP_IFACE
        ip rule add fwmark 0xf221 table sdual priority 1
}

if [ "$PPP_REMOTE" == "10.66.4.67" ]; then
        sdual
fi
```
 * **在辅VPN服务器上创建 /etc/ppp/ip-down.d/ip-down-sdual ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

delsdual() {
        iptables -t mangle -D PREROUTING -i $PPP_IFACE -j MARK --set-mark 0xf223
        ip route del table sdual default
        ip rule del priority 1
}

if [ "$PPP_REMOTE" == "10.66.4.67" ]; then
        delsdual
fi
```
 * **在辅VPN服务器上创建 /etc/ppp/ip-up.d/ip-up-cdual ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

cdual() {
        CHKIPROUTE=$(grep cdual /etc/iproute2/rt_tables)
        if [ -z "$CHKIPROUTE" ]; then
                echo "72 cdual" >> /etc/iproute2/rt_tables
        fi

        ifconfig $PPP_IFACE mtu 1448

        iptables -t mangle -I PREROUTING -i $PPP_IFACE -j MARK --set-mark 0xf221
        ip route add table cdual default via $PPP_LOCAL dev $PPP_IFACE
        ip rule add fwmark 0xf223 table cdual priority 2
}

if [ "$PPP_REMOTE" == "10.66.4.65" ]; then
        cdual
fi
```
 * **在辅VPN服务器上创建 /etc/ppp/ip-down.d/ip-down-cdual ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

delcdual() {
        iptables -t mangle -D PREROUTING -i $PPP_IFACE -j MARK --set-mark 0xf221
        ip route del table cdual default
        ip rule del priority 2
}

if [ "$PPP_REMOTE" == "10.66.4.65" ]; then
        delcdual
fi
```
 * **在Openwrt上添加VPN连接，修改 /etc/config/network**

```
config 'interface' 'bwall'
        option 'proto' 'pptp'
        option 'server' 'secondary.example.com'
        option 'username' 'cdual'
        option 'password' 'password'
        option 'defaultroute' '0'
        option 'auto' '1'

config 'interface' 'dwall'
        option 'proto' 'pptp'
        option 'server' 'main.example.com'
        option 'username' 'client'
        option 'password' 'password'
        option 'defaultroute' '0'
        option 'auto' '1'
```
main.example.com 即为主VPN服务器地址，secondary.example.com为辅VPN地址

 * **修改Openwrt上的防火墙，在 /etc/config/firewall 的wan区域中加入主vpn接口，找到类似**

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
在 network 选项中加入 dwall

```
config zone
        option name             wan
        option network          'wan wall dwall'
        option input            REJECT
        option output           ACCEPT
        option forward          REJECT
        option masq             1
        option mtu_fix          1
```
 * **在Openwrt上创建 /etc/ppp/ip-up.d/ip-up-client ，将该文件设置成可执行，并输入如下内容**

```bash
dupgre() {
        ip route del default via $PPP_REMOTE

        ifconfig $PPP_IFACE mtu 1380

        iptables -t mangle -A PREROUTING -p tcp --dport 1119 -j MARK --set-mark 0xfffe
        iptables -t mangle -A OUTPUT -p tcp --dport 1119 -j MARK --set-mark 0xfffe
        iptables -t mangle -A FORWARD -p tcp --dport 1119 -j MARK --set-mark 0xfffe

        CHKIPROUTE=$(grep dupgre /etc/iproute2/rt_tables)
        if [ -z "$CHKIPROUTE" ]; then
                echo "37 dupgre" >> /etc/iproute2/rt_tables
        fi

        ip route add table dupgre default via $PPP_LOCAL
        ip rule add fwmark 0xfffe table dupgre priority 20
}

if [ "$PPP_IFACE" == "pptp-dwall" ]; then
        dupgre
fi
```
上面的1119即为D3所用端口
 * **在Openwrt上创建 /etc/ppp/ip-down.d/ip-down-client ，将该文件设置成可执行，并输入如下内容**

```bash
deldupgre() {
        iptables -t mangle -D PREROUTING -p tcp --dport 1119 -j MARK --set-mark 0xfffe
        iptables -t mangle -D OUTPUT -p tcp --dport 1119 -j MARK --set-mark 0xfffe
        iptables -t mangle -D FORWARD -p tcp --dport 1119 -j MARK --set-mark 0xfffe

        ip route del table dupgre default
        ip rule del priority 20
}

if [ "$PPP_IFACE" == "pptp-dwall" ]; then
        deldupgre
fi
```
 * **在Openwrt上创建 /etc/ppp/ip-up.d/ip-up-cdual ，将该文件设置成可执行，并输入如下内容**

```bash
cdual() {
        ip route del default via $PPP_REMOTE

        ifconfig $PPP_IFACE mtu 1448

        iptables -I INPUT ! -p 47 -i $PPP_IFACE -j DROP
        iptables -I INPUT -p tcp --sport 1723 -i $PPP_IFACE -j ACCEPT
        iptables -I INPUT -p tcp --dport 1723 -i $PPP_IFACE -j ACCEPT
        iptables -I FORWARD -p 47 -o $PPP_IFACE -j ACCEPT
        iptables -I FORWARD -p tcp --sport 1723 -o $PPP_IFACE -j ACCEPT
        iptables -I FORWARD -p tcp --dport 1723 -o $PPP_IFACE -j ACCEPT

        iptables -t mangle -N dupgre
        iptables -t mangle -I OUTPUT -p 47 -d main.example.com -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF" -j dupgre

        iptables -t mangle -N duptcp
        iptables -t mangle -I OUTPUT -p tcp --sport 1723 -d main.example.com -j duptcp
        iptables -t mangle -I OUTPUT -p tcp --dport 1723 -d main.example.com -j duptcp

        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8>>24=0xFD" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12>>24=0xFD" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12>>24=0xFD" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16>>24=0xFD" -j TEE --gateway $PPP_REMOTE

        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8=0xFF0300FD" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12=0xFF0300FD" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12=0xFF0300FD" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16=0xFF0300FD" -j TEE --gateway $PPP_REMOTE

        iptables -t mangle -I duptcp -m u32 --u32 "0>>22&0x3C@12>>26&0x3C@2>>16=0x1 && 0>>22&0x3C@12>>26&0x3C@8>>16=0x5:0x6" -j TEE --gateway $PPP_REMOTE

        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16>>8=0xC02109:0xC0210A" -j TEE --gateway $PPP_REMOTE

        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2001880b && 0>>22&0x3C@8=0xFF03C021 && 0>>22&0x3C@12>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3001880b && 0>>22&0x3C@12=0xFF03C021 && 0>>22&0x3C@16>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x2081880b && 0>>22&0x3C@12=0xFF03C021 && 0>>22&0x3C@16>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
        iptables -t mangle -I dupgre -m u32 --u32 "0>>22&0x3C@0=0x3081880b && 0>>22&0x3C@16=0xFF03C021 && 0>>22&0x3C@20>>24=0x09:0x0A" -j TEE --gateway $PPP_REMOTE
}

if [ "$PPP_IFACE" == "pptp-bwall" ]; then
        cdual
fi
```
把上面的 main.example.com 替换成主VPN服务器的IP地址
 * **在Openwrt上创建 /etc/ppp/ip-down.d/ip-down-cdual ，将该文件设置成可执行，并输入如下内容**

```bash
delcdual() {

        iptables -D INPUT ! -p 47 -i $PPP_IFACE -j DROP
        iptables -D INPUT -p tcp --sport 1723 -i $PPP_IFACE -j ACCEPT
        iptables -D INPUT -p tcp --dport 1723 -i $PPP_IFACE -j ACCEPT
        iptables -D FORWARD -p 47 -o $PPP_IFACE -j ACCEPT
        iptables -D FORWARD -p tcp --sport 1723 -o $PPP_IFACE -j ACCEPT
        iptables -D FORWARD -p tcp --dport 1723 -o $PPP_IFACE -j ACCEPT

        iptables -t mangle -D OUTPUT -p 47 -d main.example.com -m u32 --u32 "0>>22&0x3C@4>>16=0x1:0xFFFF" -j dupgre
        iptables -t mangle -F dupgre
        iptables -t mangle -X dupgre

        iptables -t mangle -D OUTPUT -p tcp --sport 1723 -d main.example.com -j duptcp
        iptables -t mangle -D OUTPUT -p tcp --dport 1723 -d main.example.com -j duptcp
        iptables -t mangle -F duptcp
        iptables -t mangle -X duptcp
}

if [ "$PPP_IFACE" == "pptp-bwall" ]; then
        delcdual
fi
```
把上面的 main.example.com 替换成主VPN服务器的IP地址
 * **全部设置好后重启主/辅VPN服务器和Openwrt即可使用双线VPN了**

# 后记
 * 在实际使用中可能需要调整上面脚本中的mtu值以保证ip包不会被拆分, 或因为mtu过小而被丢弃
 * 之所以使用ec2做主vpn是因为其网络质量非常稳定（相较于其他低端VPS，高端的我没用过不清楚），在几百小时D3游戏时间中除了暴雪自身服务器出问题，从来没有掉线过。
 * **注：如果你同时还在使用multiwan的话可能需要修改Openwrt上的mark以兼容multiwan**
