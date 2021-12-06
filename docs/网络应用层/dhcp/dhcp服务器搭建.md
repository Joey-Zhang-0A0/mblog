>本文介绍在Ubuntu主机上搭建dhcp服务器

| 版本号 | 作者                         | 修改时间   | 修改说明 |
| ------ | :--------------------------- | ---------- | -------- |
| v1.0   | Joey zhang 2298667492@qq.com | 2021-12-06 | 初始版本 |





## DHCP服务器搭建

### 安装

```shell
sudo apt install isc-dhcp-server
```

### 配置

#### 网络接口配置

```shell
sudo vim /etc/default/isc-dhcp-server


# Defaults for isc-dhcp-server initscript
# sourced by /etc/init.d/isc-dhcp-server       # dhcp服务启动脚本
# installed at /etc/default/isc-dhcp-server by the maintainer scripts  # 服务启动参数配置脚本

#
# This is a POSIX shell fragment
#

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPD_CONF=/etc/dhcp/dhcpd.conf  # 服务配置文件脚本路径

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPD_PID=/var/run/dhcpd.pid	# 记录dhcp服务pid文件路径

# Additional options to start dhcpd with.
#	Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""						# 启动参数选项

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#	Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACES="enp4s0"
```

配置文件中的 INTERFACES="enp4s0"， enp4s0为实际网络接口



#### 服务参数配置

```shell
sudo vim /etc/dhcp/dhcpd.conf
```

```shell
# A slightly different configuration for an internal subnet.
subnet 192.168.122.0 netmask 255.255.255.0 {
  range 192.168.122.2 192.168.122.30;
  option domain-name-servers ns1.internal.example.org;
  option domain-name "internal.example.org";
  option subnet-mask 255.255.255.0;
  option routers 192.168.122.89;
  option broadcast-address 192.168.122.31;
  default-lease-time 600;
  max-lease-time 7200;
}
```



#### 自定义option

有时需要自定义一些option来满足客户端请求。

下面定义了两个自定义的option代码分别为132和133，且都是字符类型。

```shell
option vlan_id code 132 = string;
option vlan_pri code 133 = string;

subnet 192.168.122.0 netmask 255.255.255.0 {
  range 192.168.122.127 192.168.122.128;
  option domain-name-servers ns1.internal.example.org;
  option domain-name "internal.example.org";
  option subnet-mask 255.255.255.0;
  option routers 192.168.122.89;
  option broadcast-address 192.168.122.31;
  option vlan_id "1";
  option vlan_pri "3";
  default-lease-time 600;
  max-lease-time 7200;
}
```



### 服务开启

#### 普通服务开启

开启服务

```shell
sudo service isc-dhcp-server restart
```

确认开启

```shell
ps -axu | grep dhcp

dhcpd -user dhcpd -group dhcpd -f -4 -pf /run/dhcp-server/dhcpd.pid -cf /etc/dhcp/dhcpd.conf enp4s0
```

关闭服务

```shelll
sudo service isc-dhcp-server stop
```



#### dhcp vlan服务测试

1、主机上先开启vlan节点，设置ip和出去的实际网络接口相同

```shell
sudo vconfig add enp4s0 99
sudo ifconfig enp4s0.99 192.168.122.89
```

   

2、修改服务器配置

- 指定dhcp启动到固定的网络接口, vlan网段下的enp4s0.99的服务不会开启。注释掉 /etc/default/isc-dhcp-server 中的INTERFACES这一行。这一步是关键，否则服务器只会开启在enp4s0上

```shell
#INTERFACES="enp4s0"
INTERFACES="enp4s0.99"
```

   - 发送包含vlan id和vlan priority的自定义option给客户端， /etc/dhcp/dhcpd.conf 修改其中配置，在原有的配置上加上如下配置。

```shell
option vlan_id code 132 = string; # option
option vlan_pri code 133 = string; # option

subnet 192.168.122.0 netmask 255.255.255.0 {
option vlan_id "99";
option vlan_pri "3";
}
```

   3、启动服务。

```shell
sudo service isc-dhcp-server restart
```



### DHCP6

- 创建 **/etc/dhcp/dhcpd6.conf** 作为配置文件：

```shell
# sudo vim /etc/dhcp/dhcpd6.conf
default-lease-time 600;
max-lease-time 7200;
log-facility local7;
subnet6 2001:db8:1:1::/64 {
        # Range for clients
        range6 2001:db8:1:1::129 2001:db8:1:1::254;

        # Range for clients requesting a temporary address
        range6 2001:db8:1:1::/64 temporary;

        # Additional options
        option dhcp6.name-servers fec0:0:0:1::1;
        option dhcp6.domain-search "domain.example";

        # Prefix range for delegation to sub-routers
        prefix6 2001:db8:1:100:: 2001:db8:1:f00::/56;

        # Example for a fixed host address
        host specialclient {
                host-identifier option dhcp6.client-id 00:01:00:01:4a:1f:ba:e3:60:b9:1f:01:23:45;
                fixed-address6 2001:db8:1:1::127;
        }
}
```

- 根据提示创建如下文件

```shell
sudo touch /var/lib/dhcp/dhcpd6.leases
```

- 配置网卡静态ipv6地址，需要和上面配置文件中的subnet6相同子网。

```shell
sudo ifconfig enx000ec6a1f188 inet6 add 2001:db8:1:1::1/64
```

- 启动到指定网卡上

```shell
sudo dhcpd -6 -cf /etc/dhcp/dhcpd6.conf enx000ec6a1f188
```

- 如果设备无法ping通，则设备上需要设置如下路由。（isc ipv6 dhcp serve好像不会下发网关）

```shell
# 设备上添加路由
iproute add 2001:db8:1:1::/64 dev eth0

# 设备上添加ipv6默认网关
route -A inet6 add ::/0 gw 2001:db8:1:1::1
```

  查看ipv6网关路由

```shell
route -A inet6
```

  

## 注意

在结束调试后记得将相关服务全部关闭，避免影响到原本的网络环境。

```shell
ps -aux | grep dhcp
sudo kill <pid>
```





## 错误调试

### 日志查询

```shell
vim /var/log/syslog
```



## 资料

感谢以下链接对本文章的帮助！

[ipv4 dhcp server搭建](https://linzyjx.com/archives/20.html)

[ipv6 dhcp server搭建](https://blog.csdn.net/rainforest_c/article/details/71172738)
