# docker_bridge_network_setup_ipv6
docker容器使用缺省bridge模式获得公网IPv6地址并配置路由的方法

这段时间需要使用容器去部署一些应用，而且这些应用需要使用公网的ipv6地址对外发布，
于是去研究了一下docker官方的文档及网上一些大神的教程，发现都是问题多多，
按照那些说明文档及教程基本是不能工作或工作不稳定的，这个也从一个侧面反映出docker根本没有多少心思放在完善ipv6支持上

下面是我实际操作的流程及提醒特别注意的地方，流程假设你已经从服务供应商获得ipv6公网地址段::/64，
部署容器的宿主机对外网卡为ens33，宿主机系统为ubuntu 18

首先查看宿主机网卡ipv6地址段
#ip -f inet6 addr show ens33

选取的是公网IP，也就是后面scope global指示的，下面的ipv6子网为2a01:53c0:ff0e:2e::/64
inet6 2a01:53c0:ff0e:2e::b139/128 scope global dynamic noprefixroute
inet6 2a01:53c0:ff0e:2e:20c:29ff:feff:1453/64 scope global dynamic mngtmpaddr noprefixroute


以下daemon.json配置文件使得容器实例获得外访ipv6的能力，这时只要是使用系统缺省bridge网络模式的容器实例，
都会被随机分配到2a01:53c0:ff0e:2e:2::/80网段的一个ipv6地址，但要外部ipv6主动发起入站访问就比较麻烦，
原因是使用缺省bridge网络模式时docker run命令的--ip6参数不起作用，所以每次重启后都不能确定容器实例的ipv6地址，
必须查看，对于想把ipv6用于固定解析域名就比较棘手了，这种情况的解决方法可以是把用到bridge模式的容器个数，
即从::2开始都写一次ip -6 neigh add proxy <ipv6_addr> dev ens33开机命令，
然后在需要解析域名的每个容器里面写更新域名解析商如cloudflare的脚本，然后容器内设置脚本定时(非@reboot)执行，
啰嗦了一些题外话，言归正传


编辑或追加配置文件内容，至少要定义一个ipv6子网段，而且这个子网段必须在::/64内，否则报错
#nano /etc/docker/daemon.json
写入下面内容

{
"ipv6": true,
"fixed-cidr-v6": "2a01:53c0:ff0e:2e:2::/80"
}

保存退出并重启容器服务
#systemctl restart docker

至此容器实例使用系统缺省bridge(即NAT)模式的，已经可以获得公网ipv6地址，
但需要下面的设置才能使容器与宿主机或外部ipv6网络互访

这些配置写在/etc/sysctl.conf文件里面，关键是net.ipv6.conf.all.proxy_ndp=1的“all“千万不要按网上很多教程所说的，
写成网卡接口名字(如ens33)，会导致时通时不通
#echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
#echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
#echo "net.ipv6.conf.all.proxy_ndp=1" >> /etc/sysctl.conf


使配置生效
#sysctl -p

然后创建容器实例，之后docker attach到容器里面查看ipv6地址，假设是2a01:53c0:ff0e:2e:2::2，然后退出，
在宿主机命令行打入下面命令，增加ipv6邻居代理，这时外界才能真正与容器进行ipv6互通

#ip -6 neigh add proxy 2a01:53c0:ff0e:2e:2::2 dev ens33

但麻烦的是每次重启宿主机或容器服务，或几个容器实例一起重启，这个容器实例的ipv6地址都会变，那每次都要进去查看当时的ipv6地址,
然后再修改上面命令的ipv6地址，再打这个命令，那基本是灾难

那有没有方法指定容器实例的ipv6地址呢？有的，就是配置自定义的bridge模式网络，下面步骤创建自定义的bridge模式网络，
主要目的是使用自定义bridge模式网络后，可以在docker run命令使用--ip6参数指定容器ipv6地址，
当然下面的ipv6子网号及掩码可以修改，只要与主网段不冲突即可，ipv6的网关参数可以不写，系统会自动指定::1，ipv4网关参数同样
#docker network create -d bridge --ipv6 --subnet=2a01:53c0:ff0e:2e:1::/80 --subnet=172.28.0.0/16 --gateway=172.28.0.1 ip6bridge

然后使用自定义的bridge的网络名称ip6bridge创建指定ipv6地址的容器，这时的--ip6参数才能起作用，
因为使用系统缺省bridge网络是不能指定--ip6参数的，即使指定了也不生效，也不报错
#docker run -dit --restart=always --network=ip6bridge --ip6=2a01:53c0:ff0e:2e:1::2 --name=u18ip6bridge2 -v /data:/data ubuntu:bionic-20210827 /bin/bash -c "/etc/init.d/cron start;/etc/init.d/run;/bin/bash"

#docker run -dit --restart=always --network=ip6bridge --ip6=2a01:53c0:ff0e:2e:1::3 --name=u18ip6bridge3 -v /data:/data ubuntu:bionic-20210827 /bin/bash -c "/etc/init.d/cron start;/etc/init.d/run;/bin/bash"


宿主机增加ipv6邻居代理
#ip -6 neigh add proxy 2a01:53c0:ff0e:2e:1::2 dev ens33
#ip -6 neigh add proxy 2a01:53c0:ff0e:2e:1::3 dev ens33

这两条命令要写在宿主机的定时任务@reboot或启动脚本，“2a01:53c0:ff0e:2e:1::2”为容器内部的ipv6地址，
缺点是每创建一个容器都需要在宿主机添加ip -6 neigh add proxy <ipv6_addr> dev ens33命令，对于自动化构建较为麻烦


至此我们已经创建出指定公网ipv6地址的容器实例，并且互联网上面所有ipv6机器都可以直接访问这些具有公网ipv6的容器服务

