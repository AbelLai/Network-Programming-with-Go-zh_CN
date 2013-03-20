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

编译并运行Mask

.. code-block:: shell

    Mask 127.0.0.1
    
将返回

.. code-block:: shell

    Address is  127.0.0.1  Default mask length is  8  Network is  127.0.0.0
    
IPAddr 类型
~~~~~~~~~~~~~

在net包的许多函数和方法会返回一个指向IPAddr的指针。这不过只是一个包含IP类型的结构体。

.. code-block:: go

    type IPAddr {
        IP IP
    }
        
这种类型的主要用途是通过IP主机名执行DNS查找。

.. code-block:: go

    func ResolveIPAddr(net, addr string) (*IPAddr, os.Error)
    
其中net是"ip","ip4"或者"ip6"的其中一个. 下面的程序中将会展示。

.. code-block:: go

    /* ResolveIP
     */

    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s hostname\n", os.Args[0])
                    fmt.Println("Usage: ", os.Args[0], "hostname")
                    os.Exit(1)
            }
            name := os.Args[1]

            addr, err := net.ResolveIPAddr("ip", name)
            if err != nil {
                    fmt.Println("Resolution error", err.Error())
                    os.Exit(1)
            }
            fmt.Println("Resolved address is ", addr.String())
            os.Exit(0)
    }
        
运行ResolveIP www.google.com返回

.. code-block:: shell

    Resolved address is  66.102.11.104
    
主机查询
~~~~~~~~~~~~

ResolveIPAddr函数将对某个主机名执行DNS查询，并返回一个简单的IP地址。然而，通常主机如果有多个网卡，则可以有多个IP地址。它们也可能有多个主机名，作为别名。

.. code-block:: go

    func LookupHost(name string) (cname string, addrs []string, err os.Error)
    
这些地址将会被归类为“canonical”主机名。如果你想找到的规范名称，使用 func LookupCNAME(name string) (cname string, err os.Error)

下面是一个演示程序

.. code-block:: go

    /* LookupHost
     */

    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s hostname\n", os.Args[0])
                    os.Exit(1)
            }
            name := os.Args[1]

            addrs, err := net.LookupHost(name)
            if err != nil {
                    fmt.Println("Error: ", err.Error())
                    os.Exit(2)
            }

            for _, s := range addrs {
                    fmt.Println(s)
            }
            os.Exit(0)
    }

注意，这个函数返回字符串，而不是IPAddress。

服务
-----

服务运行在主机。它们通常长期存活，同时被设计成等待的请求和响应请求。有许多类型的服务，有他们能够通过各种方法向客户提供服务。互联网的世界基于TCP和UDP这两种通信方法提供许多这些服务，虽然也有其他通信协议如SCTP​​伺机取代。许多其他类型的服务，例如点对点, 远过程调用, 通信代理, 和许多其他建立在TCP和UDP之上的服务之上。

端口
~~~~~~

服务存活于主机内。IP地址可以定位主机。但在每台计算机上可能会提供多种服务，需要一个简单的方法对它们加以区分。TCP，UDP，SCTP或者其他协议使用端口号来加以区分。这里使用一个1到65,535的无符号整数，每个服务将这些端口号中的一个或多个相关联。

有很多“标准”的端口。Telnet服务通常使用端口号23的TCP协议。DNS使用端口号53的TCP或UDP协议。FTP使用端口21和20的命令，进行数据传输。HTTP通常使用端口80，但经常使用，端口8000，8080和8088，协议为TCP。X Window系统往往需要端口6000-6007，TCP和UDP协议。

在Unix系统中, /etc/services文件列出了常用的端口。Go语言有一个函数可以获取该文件。

.. code-block:: go

    func LookupPort(network, service string) (port int, err os.Error)
    
network是一个字符串例如"tcp"或"udp", service也是一个字符串，如"telnet"或"domain"(DNS)。

示例程序如下

