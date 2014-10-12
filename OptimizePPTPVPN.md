# 介绍
Openwrt 12.09 和 14.07 所用的内核pptp不支持缓存数据包，而这个功能对于双线VPN至关重要，因此该文将解决这个问题，同时优化下性能。  
补丁放在了 [kernel_patch](kernel_patch)：  
**pptp_accept_seq_window.patch** 为内核pptp加入了缓存数据包的功能，默认是最多缓存0.333秒和最多2048个包，如果你修改该值，请务必保证其为2的N次方，否则在序列号由0xffffffff变为0的时候会出bug。另外判断超时没有使用timer，而是需要有数据包到达，这么做的理由是如果数据包重要的话，对方也会重新发送的，即使对方不发，链路上的定时echo请求也会触发超时判断。  
**arc4_add_ecd.patch** 和 **arc4_use_u32_for_ctx.patch** 是backport上游的修改到Openwrt 12.09所用的内核，提升了mppe加密的性能，Openwrt 14.07不需要打这两个补丁  
**arc4_openssl_high_perf.patch** 是将openssl的加密代码移植到内核，加/解密性能提升在路由器上非常明显

# 解决方法
 * 如果使用的是Openwrt 12.09，则将 [12.09](kernel_patch/12.09) 下的4个文件复制到编译环境的 target/linux/generic/patches-3.3 目录，重新编译即可
 * 如果使用的是Openwrt 14.07，则将 [14.07](kernel_patch/14.07) 下的2个文件复制到编译环境的 target/linux/generic/patches-3.10 目录，重新编译即可
 * 打上补丁后VPN链接的统计信息会放到 /proc/pptp/ 下，例如执行 cat /proc/pptp/pptp-wall 会显示

```
Accepted: 8748724
Under Window: 8654920
Buffered: 238735
Duplicated: 3944
Lost: 1
```

其中 **Accepted** 是收到实际有效的数据包数量（既交给上层应用的数据包数量）  
**Under Window** 有两种情况触发，一种是从较快的VPN链路上已经收到了所有包，之后从较慢的VPN链路上收到的重复包会计入该值。第二种情况是某个包在超时之后才收到，虽然对于VPN链路来说不算丢包，但对于上层应用在之前触发超时的时候就已经发生了丢包  
**Buffered** 是收到不连续包的数量，通常是由于其中一条或多条VPN链路不稳定导致，Buffered/Accepted的比值越大说明越不稳定  
**Duplicated** 是收到的确定重复的数据包数量。和 Under Window 的区别在于 Under Window 不能确定就是重复包导致的，而 Duplicated 可以保证是重复包造成的。如果没有发生某个包在超时之后才收到，那么 Under Window + Duplicated = 收到的总重复包数量  
**Lost** 是最终的丢包数量（既上层应用感知到的丢包数量），这个值通常是从1开始的。如果你发现这个值为0，则说明你使用的VPN服务器有bug  
  
对于双线VPN的主链接，Accepted + (Lost - 1) * 2 - Under Window - Duplicated = 主VPN链路的丢包数量 + 辅VPN链路的丢包数量  
对于单线VPN或双线VPN的辅链接，Duplicated应为0，(Lost - 1) - Under Window = VPN链路的丢包数量


# 性能测试
关于pptp在Openwrt 12.09路由器上的性能：
 * 不使用VPN直连局域网的测试机器,用iperf（以下测试均在路由器上使用该工具）下载速度 220Mbits/s
 * 使用用户态pptp不打任何补丁，下载速度 10Mbits/s
 * 使用内核pptp,但不打arc4相关补丁,下载速度 41Mbits/s
 * 使用内核pptp,打了arc4_add_ecd和arc4_use_u32_for_ctx两个arc4的补丁,下载速度 46Mbits/s
 * 使用内核pptp,打了arc4_openssl_high_perf的arc4的补丁,下载速度 52Mbits/s
 * 不使用VPN直连局域网的测试机器，但测试机器使用tc设置发包延迟为3ms±1ms（会造成乱序包，以下均记为3ms±1ms）下载速度 95Mbits/s
 * 3ms±1ms，使用内核pptp,但不打arc4相关补丁,下载速度 38Mbits/s

关于arc4在Openwrt 12.09路由器上的性能：
 * 打了arc4_add_ecd和arc4_use_u32_for_ctx两个补丁，arc4 19.4MBytes/s，对比测试了 AES 128位 8.3MBytes/s，AES 256位 6.1MBytes/s
 * 打了arc4_openssl_high_perf补丁后，arc4 28.8MBytes/s，提升接近50%
 * 使用 openssl speed测试用户态加密结果为：arc4 34.2MBytes/s； AES 128位 8.4MBytes/s；AES 256位 6.5MBytes/s


路由器上的测试做得不多，因为一旦出问题，我就无网可上了，大部分的测试都是在虚拟机中做的，测试环境为主机E3-1230，虚拟机ubuntu 12.04分配双核及虚拟机openwrt 12.09基于官方固件编译分配单核

关于内核加密在ubuntu 12.04上的性能：  
AES-128: 233MBytes/s  
AES-256: 166MBytes/s  
未打任何补丁的arc4: 182MBytes/s  
打了arc4_add_ecd和arc4_use_u32_for_ctx补丁的arc4: 449MBytes/s 提升147%  
打了arc4_openssl_high_perf补丁的arc4: 515MBytes/s  可以看到在x86上arc4_openssl_high_perf补丁的性能提升远没有在路由器上的提升多，其实补丁里的注释已经解释了原因

openssl的用户态加密在ubuntu 12.04上的性能：  
AES-128: 258MBytes/s  
AES-256: 189MBytes/s  
arc4: 761MBytes/s  


以下测试是虚拟机ubuntu 12.04做VPN服务器（使用用户态pptp），虚拟机openwrt 12.09做客户端，raw表示不经过VPN直连，kvpn表示使用12.09自带的内核VPN，不打任何补丁，pkvpn表示使用打过pptp_accept_seq_window但是没有打arc4相关补丁的内核pptp，以下速度均为在openwrt上测试的下载速度（tcp窗口大小256k）

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
