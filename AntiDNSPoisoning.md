# 介绍
由于GFW的域名黑名单是不断变化的，如果所有DNS查询都走VPN会丧失CDN加速功能，经常出现本地电信线路，但是却访问网站的联通线路，而且当VPN线路不稳定的时候，会影响所有网站的访问。如果使用域名白名单或黑名单，对于经常访问外国网站的用户，体验很不好，需要用户自己维护域名列表。
该文章介绍如何通过判断并丢弃包含特征IP的DNS包来防止DNS污染，但是又不会失去本地DNS的CDN加速功能，而且还不需要VPN。


## 前期准备
  * 一台Openwrt路由器
  * 使用dnsmasq作为Openwrt的dns服务器
  * 安装 dig (opkg install bind-dig bind-libs)
  * 安装 iptables 的 string 模块及对应的内核模块 (opkg install iptables-mod-filter kmod-ipt-filter)
  * 安装 iptables 的 u32 模块及对应的内核模块 (opkg install iptables-mod-u32 kmod-ipt-u32)


# 原理
分析：
 * GFW污染掉的域名返回的IP是有限的，即使用不提供的DNS服务的国外IP地址作为DNS服务器查询被污染的域名，GFW也会返回解析出的错误IP地址。
 * 使用提供DNS服务的国外IP地址作为DNS服务器查询被污染的域名，GFW会返回解析出的错误IP地址，但是正确的IP地址也会在随后被返回。
 * 使用本地ISP提供的DNS查询被污染的域名，由于DNS的逐级查询，而且域名在国外，查询过程会通过GFW，因此查到的结果也是被GFW污染的，但是和上面不同，这次不会返回正确的IP
 * 另外GFW除了会返回被污染的IP同时，还有可能返回不包含任何查询结果的应答,Answer RRs 和 Authority RRs 均为0. 这个正常情况是不会出现的。即使查询完全不存在的域名，其应答也会包含根服务器的Authority记录

解决思路：
 * 为了使用本地CDN，DNS列表需要同时包含本地ISP提供的DNS服务器和国外的DNS比如8.8.8.8，并将dnsmasq设置成同时查询所有服务器
 * 多次查询被污染的域名，获得GFW返回的错误IP列表
 * 使用iptables判断DNS数据包是否包含被污染的IP，如果是则直接丢弃
 * 使用iptables判断DNS数据包是否不包含任何查询结果，如果是则直接丢弃

好处：
 * 只要有任意一个本地DNS服务器的返回速度快于国外的（通常本地DNS的返回速度都是会快于国外的，毕竟距离在那里摆着的），查询的结果就是经过CDN加速的
 * 如果查的是被污染的域名，包含被污染IP的数据包会被丢弃，最终会由国外DNS返回正确的IP，可以避免DNS污染
 * DNS查询不需要通过VPN通道，即使VPN再不稳定，用户也可以正常查询DNS

坏处：
 * 一开始为了获得GFW返回的错误IP列表，多次查询被污染的域名，耗时过长

# 解决方法
 * **创建 /etc/config/resolv.conf ，将你的本地dns服务器和国外dns服务器都添加进去，下面是例子（注意需要将其中的DNS服务器替换成你的ISP提供的）**

```
nameserver 202.96.199.133
nameserver 202.97.16.195
nameserver 8.8.8.8
nameserver 8.8.4.4
```

 * **修改 /etc/config/dhcp ，让dnsmasq使用 /etc/config/resolv.conf 中的服务器,并将ISP的DNS返回不存在域名的IP地址加入黑名单**

```
option resolvfile       '/etc/config/resolv.conf'
list bogusnxdomain     '63.251.179.17'
```

 你需要将上面的 63.251.179.17 替换掉。比如你向ISP的DNS查询 non.exist.domain，按规范应该返回NXDOMAIN，表示该域名不存在，但是运营商通常会返回一个他自己的地址，然后将你导向到其广告页。你需要将运营商返回的那个IP添加到bogusnxdomain中。
 * **修改 /etc/dnsmasq.conf ，使得dnsmasq同时查询所有dns服务器，添加一行**

```
all-servers
```

 * **创建 /etc/protectdns.sh ，将该文件设置成可执行，并输入如下内容**

