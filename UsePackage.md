# 使用软件包
在OpenWrt编译环境下的 feeds.conf 中加入一行

    src-git gfw https://github.com/hackgfw/openwrt-gfw-packages.git
    
然后执行

    ./scripts/feeds update -a
    ./scripts/feeds install -a
    
就可以在 make menuconfig 里的 Network 类别下找到 gfw-*

# gfw-dns
该包对应 [AntiDNSPoisoning](AntiDNSPoisoning.md) 提到的内容  
可以手动修改 /etc/config/gfw-dns 添加/删除域名  

# gfw-vpn
该包对应 [PPTPVPN](PPTPVPN.md) 提到的内容  
需要手动修改 /etc/config/network 或通过web界面创建VPN链接并将其加入到防火墙的wan区域，然后修改 /etc/config/gfw-vpn 中的 interface 选项为刚添加的VPN链接的接口名称  
可以根据需要添加/修改/删除 /etc/config/gfw-vpn 中的rule，符合rule的数据包会走VPN，目前只支持tcp和udp协议  
/etc/config/gfw-vpn.whitezone 对应于文章中提到的 cn.zone  
/etc/config/gfw-vpn.whiteip 对应于文章中提到的 ip.whitelist  



# gfw-dualpptp
该包对应 [DualLinePPTPVPN](DualLinePPTPVPN.md) 提到的内容  
需要手动创建主/辅VPN链接并将主VPN连接加入防火墙wan区域，然后修改 /etc/config/gfw-dualpptp 中的主/辅VPN接口名称。可以使用gfw-vpn中的VPN链接作为主VPN，这样只需要创建辅VPN即可。  
如果使用openwrt 12.09或更新版，还需要根据 [OptimizePPTPVPN](OptimizePPTPVPN.md) 给内核打补丁  
最后依然需要根据文章提示，修改VPN服务器的设置  