.. code-block:: go

    /* LookupPort
     */

    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {
            if len(os.Args) != 3 {
                    fmt.Fprintf(os.Stderr,
                            "Usage: %s network-type service\n",
                            os.Args[0])
                    os.Exit(1)
            }
            networkType := os.Args[1]
            service := os.Args[2]

            port, err := net.LookupPort(networkType, service)
            if err != nil {
                    fmt.Println("Error: ", err.Error())
                    os.Exit(2)
            }

            fmt.Println("Service port ", port)
            os.Exit(0)
    }

举个例子, 运行LookupPort tcp telnet 打印 Service port: 23

TCPAddr 类型
~~~~~~~~~~~~~~

TCPAddr类型包含一个IP和一个port的结构：

.. code-block:: go

    type TCPAddr struct {
        IP   IP
        Port int
    }

函数ResolveTCPAddr用来创建一个TCPAddr

.. code-block:: go

    func ResolveTCPAddr(net, addr string) (*TCPAddr, os.Error)

net是"tcp", "tcp4"或"tcp6"其中之一，addr是一个字符串，由主机名或IP地址，以及":"后跟随着端口号组成，例如： "www.google.com:80" 或 '127.0.0.1:22"。如果地址是一个IPv6地址，由于已经有冒号，主机部分，必须放在方括号内, 例如："[::1]:23". 另一种特殊情况是经常用于服务器, 主机地址为0, 因此，TCP地址实际上就是端口名称, 例如：":80" 用来表示HTTP服务器。

TCP套接字
-----------

当你知道如何通过网络和端口ID查找一个服务时，然后呢？如果你是一个客户端，你需要一个API，让您连接到服务，然后将消息发送到该服务，并从服务读取回复。

如果你是一个服务器，你需要能够绑定到一个端口，并监听它。当有消息到来，你需要能够读取它并回复客户端。

net.TCPConn是允许在客户端和服务器之间的全双工通信的Go类型。两种主要方法是：

.. code-block:: go

    func (c *TCPConn) Write(b []byte) (n int, err os.Error)
    func (c *TCPConn) Read(b []byte) (n int, err os.Error)   

TCPConn被客户端和服务器用来读写消息。

TCP客户端
~~~~~~~~~~

一旦客户端已经建立TCP服务, 就可以和对方设备"通话"了. 如果成功，该调用返回一个用于通信的TCPConn。客户端和服务器通过它交换消息。通常情况下，客户端使用TCPConn写入请求到服务器, 并从TCPConn的读取响应。持续如此，直到任一（或两者）的两侧关闭连接。客户端使用该函数建立一个TCP连接。

.. code-block:: go

    func DialTCP(net string, laddr, raddr *TCPAddr) (c *TCPConn, err os.Error)
    
其中 laddr 是本地地址，通常设置为 nil 和 raddr 是一个服务的远程地址, net 是一个字符串，根据您是否希望是一个 TCPv4 连接， TCPv6 连接来设置为" tcp4 ", " tcp6 "或" tcp "中的一个，当然你也可以不关心链接形式。

一个简单的例子，展示个客户端连接到一个网页(HTTP)服务器。在后面的章节，我们将处理大量的HTTP客户端和服务器细节，现在我们先从简单的看看。

客户端可能发送的消息之一就是“HEAD”消息。这用来查询服务器的信息和文档信息。 服务器返回的信息，不返回文档本身。发送到服务器的请求可能是

.. code-block:: go

    "HEAD / HTTP/1.0\r\n\r\n"
    
这是在请求服务器的根文件信息。 一个典型的响应可能是

.. code-block:: go

    HTTP/1.0 200 OK
    ETag: "-9985996"
    Last-Modified: Thu, 25 Mar 2010 17:51:10 GMT
    Content-Length: 18074
    Connection: close
    Date: Sat, 28 Aug 2010 00:43:48 GMT
    Server: lighttpd/1.4.23
    