```
#!/bin/sh

POSIONEDDOMAIN="www.twitter.com twitter.com www.facebook.com facebook.com www.youtube.com youtube.com encrypted.google.com plus.google.com www.appspot.com appspot.com www.openvpn.net openvpn.net forums.openvpn.net svn.openvpn.net shell.cjb.net"
LOOPTIMES=1
RULEFILENAME=/var/g.firewall.user

# wait tail has internet connection
while ! ping -W 1 -c 1 8.8.8.8 >&/dev/null; do sleep 30; done

badip=""

querydomain=""
matchregex="^${POSIONEDDOMAIN//\ /|^}"
for i in $(seq $LOOPTIMES) ; do
        querydomain="$querydomain $POSIONEDDOMAIN"
done

for DOMAIN in $POSIONEDDOMAIN ; do
        for IP in $(dig +time=1 +tries=1 +retry=0 @$DOMAIN $querydomain | grep -E "$matchregex" | grep -o -E "([0-9]+\.){3}[0-9]+") ; do
                if [ -z "$(echo $badip | grep $IP)" ] ; then
                        badip="$badip   $IP"
                fi
        done
done

for IP in $badip ; do
        hexip=$(printf '%02X ' ${IP//./ }; echo)
        echo "iptables -I INPUT -p udp --sport 53 -m string --algo bm --hex-string \"|$hexip|\" --from 60 --to 180  -j DROP" >> $RULEFILENAME.tmp 
        echo "iptables -I FORWARD -p udp --sport 53 -m string --algo bm --hex-string \"|$hexip|\" --from 60 --to 180 -j DROP" >> $RULEFILENAME.tmp
done

echo "iptables -I INPUT -p udp --sport 53 -m u32 --u32 \"4 & 0x1FFF = 0 && 0 >> 22 & 0x3C @ 8 & 0x8000 = 0x8000 && 0 >> 22 & 0x3C @ 14 = 0\" -j DROP" >> $RULEFILENAME.tmp
echo "iptables -I FORWARD -p udp --sport 53 -m u32 --u32 \"4 & 0x1FFF = 0 && 0 >> 22 & 0x3C @ 8 & 0x8000 = 0x8000 && 0 >> 22 & 0x3C @ 14 = 0\" -j DROP" >> $RULEFILENAME.tmp

if [[ -s $RULEFILENAME ]] ; then
        grep -Fvf $RULEFILENAME $RULEFILENAME.tmp > $RULEFILENAME.action
        cat $RULEFILENAME.action >> $RULEFILENAME
else
        cp $RULEFILENAME.tmp $RULEFILENAME
        cp $RULEFILENAME.tmp $RULEFILENAME.action
fi

. $RULEFILENAME.action
rm $RULEFILENAME.tmp
rm $RULEFILENAME.action
```

 由于上面脚本在查询GFW返回的错误IP时，使用了被污染的域名本身作为DNS服务器，因此如果以后GFW将该域名解禁，查询将不返回任何结果（但是会超时，使得耗时更长），并不会影响badip列表，只需要在开始把足够多的被污染域名填入POSIONEDDOMAIN即可。放入POSIONEDDOMAIN的域名应保证该域名以后不会提供DNS解析服务，同时该域名也不太容易被GFW解封
 另外iptables的操作都存到了/var/g.firewall.user，以方便在防火墙重启的时候重新添加规则
 * **修改 /etc/config/firewall 加入**

```
config include
        option path /var/g.firewall.user
```

 以便在防火墙重启后重新添加规则
 * **创建启动脚本 /etc/init.d/protectdns 以便随系统自动启动**

```
#!/bin/sh /etc/rc.common

START=99
RULEFILENAME=/var/g.firewall.user

start() {
        /etc/protectdns.sh &
        }

stop() {
        local pid
        pid=`ps w | grep protectdns.sh | grep -v grep | awk '{print $1}'`
        for i in $pid;do kill -9 $i;done
        [[ -s $RULEFILENAME ]] && {
                sed -ie 's/iptables \+-I \+/iptables -D /' $RULEFILENAME
                . $RULEFILENAME
                rm $RULEFILENAME
                }
        }
```

 执行 /etc/init.d/protectdns enable 以将该脚本设置成自动启动，执行 /etc/init.d/protectdns stop 会移除所有已添加的iptables规则并将 /var/g.firewall.user 删除，执行 /etc/init.d/protectdns start 会查找新的被污染IP并添加iptables规则
 * **为了定时更新，可以在 /etc/crontabs/root 加入**

```
0 6 * * * /etc/init.d/protectdns start
```

 上面的例子是在每天凌晨6点重新查找新的被污染IP，你可以根据需要修改更新频率和时间

# 后记
 * 如果你使用VPN，而且VPN服务器推送DNS的话，可以不修改/etc/config/resolv.conf 和 /etc/config/dhcp，dnsmasq会自动使用本地DNS和VPN推送的DNS
