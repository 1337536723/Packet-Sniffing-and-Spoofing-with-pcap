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

## 包的嗅探

### sniffer(嗅探器)

sniffer可以理解为抓包软件，也可以说是一种基于被动监听原理的网络分析方式，借助这种技术，我们可以监视网络的状态、数据流动的情况以及网络上传输的信息。Wireshark就具备sniffer的功能，而项目中为了更好地理解sniffer的工作原理，所以用到了Tim Carstens编写的sniffex程序，这个程序借助libpcap来实现sniffer的功能。

### 拆解sniffex程序

由于sniffex是借助pcap库实现的，所以我们可以看看程序中用到了哪些函数：

- ##### char \*pcap_lookupdev(char \*errbuf);

> 这个函数用于查找网络设备，返回设备列表中非环回接口(lo)的第一个设备。跑sniffex的时候可以从命令行传参，指定嗅探某个设备上的包，如果没有传参就调用这个函数找到默认的设备(比如linux上一般是eth0)。此外，如果没有可用的设备(返回NULL)就会把错误信息写到errbuf中，可以打印出来查看。


- ##### int pcap_lookupnet(const char \*device, bpf_u_int32 \*netp, bpf_u_int32 \*maskp, char \*errbuf);

> 这个函数用于获取指定设备的Ipv4网络号(IP地址)和子网掩码。失败会返回-1，并且把错误信息写入effbuf。


- ##### pcap_t \*pcap_open_live(const char \*device, int snaplen, int promisc, int to_ms, char \*errbuf);

> 这个函数用于打开指定设备，创建监听会话以进行抓包。特别地，在Linux下如果把设备名设为“any”或者“NULL”，sniffex会对所有接口都进行抓包。


- ##### int pcap_datalink(pcap_t \*p);

> 这个函数用于返回前面创建的监听会话的链路层头部类型，比方说我们希望抓取Ethernet接口的包，那么链路层头部类型就应该是“DLT_EN10MB”。


- ##### int pcap_compile(pcap_t \*p, struct bpf_program \*fp, const char *str, int optimize, bpf_u_int32 netmask);

> 这个函数用于编译一个过滤表达式。如果我们希望抓取特定类型的包或者某一个端口上的包而非所有包，就需要对数据包进行过滤。按官网的说法这个函数把过滤表达式(字符串格式，即第三个参数)转换为滤波器（pcap库中定义好的结构体，即第二个参数）。然后就可以这个滤波器进行过滤了。

> 表达式用法详见：[pcap filter](http://www.tcpdump.org/manpages/pcap-filter.7.html)



- ##### int pcap_setfilter(pcap_t \*p, struct bpf_program \*fp);

> 这个函数用于给监听会话设置过滤器，也即前面compile得到的结构体。


- ##### int pcap_loop(pcap_t \*p, int cnt, pcap_handler callback, u_char \*user);

> 这个函数提供持续监听的功能，会一直抓包，直到抓完cnt个包时结束监听，在sniffex中默认设置的是抓10个包然后结束。callback是个回调函数，每抓到一个包就会交给这个函数处理，我们可以在里面自己编写要对抓到的包进行什么操作，sniffex里面是把抓到的包的协议、从哪里发、要发去哪解析出来，如果包里有payload，就把payload也解析出来。


- ##### void pcap_freecode(struct bpf_program \*);

> 这个函数用于释放滤波器，监听结束后，如果不再需要使用滤波器，我们就可以用这个函数把分配给滤波器的内存释放掉。


- ##### void pcap_close(pcap_t \*p);

> 这个函数用于关闭监听会话并释放相应的资源。

**p.s. 以上函数的用途和原型都可以在[tcpdump](http://www.tcpdump.org/manpages/)网页对应的文件中查找到。**

#### 简单归纳一下sniffex嗅探数据包的流程

1. 取得网络设备，若用户未指定则利用pcap_lookupdev来获取；

2. 获取设备的网络号和掩码；

3. 启动设备，创建监听会话，获取到会话的句柄(Handle）；

4. 验证设备是否以太网类型，是则继续，否则报错；

5. 编译过滤表达式

6. 把过滤表达式挂载到监听会话的句柄上，只有符合过滤规则的数据包才会被接收；

7. 开始监听，直到接收到指定数量的数据包，特别地，可以通过设置回调函数来处理接收到的数据包；

8. 监听结束，释放滤波器占用的空间

9. 关闭监听会话并释放相应资源。