我们首先通过(GetHeadInfo.go)程序来建立TCP连接，发送请求字符串，读取并打印响应。编译后就可以调用，例如：

.. code-block:: go

    GetHeadInfo www.google.com:80

程序

.. code-block:: go

    /* GetHeadInfo
     */
    package main

    import (
            "net"
            "os"
            "fmt"
            "io/ioutil"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s host:port ", os.Args[0])
                    os.Exit(1)
            }
            service := os.Args[1]

            tcpAddr, err := net.ResolveTCPAddr("tcp4", service)
            checkError(err)

            conn, err := net.DialTCP("tcp", nil, tcpAddr)
            checkError(err)

            _, err = conn.Write([]byte("HEAD / HTTP/1.0\r\n\r\n"))
            checkError(err)

            //result, err := readFully(conn)
     result, err := ioutil.ReadAll(conn)
            checkError(err)

            fmt.Println(string(result))

            os.Exit(0)
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

第一个要注意的点是近乎多余的错误检查。因为正常情况下，网络程序失败的机会大大超过单机的程序。在客户端，服务器端或任何路由和中间交换上，硬件可能失败；通信可能会被防火墙阻塞;因网络负载可能会出现超时;当客户端联系服务器，服务器可能会崩溃，下列检查是必须的：

- 指定的地址中可能存在语法错误
- 尝试连接到远程服务可能会失败。例如, 所请求的服务可能没有运行, 或者有可能是主机没有连接到网络
- 虽然连接已经建立，如果连接突然丢失也可能导致写失败，或网络超时
- 同样，读操作也可能会失败

值得一提的是,如何从服务端读取数据。在这种情况下，读本质上是一个单一的来自服务器的响应，这将终止文件结束的连接。但是，它可能包括多个TCP数据包，所以我们需要不断地读，直到文件的末尾。在io/ioutil下的ReadAll函数考虑这些问题，并返回完整响应。(感谢Roger Peppe在golang-nuts上的邮件列表。)。

有一些涉及语言的问题，首先, 大多数函数返回两个值, 第二个值是可能出现的错误。如果没有错误发生, 那么它的值为nil。在C中, 如果需要的话，同样的行为通过定义特殊值例如NULL, 或 -1, 或0来返回。在Java中, 同样的错误检查通过抛出和捕获异常来管理，它会使代码看起来很凌乱。

在这个程序的早期版本, 我在返回结果中返回buf数组, 它的类型是[512]byte。我试图强迫类型为一个字符串但失败了- 只有字节数组类型[]byte可以强制转换。这确实有点困扰。

一个时间（ Daytime ）服务器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

最简单的服务，我们可以建立是时间(Daytime)服务。这是一个标准的互联网服务, 由RFC 867定义, 默认的端口13, 协议是TCP和UDP。很遗憾, 对安全的偏执，几乎没有任何站点运行着时间(Daytime)服务器。不过没关系，我们可以建立我们自己的。 (对于那些有兴趣, 你可以在你的系统安装inetd, 你通常可以得到一个时间(Daytime)服务器。)

在一个服务器上注册并监听一个端口。然后它阻塞在一个"accept"操作，并等待客户端连接。当一个客户端连接, accept调用返回一个连接(connection)对象。时间(Daytime)服务非常简单，只是将当前时间写入到客户端, 关闭该连接，并继续等待下一个客户端。

有关调用

.. code-block:: go

    func ListenTCP(net string, laddr *TCPAddr) (l *TCPListener, err os.Error)
    func (l *TCPListener) Accept() (c Conn, err os.Error)

net参数可以设置为字符串"tcp", "tcp4"或者"tcp6"中的一个。如果你想监听所有网络接口，IP地址应设置为0，或如果你只是想监听一个简单网络接口，IP地址可以设置为该网络的地址。如果端口设置为0，O/S会为你选择一个端口。否则，你可以选择你自己的。需要注意的是，在Unix系统上，除非你是监控系统，否则不能监听低于1024的端口，小于128的端口是由IETF标准化。该示例程序选择端口1200没有特别的原因。TCP地址如下":1200" - 所有网络接口, 端口1200。

