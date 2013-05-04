# 介绍
之前双线VPN只是打游戏用到，最近因为写该系列，将上网也改成了双线，发现一些问题：
 * 目前我使用的 [pptp](http://pptpclient.sourceforge.net/) 性能并不好, 连接局域网内的VPN服务器，用iperf测试下载速度只有 10Mbits/s
 * 该软件同时还存在内存泄漏，不能释放收到的重复包所占内存

因此决定升级到内核pptp，解决 [DualLinePPTPVPN](DualLinePPTPVPN.md) 中提到的不支持缓存数据包问题，同时优化下性能。

# 解决方法
 * 首先为不想升级到内核pptp的同学提供3个补丁 [userland_pptp_patch](userland_pptp_patch) 如果你打算使用内核pptp可以跳过这部分
其中 [100-default_timeout.patch](userland_pptp_patch/100-default_timeout.patch) 修改了默认的超时时间和窗口大小，你可以根据具体需要再修改；[101-dup_fix.patch](userland_pptp_patch/101-dup_fix.patch)修复了内存泄漏的问题；[102-high_perf_packet_queue.patch](userland_pptp_patch/102-high_perf_packet_queue.patch)优化了缓存数据包的算法，不过比起内核pptp，速度的瓶颈不在这里
 * 其次我用的固件没有内核pptp的软件包，所以我将其backport到我目前用的固件上了 [ppp_package](ppp_package) 如果使用的是最新版Openwrt请跳过这部分
 * 最后就是最重要的内核pptp及优化了 [kernel_pptp_patch](kernel_pptp_patch)：  
[989-pptp_add_seq_window.patch](kernel_pptp_patch/989-pptp_add_seq_window.patch) 为内核pptp加入了缓存数据包的功能，默认是最多缓存0.333秒和最多2048个包，如果你修改该值，请务必保证其为2的N次方，否则在序列号由0xffffffff变为0的时候会出bug。另外判断超时没有使用timer，而是需要有数据包到达，这么做的理由是如果数据包重要的话，对方也会重新发送的，即使对方不发，链路上的定时echo请求也会触发超时判断。  
[990-arc4_add_ecd.patch](kernel_pptp_patch/990-arc4_add_ecd.patch) 和 [991-arc4_use_u32_for_ctx.patch](kernel_pptp_patch/991-arc4_use_u32_for_ctx.patch) 是backport上游的修改到Openwrt 12.09所用的内核，提升了mppe加密的性能  
[992-arc4_openssl_high_perf.patch](kernel_pptp_patch/992-arc4_openssl_high_perf.patch) 是将openssl的加密代码移植到内核，加/解密性能提升在路由器上非常明显

# 性能测试
关于pptp在路由器上的性能：
 * 不使用VPN直连局域网的测试机器,用iperf（以下测试均在路由器上使用该工具）下载速度 220Mbits/s
 * 使用用户态pptp不打任何补丁，下载速度 10Mbits/s
 * 使用内核pptp,但不打arc4相关补丁,下载速度 41Mbits/s
 * 使用内核pptp,打了990和991两个arc4的补丁,下载速度 46Mbits/s
 * 使用内核pptp,打了992的arc4的补丁,下载速度 52Mbits/s
 * 不使用VPN直连局域网的测试机器，但测试机器使用tc设置发包延迟为3ms±1ms（会造成乱序包，以下均记为3ms±1ms）下载速度 95Mbits/s
 * 3ms±1ms，使用内核pptp,但不打arc4相关补丁,下载速度 38Mbits/s

关于arc4在路由器上的性能：
 * 打了990和991两个补丁，arc4 19.4MBytes/s，对比测试了 AES 128位 8.3MBytes/s，AES 256位 6.1MBytes/s
 * 打了992补丁后，arc4 28.8MBytes/s，提升接近50%
 * 使用 openssl speed测试用户态加密结果为：arc4 34.2MBytes/s； AES 128位 8.4MBytes/s；AES 256位 6.5MBytes/s


路由器上的测试做得不多，因为一旦出问题，我就无网可上了，大部分的测试都是在虚拟机中做的，测试环境为主机E3-1230，虚拟机ubuntu 12.04分配双核及虚拟机openwrt 12.09基于官方固件编译分配单核

关于内核加密在ubuntu 12.04上的性能：  
AES-128: 233MBytes/s  
AES-256: 166MBytes/s  
未打任何补丁的arc4: 182MBytes/s  
打了990和991补丁的arc4: 449MBytes/s 提升147%  
打了992补丁的arc4: 515MBytes/s  可以看到在x86上992补丁的性能提升远没有在路由器上的提升多，其实补丁里的注释已经解释了原因

openssl的用户态加密在ubuntu 12.04上的性能：  
AES-128: 258MBytes/s  
AES-256: 189MBytes/s  
arc4: 761MBytes/s  


以下测试是虚拟机ubuntu 12.04做VPN服务器（使用用户态pptp），虚拟机openwrt 12.09做客户端，raw表示不经过VPN直连，kvpn表示使用12.09自带的内核VPN，不打任何补丁，pkvpn表示使用打过989但是没有打990-992补丁的内核pptp，以下速度均为在openwrt上测试的下载速度（tcp窗口大小256k）

|                  |   raw   |  kvpn    |  pkvpn  |
| -----------------|:-------:| --------:|--------:|
| 无延迟           | 4.5Gb/s | 320Mb/s  | 350Mb/s |
| 3ms 延迟         | 300Mb/s | 250Mb/s  | 260Mb/s |
| 3ms±1ms 延迟    | 200Mb/s |  3Mb/s   | 210Mb/s |
| 200ms 延迟       |  3Mb/s  |  3Mb/s   | 4.5Mb/s |
| 200ms±50ms 延迟 |  3Mb/s  | 0.2Mb/s  | 3.5Mb/s |
|              将tcp窗口大小改为8M后              |
| 200ms 延迟       | 12Mb/s  | 12Mb/s   | 16Mb/s  |
| 200ms±50ms 延迟 |  9Mb/s  |  2Mb/s   |  9Mb/s  |

可见在有乱序包的情况下，打了补丁的内核vpn速度提升了1-2个数量级。
