# 使用软件包
在OpenWrt编译环境下的 feeds.conf 中加入一行

    src-git gfw https://github.com/hackgfw/openwrt-gfw-packages.git
    
然后执行

    ./scripts/feeds update -a
    ./scripts/feeds install -a
    
就可以在 make menuconfig 里的 Network 类别下找到 gfw-*

关于各包的配置可参见 [README](README.md)
