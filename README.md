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

1. NAT模式：让虚拟系统借助NAT(网络地址转换)功能，通过宿本机器所在的网络来访问公网。（10.0.xx.xx保留地址）
2. 桥接：在这种模式下，虚拟机就像是局域网中的一台独立的本机，它可以访问网内任何一台机器。
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

sniffer可以理解为抓包软件，也可以说是一种基于被动监听原理的网络分析方式，借助这种技术，我们可以监视网络的状态、数据流动的情况以及网络上传输的信息。Wireshark就具备sniffer的功能，而项目中为了更好地理解sniffer的工作原理，用到了Tim Carstens编写的sniffex程序，这个程序借助libpcap来实现sniffer的功能。

### 拆解sniffex程序

由于sniffex是借助pcap库实现的，所以我们可以看看程序中用到了哪些函数：

- ** char \*pcap_lookupdev(char \*errbuf); **

> 这个函数用于查找网络设备，返回设备列表中非环回接口(lo)的第一个设备。跑sniffex的时候可以从命令行传参，指定嗅探某个设备上的包，如果没有传参就调用这个函数找到默认的设备(比如linux上一般是eth0)。此外，如果没有可用的设备(返回NULL)就会把错误信息写到errbuf中，可以打印出来查看。


- ** int pcap_lookupnet(const char \*device, bpf_u_int32 \*netp, bpf_u_int32 \*maskp, char \*errbuf); **

> 这个函数用于获取指定设备的Ipv4网络号(IP地址)和子网掩码。失败会返回-1，并且把错误信息写入effbuf。


- ** pcap_t \*pcap_open_live(const char \*device, int snaplen, int promisc, int to_ms, char \*errbuf); **

> 这个函数用于打开指定设备，创建监听会话以进行抓包。特别地，在Linux下如果把设备名设为“any”或者“NULL”，sniffex会对所有接口都进行抓包。


- ** int pcap_datalink(pcap_t \*p); **

> 这个函数用于返回前面创建的监听会话的链路层头部类型，比方说我们希望抓取Ethernet接口的包，那么链路层头部类型就应该是“DLT_EN10MB”。


- ** int pcap_compile(pcap_t \*p, struct bpf_program \*fp, const char *str, int optimize, bpf_u_int32 netmask); **

> 这个函数用于编译一个过滤表达式。如果我们希望抓取特定类型的包或者某一个端口上的包而非所有包，就需要对数据包进行过滤。按官网的说法这个函数把过滤表达式(字符串格式，即第三个参数)转换为滤波器（pcap库中定义好的结构体，即第二个参数）。然后就可以这个滤波器进行过滤了。

> 表达式用法详见：[pcap filter](http://www.tcpdump.org/manpages/pcap-filter.7.html)



- ** int pcap_setfilter(pcap_t \*p, struct bpf_program \*fp); **

> 这个函数用于给监听会话设置过滤器，也即前面compile得到的结构体。


- ** int pcap_loop(pcap_t \*p, int cnt, pcap_handler callback, u_char \*user); **

> 这个函数提供持续监听的功能，会一直抓包，直到抓完cnt个包时结束监听，在sniffex中默认设置的是抓10个包然后结束。callback是个回调函数，每抓到一个包就会交给这个函数处理，我们可以在里面自己编写要对抓到的包进行什么操作，sniffex里面是把抓到的包的协议、从哪里发、要发去哪解析出来，如果包里有payload，就把payload也解析出来。


- ** void pcap_freecode(struct bpf_program \*); **

> 这个函数用于释放滤波器，监听结束后，如果不再需要使用滤波器，我们就可以用这个函数把分配给滤波器的内存释放掉。


- ** void pcap_close(pcap_t \*p); **

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

### 运行sniffex程序

运行sniffex程序除了基本的编写Makefile编译之外，有两点需要特别注意。一是需要**使用root权限**运行，二是需要**开启网卡的混杂模式**。不妨进行一下对比：

#### 关于root权限

首先看看不采用root权限运行时的报错：

![root权限1](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image7.png)

对比一下源代码不难发现实在pcap_lookupdev这个函数处出错抛出的：

![root权限2](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image8.png)

在使用sniffex的时候，我们需要获取网络设备，并且进行设置，而这些权限只有root用户有，普通用户权限不够自然就报错了。

#### 关于混杂模式

首先要清楚混杂模式和非混杂模式的区别，非混杂模式下，网卡只接收与当前本机相关的包，比方说源地址或者目的地址是当前本机，也包括广播包和多播包。 混杂模式下，网卡接收所有经过的包，即使是发往别处的包，也会进行接收，因此我们可以监听别人的数据包。

在非混杂模式下，使用sniffex嗅探到的包只有以172.16.85.118(eth0的网络号)为目的，或由它发出的包。而如果启用了混杂模式，就可以抓到由网络中别的本机发出或发往别的本机的包，比如下面这个：

![混杂模式](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image9.png)

172.16.84.44是本机的网络号，因为前面设置了桥接模式，所以此时的网络拓扑中，虚拟的的eth0和本机是网络中各自独立的两台机器，发往本机的这个包经过了eth0，虽然与eth0无关，但由于eth0启用了混杂模式，所有经过的数据包都会被截获，所以还是能够抓下来。

### 过滤表达式的使用

#### 抓取两个特定本机之间的ICMP包

![filter](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image10.png)

![filter](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image11.png)

首先限定包类型为icmp，然后限定只接收指定两台本机之间的包，这里设置为本机和eth0之间，当我从本机 ping eth0，或者从eth0 ping 本机时就会把包抓下来了，而ping其他网络的icmp包则会被过滤掉。

#### 抓取目的端口号为10-100的TCP包

![filter](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image12.png)

![filter](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image13.png)

首先使用tcp限定包的协议类型，然后用dst关键字指定后面的端口号是目的端口号，portrange可以用来限定一个端口的范围。这里抓到的包是本机发给阿里云的一个包，可以用[www.db-ip.com](www.db-ip.com)查询网络号归属。 80端口是为HTTP(超文本传输协议)开放的,是我们上网时用得最多的一个端口，一般我们浏览网页都是用的都是80端口。

### 嗅探密码

这是一个真正的借助嗅探器盗取密码的应用，黑客通过sniffex程序来嗅探网络中的数据包，盗取别的用户在使用Windows自带的远程登录(Telnet)程序时输入的用户名和密码。当然这里的黑客是我们自己，而受攻击的是我们自己的本机。

首先需要在本机启用Telnet，Win10默认是禁用的。

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image14.png)

