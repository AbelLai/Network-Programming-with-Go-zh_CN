套接字级编程
==============

    本章将着眼于网络编程的基础方法，将涉及到主机和服务寻址，也会考虑到TCP和UDP。同时也将展示如何使用GO的TCP和UDP相关的API来构建服务器和客户端。最后介绍了原生套接字，如果你需要基于IP协议实现你自己的协议的话。
    

介绍
--------------

世上存在很多种网络。它们涵盖了从古老如串行链路，到为计算机或手机这样的通讯设备所搭建的铜缆和光纤的广域网，或各种各样的无线网络。在物理链路层它们区别明显，但很多时候，在更高层次的OSI模型它们也存在差异。

多年的发展，使得IP和TCP/UDP协议基本上就等价于网络协议栈。例如, 蓝牙定义了物理层和协议层，但最重要的是IP协议栈，可以在许多蓝牙设备使相同的互联网编程技术。同样, 开发4G无线手机技术，如LTE（Long Term Evolution）也将使用IP协议栈。

IP提供了第3层的OSI网络协议栈，TCP和UDP则提供了第4层。即使在因特网世界，这些都不是固定不变的：TCP和UDP将面临来自SCTP（STREAM CONTROL TRANSMISSION PROTOCOL 流控制传输协议）的挑战，同时在星际空间中提供互联网服务需要新的像正在开发的DTN协议。不过，IP, TCP和UDP至少在当前甚至未来相当长的时间内是主要的网络技术。Go语言提供了对这种编程的全面支持。

本章介绍如何使用GO编写TCP和UDP程序，以及如何使用其他协议的原始套接字。

TCP/IP协议栈
----------------

OSI模型标准的建立和实施是一个委员会（国际标准化组织ISO--译者注）设计的。OSI标准中的一些部分是模糊的，有些部件不能很容易地实现，一些地方还没有得到落实。

TCP/IP协议由长期运行的一个DARPA（美国国防先进研究项目局）项目设计。该工作其次由RFC (Request For Comment)实施。TCP/IP是Unix的首要网络协议。TCP/IP等于传输控制协议/互联网协议。

TCP/IP协议栈是OSI模型的一部分：

.. image:: _static/img/tcp_stack.gif

TCP是一个面向连接的协议，UDP（User Datagram Protocol，用户数据报协议）是一种无连接的协议。

IP数据包
~~~~~~~~~~

IP层提供了无连接的不可靠的传输系统，任何数据包之间的关联必须依赖更高的层来提供。

IP层包头支持数据校验，在包头包括源地址和目的地址。

IP层通过路由连接到因特网，还负责将大数据包分解为更小的包，并传输到另一端后进行重组。

UDP
~~~~~

UDP是无连接的，不可靠的。它包括IP数据报的内容和端口号的校验。在后面，我们会用它来构建一些客户端/服务器例子。

TCP
~~~~

UDP是无连接的，不可靠的。它包括IP数据报的内容和端口号的校验。在后面，我们会用它来构建一些客户端/服务器例子。

互联网地址
-------------

要想使用一项服务，你必须先能找到它。互联网使用地址定位例如计算机的设备。这种寻址方案最初被设计出来只允许极少数的计算机连接上，使用32位无符号整形，拥有高达2^32个地址。这就是所谓的IPv4地址。近年来，连接（至少可以直接寻址）的设备的数量可能超过这个数字，所以在不久的某一天我们将切换到利用128位无符号整数，拥有高2^128个地址的IPv6寻址。这种转换最有可能被已经耗尽了所有的IPv4地址的新兴国家发达地区。

IPv4地址
~~~~~~~~~

IP地址是一个32位整数构成。每个设备的网络接口都有一个地址。该地址通常使用'.'符号分割的4字节的十进制数，例如："127.0.0.1" 或 "66.102.11.104"。

所有设备的IP地址，通常是由两部分组成：网段地址和网内地址。从前，网络地址和网内地址的分辨很简单，使用字节构建IP地址。