程序

.. code-block:: go

    /* DaytimeServer
     */
    package main

    import (
            "fmt"
            "net"
            "os"
            "time"
    )

    func main() {

            service := ":1200"
            tcpAddr, err := net.ResolveTCPAddr("ip4", service)
            checkError(err)

            listener, err := net.ListenTCP("tcp", tcpAddr)
            checkError(err)

            for {
                    conn, err := listener.Accept()
                    if err != nil {
                            continue
                    }

                    daytime := time.Now().String()
                    conn.Write([]byte(daytime)) // don't care about return value
             conn.Close()                // we're finished with this client
     }
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

如果你运行该服务器, 它会在那里等待， 没有做任何事。当一个客户端连接到该服务器, 它会响应发送时间(Daytime)字符串，然后继续等待下一个客户端。

相比客户端服务器更要注意对错误的处理。服务器应该永远运行，所以，如果出现任何错误与客户端，服务器只是忽略客户端继续运行。否则，客户端可以尝试搞砸了与服务器的连接，并导致服务器宕机。

我们还没有建立一个客户端。这很简单，只是改变以前的客户端省略的初始写入。另外, 只需打开一个telnet连接到该主机：

.. code-block:: shell

    telnet localhost 1200
    
输出如下：

.. code-block:: shell

    $telnet localhost 1200
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    Sun Aug 29 17:25:19 EST 2010Connection closed by foreign host.

服务器输出："Sun Aug 29 17:25:19 EST 2010"。

多线程服务器
~~~~~~~~~~~~~~~~~


"echo"是另一种简单的IETF服务。只是读取客户端数据，并将其发送回去:

.. code-block:: go

    /* SimpleEchoServer
     */
    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {

            service := ":1201"
            tcpAddr, err := net.ResolveTCPAddr("tcp4", service)
            checkError(err)

            listener, err := net.ListenTCP("tcp", tcpAddr)
            checkError(err)

            for {
                    conn, err := listener.Accept()
                    if err != nil {
                            continue
                    }
                    handleClient(conn)
                    conn.Close() // we're finished
     }
    }

    func handleClient(conn net.Conn) {
            var buf [512]byte
            for {
                    n, err := conn.Read(buf[0:])
                    if err != nil {
                            return
                    }
                    fmt.Println(string(buf[0:]))
                    _, err2 := conn.Write(buf[0:n])
                    if err2 != nil {
                            return
                    }
            }
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

工作时，此服务器有一个明显的问题: 它是单线程的。当有一个客户端连接到它，就没有其他的客户端可以连接上。其他客户端将被阻塞，可能会超时。幸好客户端很容易使用go-routine扩展。我们仅仅需要把连接关闭移到处理程序结束后，示例代码如下：

.. code-block:: go

    /* ThreadedEchoServer
     */
    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {

            service := ":1201"
            tcpAddr, err := net.ResolveTCPAddr("ip4", service)
            checkError(err)

            listener, err := net.ListenTCP("tcp", tcpAddr)
            checkError(err)

            for {
                    conn, err := listener.Accept()
                    if err != nil {
                            continue
                    }
                    // run as a goroutine
             go handleClient(conn)
            }
    }

    func handleClient(conn net.Conn) {
            // close connection on exit
     defer conn.Close()

            var buf [512]byte
            for {
                    // read upto 512 bytes
             n, err := conn.Read(buf[0:])
                    if err != nil {
                            return
                    }

                    // write the n bytes read
             _, err2 := conn.Write(buf[0:n])
                    if err2 != nil {
                            return
                    }
            }
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

控制TCP连接
------------

超时
~~~~~~

服务端会断开那些超时的客户端，如果他们响应不够快，比如没有及时往服务端写一个请求。这应该是长时间(几分钟)的，因为用户可能花费了时间。相反, 客户端可能希望超时服务器(一个更短的时间后)。通过下面的来实现这两种：

.. code-block:: go

    func (c *TCPConn) SetTimeout(nsec int64) os.Error

套接字读写前。

存活状态
~~~~~~~~~~

即使没有任何通信，一个客户端可能希望保持连接到服务器的状态。可以使用

.. code-block:: go

    func (c *TCPConn) SetKeepAlive(keepalive bool) os.Error
    
还有几个其他的连接控制方法, 可以查看"net"包。

UDP数据报
~~~~~~~~~~~

在一个无连接的协议中，每个消息都包含了关于它的来源和目的地的信息。没有"session"建立在使用长寿命的套接字。UDP客户端和服务器使用的数据包，单独包含来源和目的地的信息。除非客户端或服务器这样做，否则消息的状态不会保持。这些消息不能保证一定到达，也可能保证按顺序到达。

客户端最常见的情况发送消息，并希望响应正常到达。服务器最常见的情况为将收到一条消息，然后发送一个或多个回复给客户端。而在点对点的情况下， 服务器可能仅仅是把消息转发到其他点。

Go下处理TCP和UDP之间的主要区别是如何处理多个客户端可能同时有数据包到达，没有一个管理TCP会话的缓冲。主要需要调用的是

.. code-block:: go

    func ResolveUDPAddr(net, addr string) (*UDPAddr, os.Error)
    func DialUDP(net string, laddr, raddr *UDPAddr) (c *UDPConn, err os.Error)
    func ListenUDP(net string, laddr *UDPAddr) (c *UDPConn, err os.Error)
    func (c *UDPConn) ReadFromUDP(b []byte) (n int, addr *UDPAddr, err os.Error
    func (c *UDPConn) WriteToUDP(b []byte, addr *UDPAddr) (n int, err os.Error)

UDP时间服务的客户端并不需要做很多的变化，仅仅改变...TCP...调用为...UDP...调用：

.. code-block:: go

    /* UDPDaytimeClient
     */
    package main

    import (
            "net"
            "os"
            "fmt"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s host:port", os.Args[0])
                    os.Exit(1)
            }
            service := os.Args[1]

            udpAddr, err := net.ResolveUDPAddr("up4", service)
            checkError(err)

            conn, err := net.DialUDP("udp", nil, udpAddr)
            checkError(err)

            _, err = conn.Write([]byte("anything"))
            checkError(err)

            var buf [512]byte
            n, err := conn.Read(buf[0:])
            checkError(err)

            fmt.Println(string(buf[0:n]))

            os.Exit(0)
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error ", err.Error())
                    os.Exit(1)
            }
    }

服务器也有很少的改动：

.. code-block:: go

    /* UDPDaytimeServer
     */
    package main

    import (
            "fmt"
            "net"
            "os"
            "time"
    )

    func main() {

            service := ":1200"
            udpAddr, err := net.ResolveUDPAddr("up4", service)
            checkError(err)

            conn, err := net.ListenUDP("udp", udpAddr)
            checkError(err)

            for {
                    handleClient(conn)
            }
    }

    func handleClient(conn *net.UDPConn) {

            var buf [512]byte

            _, addr, err := conn.ReadFromUDP(buf[0:])
            if err != nil {
                    return
            }

            daytime := time.Now().String()

            conn.WriteToUDP([]byte(daytime), addr)
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error ", err.Error())
                    os.Exit(1)
            }
    }

服务器侦听多个套接字
----------------------

一个服务器可能不止在一个端口监听多个客户端，或是更多端口，在这种情况下，它在端口之间使用某种轮询机制。

在C中, 调用的内核select()可以完成这项工作。 调用需要一个文件描述符的数字。该进程被暂停。当I/O准备好其中一个，一个唤醒被完成，并且该过程可以继续。This is cheaper than busy polling. 在G中, 完成相同的功能，通过为每个端口使用一个不同的goroutine。低级别的select()时发现，I/O已经准备好该线程，一个线程将运行。

Conn，PacketConn和Listener类型
-------------------------------

迄今为止我们已经区分TCP和UDP API的不同，使用例子DialTCP和DialUDP分别返回一个TCPConn和 UDPConn。Conn类型是一个接口，TCPConn和UDPConn实现了该接口。在很大程度上，你可以通过该接口处理而不是用这两种类型。

你可以使用一个简单的函数，而不是单独使用TCP和UDP的dial函数。

.. code-block:: go

    func Dial(net, laddr, raddr string) (c Conn, err os.Error)
    
net可以是"tcp", "tcp4" (IPv4-only), "tcp6" (IPv6-only), "udp", "udp4" (IPv4-only), "udp6" (IPv6-only), "ip", "ip4" (IPv4-only)和"ip6" (IPv6-only)任何一种。它将返回一个实现了Conn接口的类型。注意此函数接受一个字符串而不是raddr地址参数，因此，使用此程序可避免的地址类型。

使用该函数需要对程序轻微的调整。例如, 前面的程序从一个Web页面获取HEAD信息可以被重新写为

.. code-block:: go
    
    /* IPGetHeadInfo
     */
    package main

    import (
            "bytes"
            "fmt"
            "io"
            "net"
            "os"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Fprintf(os.Stderr, "Usage: %s host:port", os.Args[0])
                    os.Exit(1)
            }
            service := os.Args[1]

            conn, err := net.Dial("tcp", service)
            checkError(err)

            _, err = conn.Write([]byte("HEAD / HTTP/1.0\r\n\r\n"))
            checkError(err)

            result, err := readFully(conn)
            checkError(err)

            fmt.Println(string(result))

            os.Exit(0)
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

    func readFully(conn net.Conn) ([]byte, error) {
            defer conn.Close()

            result := bytes.NewBuffer(nil)
            var buf [512]byte
            for {
                    n, err := conn.Read(buf[0:])
                    result.Write(buf[0:n])
                    if err != nil {
                            if err == io.EOF {
                                    break
                            }
                            return nil, err
                    }
            }
            return result.Bytes(), nil
    }

使用该函数同样可以简化一个服务器的编写

.. code-block:: go

    func Listen(net, laddr string) (l Listener, err os.Error)
    
返回一个实现Listener接口的对象. 该接口有一个方法

.. code-block:: go

    func (l Listener) Accept() (c Conn, err os.Error)

这将允许构建一个服务器。使用它, 将使前面给出的多线程Echo服务器改变

.. code-block:: go

    /* ThreadedIPEchoServer
     */
    package main

    import (
            "fmt"
            "net"
            "os"
    )

    func main() {

            service := ":1200"
            listener, err := net.Listen("tcp", service)
            checkError(err)

            for {
                    conn, err := listener.Accept()
                    if err != nil {
                            continue
                    }
                    go handleClient(conn)
            }
    }

    func handleClient(conn net.Conn) {
            defer conn.Close()

            var buf [512]byte
            for {
                    n, err := conn.Read(buf[0:])
                    if err != nil {
                            return
                    }
                    _, err2 := conn.Write(buf[0:n])
                    if err2 != nil {
                            return
                    }
            }
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

如果你想写一个UDP服务器, 这里有一个PacketConn的接口，和一个实现了该接口的方法：

.. code-block:: go

    func ListenPacket(net, laddr string) (c PacketConn, err os.Error)
    
这个接口的主要方法ReadFrom和WriteTo用来处理数据包的读取和写入。

Go的net包建议使用接口类型而不是具体的实现类型。但是，通过使用它们，你失去了具体的方法，比如SetKeepAlive或TCPConn和UDPConn的SetReadBuffer，除非你做一个类型转换。如何选择在于你。

原始套接字和IPConn类型
-------------------------

本节涵盖了大多数程序员可能需要的高级资料。它涉及raw sockets, ，允许程序员建立自己的IP协议，或使用TCP或UDP协议。

TCP和UDP并不是建立在IP层之上唯一的协议。该网站：http://www.iana.org/assignments/protocol-numbers 列表上大约有140关于它们(该列表往往在Unix系统的/etc/protocols文件上。)。TCP和UDP在这个名单上分别为6和17。

Go允许你建立所谓的原始套接字，使您可以使用这些其它协议通信，或甚至建立你自己的。但它提供了最低限度的支持: 它会连接主机, 写入和读取和主机之间的数据包。在接下来的章节中，我们将着眼于设计和实现自己的基于TCP之上的协议; 这部分认为同样的问题存在于IP层。

为了简单起见，我们将使用几乎最简单的例子: 如何发送一个ping消息给主机。Ping使用"echo"命令的ICMP协议。这是一个面向字节协议, 客户端发送一个字节流到另一个主机, 并等待主机的答复。格式如下：

- 首字节是8, 表示echo消息
- 第二个字节是0
- 第三和第四字节是整个消息的校验和
- 第五和第六字节是一个任意标识
- 第七和第八字节是一个任意的序列号
- 该数据包的其余部分是用户数据

下面的程序将准备一个IP连接，发送一个ping请求到主机，并得到答复。您可能需要root权限才能运行成功。

.. code-block:: go

    /* Ping
     */
    package main

    import (
            "bytes"
            "fmt"
            "io"
            "net"
            "os"
    )

    func main() {
            if len(os.Args) != 2 {
                    fmt.Println("Usage: ", os.Args[0], "host")
                    os.Exit(1)
            }

            addr, err := net.ResolveIPAddr("ip", os.Args[1])
            if err != nil {
                    fmt.Println("Resolution error", err.Error())
                    os.Exit(1)
            }

            conn, err := net.DialIP("ip4:icmp", addr, addr)
            checkError(err)

            var msg [512]byte
            msg[0] = 8  // echo
     msg[1] = 0  // code 0
     msg[2] = 0  // checksum, fix later
     msg[3] = 0  // checksum, fix later
     msg[4] = 0  // identifier[0]
     msg[5] = 13 //identifier[1]
     msg[6] = 0  // sequence[0]
     msg[7] = 37 // sequence[1]
     len := 8

            check := checkSum(msg[0:len])
            msg[2] = byte(check >> 8)
            msg[3] = byte(check & 255)

            _, err = conn.Write(msg[0:len])
            checkError(err)

            _, err = conn.Read(msg[0:])
            checkError(err)

            fmt.Println("Got response")
            if msg[5] == 13 {
                    fmt.Println("identifier matches")
            }
            if msg[7] == 37 {
                    fmt.Println("Sequence matches")
            }

            os.Exit(0)
    }

    func checkSum(msg []byte) uint16 {
            sum := 0

            // assume even for now
     for n := 1; n < len(msg)-1; n += 2 {
                    sum += int(msg[n])*256 + int(msg[n+1])
            }
            sum = (sum >> 16) + (sum & 0xffff)
            sum += (sum >> 16)
            var answer uint16 = uint16(^sum)
            return answer
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }

    func readFully(conn net.Conn) ([]byte, error) {
            defer conn.Close()

            result := bytes.NewBuffer(nil)
            var buf [512]byte
            for {
                    n, err := conn.Read(buf[0:])
                    result.Write(buf[0:n])
                    if err != nil {
                            if err == io.EOF {
                                    break
                            }
                            return nil, err
                    }
            }
            return result.Bytes(), nil
    }
    
结论
-----

本章着重IP, TCP和UDP级别的编程。如果你想实现自己的协议，或用现有的协议建立一个客户端或服务器，这些内容往往很重要。

版权所有 Jan Newmarch, jan@newmarch.name




























    