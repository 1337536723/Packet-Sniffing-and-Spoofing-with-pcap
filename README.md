# 使用pcap进行包嗅探并仿冒包

这个项目算是网络安全中比较基础的一个入门项目。因为包在传输过程中需要经过若干路由器，所以我们可以监听所有经过的数据包，从而嗅探出包里面的信息。而借助raw socket编程，我们可以仿冒包（伪造IP地址、端口等信息），从而进行欺骗。

## 配置

### SEEDUbuntu镜像使用方法

1. 使用VMware
选择"打开现有虚拟机"，然后选中解压后文件夹中的SEEDUbuntu12.04.vmx文件即可，这是虚拟机配置文件。

2. 使用Virtual Box
可以按照教程操作：http://www.cis.syr.edu/~wedu/seed/Documentation/VirtualBox/UseVirtualBox.pdf

3. 用户名和密码
    - User ID: root, Password: seedubuntu.
    - User ID: seed, Password: dees
    - 由于Ubuntu是不能直接用root用户登录的，所以都是用seed这个普通用户登录。注意要使用到root权限时，还是输入seed用户的密码。

初次打开虚拟机会比较慢，虚拟机中已经安装好所有项目中需要用到的软件和环境，只要简单安装一下和本机互传文件的插件就可以用了。

### 虚拟机网络配置

首先配置好环境：

软件：Vmware Workstation 12 Player

系统：SEEDUbuntu12.04(使用方式见附录)

网络：桥接模式

虚拟机中有三种网络连接方式：

1. NAT模式：让虚拟系统借助NAT(网络地址转换)功能，通过宿主机器所在的网络来访问公网。（10.0.xx.xx保留地址）
2. 桥接：在这种模式下，虚拟机就像是局域网中的一台独立的主机，它可以访问网内任何一台机器。
3. Host-only：在某些特殊的网络调试环境中，要求将真实环境和虚拟环境隔离开，这时你就可采用host-only模式。在host-only模式中，所有的虚拟系统是可以相互通信的，但虚拟系统和真实的网络是被隔离开的。

选用桥接模式来做这个项目，该模式下虚拟机和本机的地位是同等的：

![桥接模式](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image2.jpeg)

Vmware提供非常方便的功能，我们甚至不用主动设置桥接网络(ip要和本机一个网段，并且掩码等设置也要一样)，只需要修改一下虚拟机设置中的网络适配器，其他的都可以自动完成：

![VMware桥接模式](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image3.png)

设置完成后，打开SEEDUbuntu12.04，输入ifconfig查看网络信息，可以看到此时ip地址是172.16.85.118，这是自动分配的和本机在同一网段的一个可用ip，我本机的ip地址是172.16.84.44，子网掩码都是255.255.254.0。

![IP信息](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image4.png)

除了配置桥接网络之外，因为实验中我们希望能使用混杂模式从而能抓取到经过网卡的所有数据包，这一点不仅仅要从软件上设置，硬件上也要把对应的网卡设置为混杂模式，实验中用到的网卡是eth0，使用下面语句可以把该网卡设置为混杂模式：

![开启混杂模式](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image5.png)

![开启混杂模式](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image6.png)

设置完成后，可以看到此时eth0的网络状况中多出了PROMISC，这样网卡的混杂模式就启用了。