- 一个A类网络，前1个字节标识为网络地址，同时后3字节标识为主机地址。A类网络只有128个, 被很早的互联网成员例如IBM，通用电气公司(the General Electric Company)和MIT所拥有。(http://www.iana.org/assignments/ipv4-address-space/ipv4-address-space.xml)
- 一个B类网络使用前2个字节标识为网络地址，后2字节标识为子网的主机地址。这最多允许2^16 (65,536)个设备在同一个子网。
- 一个C类网络使用前3字节标识为网络地址，后1字节的主机地址。这最多允许2^8 (其实是254, 不是256)个设备。

但是，比如你需要400台计算机在同一个网络，该方案是不可行的。254太小，而65,536又太大。根据二进制计算，你大约需要512(2^9，译者注)。这样就可以通过使用一个23位的网络地址和9位的设备地址实现。同样，如果您需要高达1024台设备，使用一个22位网络地址和一个10位的设备地址。

知道设备的IP地址和多少字节用于网络地址，那么可以比较直接的提取出这个网络中的网络地址和设备地址。例如：“网络掩码”是一个前面N位为1，其他所有位为0的32位二进制数。例如，如果使用16位的网络地址，掩码为11111111111111110000000000000000。使用二进制有一点不方便，所以通常使用十进制字节。16位网络地址的子网掩码是255.255.0.0，而对于23位地址，这将是255.255.254.0，和22位地址，这将是255.255.252.0。

接着查找设备的网络，并将其IP地址与网络掩码进行按位与操作，而该设备在子网中的地址，可通过其IP地址同掩码与1的补码的按位与操作发现。

IPv6地址
~~~~~~~~~~

因特网的迅速发展大大超出了原来的预期。最初富余的32位地址解决方案已经接近用完。虽然有一些例如NAT地址输入这样不是很完美的解决方法，但最终我们将不得不切换到更广阔的地址空间。IPv6使用128位地址，即使表达同样的地址，字节数变得很麻烦，由':'分隔的4位16进制组成。一个典型的例子如：2002:c0e8:82e7:0:0:0:c0e8:82e7。

要记住这些地址并不容易！DNS将变得更加重要。有一些技巧用来介绍一些地址，如省略一些零和重复的数字。例如："localhost"地址是：0:0:0:0:0:0:0:1，可以缩短到::1。

IP地址类型
--------------

IP类型
~~~~~~~~~~~~

"net"包定义了许多类型, 函数，方法用于Go网络编程。IP类型被定义为一个字节数组。

.. code-block:: golang

    type IP []byte
    
有几个函数来处理一个IP类型的变量, 但是在实践中你很可能只用到其中的一些。例如, ParseIP(String)函数将获取逗号分隔的IPv4或者冒号分隔的IPv6地址, 而IP方法的字符串将返回一个字符串。请注意，您可能无法取回你期望的: 字符串 0:0:0:0:0:0:0:1是::1。

下面用一个程序来说明

.. code-block:: golang

    /* IP
     */

    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s ip-addr\n", os.Args[0])
                    os.Exit(1)
            }
            name := os.Args[1]

            addr := net.ParseIP(name)
            if addr == nil {
                    fmt.Println("Invalid address")
            } else {
                    fmt.Println("The address is ", addr.String())
            }
            os.Exit(0)
    }

如果编译它为可执行文件IP，那么它可以运行如

.. code-block:: shell
    
    IP 127.0.0.1
    
得到结果

.. code-block:: shell

    The address is 127.0.0.1

或

.. code-block:: shell

    The address is ::1
    
IP掩码
~~~~~~~

为了处理掩码操作，有下面类型：

.. code-block:: golang

    type IPMask []byte
    
下面这个函数用一个4字节的IPv4地址来创建一个掩码

.. code-block:: golang

    func IPv4Mask(a, b, c, d byte) IPMask
    
另外, 这是一个IP的方法返回默认的掩码

.. code-block:: golang

    func (ip IP) DefaultMask() IPMask
    
需要注意的是一个掩码的字符串形式是一个十六进制数，如掩码255.255.0.0为ffff0000。
    
一个掩码可以使用一个IP地址的方法，找到该IP地址的网络

.. code-block:: golang

    func (ip IP) Mask(mask IPMask) IP
    
下面的程序是一个使用了这个的例子：

.. code-block:: go

    /* Mask
     */

    package main

    import (
            "fmt"
            "net"
            "os"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s dotted-ip-addr\n", os.Args[0])
                    os.Exit(1)
            }
            dotAddr := os.Args[1]

            addr := net.ParseIP(dotAddr)
            if addr == nil {
                    fmt.Println("Invalid address")
                    os.Exit(1)
            }
            mask := addr.DefaultMask()
            network := addr.Mask(mask)
            ones, bits := mask.Size()
            fmt.Println("Address is ", addr.String(),
                    " Default mask length is ", bits,
                    "Leading ones count is ", ones,
                    "Mask is (hex) ", mask.String(),
                    " Network is ", network.String())
            os.Exit(0)
    }

    
    
    
    
    