启用Telnet之后，我们就可以在命令行直接使用telnet命令呼出Telnet客户端，然后借助Telnet登录远程本机了。

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image15.png)

在Telnet客户端界面使用命令“o 本机IP 端口号”可以登录远程本机，如果不指定端口号，则默认是端口23。其中o表示open，Telnet更详细的用法不妨看看[这篇博客](http://blog.chinaunix.net/uid-26167002-id-3054040.html)。

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image16.png)

连接成功会进行远程本机的登陆界面，这里我连的就是实验用的虚拟机SEEDUbuntu，用户名seed，密码dees，输入后我们就可以控制这台远程本机，调用它的一切软、硬件资源了，退出时键入exit命令回车即可。

我们常常在黑客电影中听到的“肉鸡”其实就是黑客入侵了别人的本机，利用这种方法，从一个“肉鸡”再登录到另一个“肉鸡”，这样黑客在入侵过程中就不会暴露自己的IP地址，从而实现隐身的功能。

回到正题，这里我们主要是希望借助sniffex嗅探到用户登录远程本机的用户名和密码，如果能嗅探到，那么我们就能入侵那台远程本机了。简单修改一下过滤表达式，嗅探telnet服务使用的端口23：

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image17.png)

由于使用telnet登录远程本机时传输的包比较多，还包括一些应用数据之类的，所以这个实验要把抓包的数量调大一些，这里我直接设置为999，想停止时直接Ctrl+C就可以了。

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image18.png)

比较有意思的是，发包的时候，密码和用户名都是明文逐字母发出的，两个字母之间会发一个无payload的tcp包，并且密码中的每个字母会重复发两次。

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image19.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image20.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image21.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image22.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image23.png)

以上合起来倒着看就是，密码：dees

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image24.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image25.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image26.png)

![telnet](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image27.png)

以上合起来倒着看就是用户名，seed

因为telnet采用明文传输数据，所以任何人都可以借助嗅探器来嗅探到别人的密码，非常不安全。因此，要使用远程登录功能时，最好还是使用基于ssh这样带加密功能协议的程序。

## 包的仿冒

采用raw socket变成来实现，代码都由[pdbuchan](http://www.pdbuchan.com/rawsock/rawsock.html)网站提供，只需要修改一下源IP地址和目标IP地址然后编译运行(root权限下)就可以了。这里主要还是解析一下代码的逻辑：

1. 填充IP header
2. 填充上层协议header
3. 利用填充好的header构建伪造的IP包
4. 创建一个套接字
5. 把套接字绑定到指定的网卡
6. 发送伪造的IP包

先看看IP包的头部：

![IP header](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image29.png)

这里我们使用Ipv4，所以Version是4，IHL头部长度是20，Type of Service不用管，设为0，总长度是40（IP头部和TCP头部各20，不携带其他数据）；因为我们只发一个包，不分段，所以Identification不需要用到了，DF是否分段设为否（0），MF是否还有下一段设为否（0），偏移也为0；TTL设为最大跳数255；假设我们要仿冒一个TCP包，那么协议这里就设置为IPPROTO_TCP；源IP地址和目的IP地址根据自己需要来设定，这里我设置为虚拟机是源，本机是目的地，最后计算头部校验和。

再看看TCP包的头部：

![TCP header](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image30.png)

源端口目标端口根据自己的需要来设置，这里设置为60端口和80端口；序列号和确认号都设为0，即这是三次握手中第一个发的包。头部长度20，六个标志位中只有SYN置为1，因为这是一个用来建立连接的包，Window size设为65535，最后同样计算一下头部校验和。

![raw socket](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image31.png)

两个header填充好之后，就可以放入包里面了，特别要注意的是我们往常使用套接字时一般都是基于传输层TCP或者UDP来用的，但使用原始套接字，我们可以设置更底层的信息，比如这里我们就对网络层的IP header进行了设置，数据链路层的信息（以太网帧的header）则交给内核来实现（若有需要我们也可自己填写）。

因为没有需要发送数据，所以包里面只有两个header，payload是空的。接下来创建套接字并且绑定到接口就可以发包了，注意指明IP header由我们自己提供，否则默认是由内核实现的。

![raw socket](https://raw.githubusercontent.com/familyld/Packet-Sniffing-and-Spoofing-with-pcap/master/graph/image32.png)
