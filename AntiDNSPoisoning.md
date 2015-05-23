# 介绍
由于GFW的域名黑名单是不断变化的，如果所有DNS查询都走VPN会丧失CDN加速功能，经常出现本地电信线路，但是却访问网站的联通线路，而且当VPN线路不稳定的时候，会影响所有网站的访问。如果使用域名白名单或黑名单，对于经常访问外国网站的用户，体验很不好，需要用户自己维护域名列表。
该文章介绍如何通过自建DNS递归服务器来防止DNS污染，但是又不会失去本地DNS的CDN加速功能。

# 原理
分析：
 * 对于一个DNS请求，递归服务器是从根域名开始查询的，然后再查询顶级域名服务器，再逐级查到所请求的域名，这个过程如果配合[gfw-vpn](PPTPVPN.md)通过VPN连接位于国外的域名服务器可以避免GFW污染，而如果待查询的域名服务器在国内则通过本地线路连接，这样得到的结果也是经过CDN加速的了（国内域名不存在被污染的问题，有关部门可以直接拔网线）。

好处：
 * 只要能建立一条到国外的VPN线路，不管GFW怎么折腾，DNS查询结果都不会被污染，同时也是经过CDN加速的

坏处：
 * DNS查询需要通过VPN通道，如果VPN不稳定，会影响域名解析，进而导致网页打不开
 * DNS解析延迟相比直接使用 114.114.114.114 和 8.8.8.8 之类的公共域名解析服务器要高不少，因为 114.114.114.114 和 8.8.8.8 用的人多，很多域名都在缓存里，可以立即返回结果。

# 解决方法1 - 使用 [fastdns](https://github.com/hackgfw/fastdns)
 * **参照 [UsePackage](UsePackage.md) 编译并安装 [fastdns](https://github.com/hackgfw/fastdns)**
 * **执行 /etc/init.d/fastdns enable 将 fastdns 设置为自动启动**
 * **修改 /etc/config/dhcp 禁用 dnsmasq 的 DNS 解析功能，在选项里加入一行**
```
option port '0'
```
 * **由于关闭 dnsmasq 的 DNS 功能后，dnsmasq 不会在DHCP应答里推送 DNS 服务器，因此还需要在 /etc/config/dhcp 里手动指定 DNS 服务器，找到类似**
```
config dhcp 'lan'
        option interface 'lan'
        option start '64'
        option limit '63'
        option leasetime '12h'
```
 加入一行 list dhcp_option '6,0.0.0.0'
```
config dhcp 'lan'
        option interface 'lan'
        option start '64'
        option limit '63'
        option leasetime '12h'
        list dhcp_option '6,0.0.0.0'
```
 如果还需要指定更多的备用服务器可以用  list dhcp_option '6,0.0.0.0,8.8.8.8'
 * **修改 /etc/config/gfw-vpn 取消注释dns相关的那几行，以便通过VPN连接位于国外的域名服务器**
 * **最后重启路由器**
 * fastdns 的安全性不如 bind, 如果你对安全比较在意, 请使用解决方法2
 * 默认 fastdns 是链接到 libstdcpp(uclibc++性能太差了), 因此安装时需要装两个包, 体积比较大, 如果你之前没有安装过 libstdcpp, 可以考虑静态链接, 将 Makefile 中的
```
DEPENDS += +libstdcpp +librt
```
替换为
```
DEPENDS += +librt
```
再将
```
CXXFLAGS="$$$$CXXFLAGS -DNDEBUG -fno-exceptions -fno-builtin -fno-rtti" \
```
替换为
```
CXXFLAGS="$$$$CXXFLAGS -DNDEBUG -fno-exceptions -fno-builtin -fno-rtti -static-libstdc++" \
```

# 解决方法2 - 使用 bind
 * **执行 opkg update 更新软件仓库列表**
 * **执行 opkg install bind-server 安装递归解析服务器**
 * **执行 /etc/init.d/named enable 将 bind 设置为自动启动**
 * **参照解决方法1禁用 dnsmasq 的 DNS 解析功能, 设置推送 DNS 服务器并修改 gfw-vpn 配置**
 * **最后重启路由器**
 * bind 的启动顺序比较靠前，有时甚至在网络初始化之前，导致其找不到正确的网络接口，可以在 /etc/bind/named.conf 中找到
```
options {
        directory "/tmp";

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        // forwarders {
        //      0.0.0.0;
        // };
......
......
......
};
```
 加入一行 interface-interval 1; 
```
options {
        directory "/tmp";

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        // forwarders {
        //      0.0.0.0;
        // };
        interface-interval 1;
......
......
......
};
```
 另其每分钟都重新查询网络接口
 * 如果之前有安装 gfw-dns，可以通过 opkg remove gfw-dns 卸载
