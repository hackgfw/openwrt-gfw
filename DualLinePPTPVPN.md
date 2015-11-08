# 介绍
通常VPN连接都是单线的，而中美线路经常抽风，如果VPN线路遇到丢包或断线会非常影响翻墙体验。该文章介绍如何通过两条冗余的VPN连接来避免断线及丢包


# 前期准备
 * 两台国外服务器，均需要有root权限以修改服务器设置。OpenVZ服务器即可，不过需要OpenVZ加载对应的内核模块。另外还需要公网ip或位于Stateless NAT后，即只映射ip而不改变端口（例如ec2的NAT）
 * 一台运行 Openwrt 12.09、14.07或15.05的路由器且必须能取得公网IP (位于Stateless NAT后或许也可以，不过未测试过)
 * 给Openwrt使用的内核pptp打补丁，详见[OptimizePPTPVPN](OptimizePPTPVPN.md)


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
  3. 客户端到主VPN服务器的丢包率是10%，延迟是150ms，客户端到辅VPN服务器的丢包率是0%，延迟是550ms，最终客户端到互联网的丢包率将为10%，延迟将保持在150ms，除非客户端到主VPN服务器完全断线。之所以如此是因为pptp默认最多缓存300ms内收到的数据包信息。


# 解决方法
 * **使用预编译的 [gfw-dualpptp](packages) 或根据 [UsePackage](UsePackage.md) 自己编译安装到Openwrt路由器上**
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
 * **在辅VPN服务器上创建 /etc/ppp/ip-up.d/ip-up-dual ，将该文件设置成可执行，并输入如下内容**

注：该脚本允许多个主VPN服务器共用同一个辅VPN服务器

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

dual() {
        CHKIPROUTE=$(grep dual /etc/iproute2/rt_tables)
        if [ -z "$CHKIPROUTE" ]; then
                echo "30 dual" >> /etc/iproute2/rt_tables
        fi

        ifconfig $PPP_IFACE mtu 1448

        iptables -t mangle -I PREROUTING -i $PPP_IFACE -j MARK --set-mark 0xf222
        ip route add table dual $PPP_IPPARAM via $PPP_LOCAL dev $PPP_IFACE
        ip rule add fwmark 0xf222 table dual priority 2
}

if [[ "$PPP_REMOTE" != "${PPP_REMOTE/10.66.4.6/}" ]]; then
        dual
fi
```
 * **在辅VPN服务器上创建 /etc/ppp/ip-down.d/ip-down-dual ，将该文件设置成可执行，并输入如下内容**

```bash
#!/bin/bash

# These variables are for the use of the scripts run by run-parts
PPP_IFACE="$1";
PPP_TTY="$2";
PPP_SPEED="$3";
PPP_LOCAL="$4";
PPP_REMOTE="$5";
PPP_IPPARAM="$6";

deldual() {
        iptables -t mangle -D PREROUTING -i $PPP_IFACE -j MARK --set-mark 0xf222
        ip route del table dual $PPP_IPPARAM
        ip rule del fwmark 0xf222 table dual priority 2
}

if [[ "$PPP_REMOTE" != "${PPP_REMOTE/10.66.4.6/}" ]]; then
        deldual
fi
```
 * **参照 [PPTPVPN](PPTPVPN.md) 在Openwrt上创建主/辅VPN链接（可以使用gfw-vpn中的VPN链接作为主VPN，这样只需要创建辅VPN即可），主VPN使用client帐号，辅VPN使用cdual帐号，并将主VPN链接加入到防火墙的wan区域（辅VPN链接不要加入到任何区域中），并在 /etc/config/gfw-vpn 中设置相应的规则**
 * **修改Openwrt上的 /etc/config/gfw-dualpptp 使主/辅VPN链接名称和上一步设置中的一致**
 * **全部设置好后重启主/辅VPN服务器和Openwrt即可使用双线VPN了**

# 后记
 * 在实际使用中可能需要调整上面脚本及配置中的mtu值以保证ip包不会被拆分, 或因为mtu过小而被丢弃
 * 为了更好的性能，也可以给服务器打内核补丁并使用 accel-pptp 作为pptp服务器和客户端。由于服务器的版本众多，如果你感兴趣的话可以手动移植 [kernel_patch](kernel_patch) 到你所使用的服务器上
 * **注：如果你同时还在使用multiwan的话可能需要修改Openwrt上 ip-up-wall 脚本中的mark以兼容multiwan**
