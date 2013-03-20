数据序列化
==============

    客户端与服务之间通过数据交换来通信。因为数据可能是高度结构化的，所以在传输前必须进行序列化。这一章将研究序列化基础并介绍一些Go API提供的序列化技术。
    
简介
--------------
    
客户端与服务器需要通过消息来交换信息。TCP与UDP是消息传递的两种机制，在这两种机制之上就需要有合适的协议来约定传输的内容的含义。

在网络上，消息被当作字节序列来传输，它们是没有结构的，仅仅只是一串字节流。我们将在下一章讨论定义消息与协议涉及到的的各种问题。本章，我们只重点关注消息的一个方面 - 被传输的数据。

程序通常构造一个复杂的数据结构来保存其自身当前的状态。在与远程的客户端或服务的交互中，程序会通过网络将这样的数据结构传输到 -应用程序所在的地址空间之外的地方。

编程语言使用的结构化的数据类型有：

- 记录/结构
- 可变记录
- 数组 - 固定大小或可变大小
- 字符串 - 固定大小或可变大小
- 表 - 例如:记录构成的数组
- 非线程结构，比如
    - 循环链表
    - 二叉树
    - 含有其他对象引用的对象
    
IP，TCP或者UDP网络包并不知道这些数据类型的含义，它们只是字节序列的载体。因此，写入网络包的时候，应用需要将要传输的(有类型的)数据 序列化 成字节流，反之，读取网络包的时候，应用需要将字节流反序列化成合适的数据结构，这两个操作被分别称为编组和解组。

例如:考虑发送如下这样一个由两列可变长度字符串构成的可变长度的表格

---------------------
fred    |programmer
liping	|analyst
sureerat|manager
---------------------

这可以通过多种方式来完成。比如：假设知道数据是一个未知行数的两列表格，那么编组形式可能是：

.. code-block:: go
    3                // 3 rows, 2 columns assumed
    4 fred           // 4 char string,col 1
    10 programmer    // 10 char string,col 2
    6 liping         // 6 char string, col 1
    7 analyst        // 7 char string, col 2
    8 sureerat       // 8 char string, col 1
    7 manager        // 7 char string, col 2
    
可变长度的事物都可以通过用一个“非法”的终结值,比如对于字符串来说的'\0',来间接获得它们的长度：

.. code-block:: go
    3
    fred\0        
    programmer\0
    liping\0
    analyst\0
    sureerat\0
    manager\0
    
假设知道数据是一个三行两列且每列长度分别是8或10的表格，那么序列化的结果可能是：

.. code-block:: go
    fred\0\0\0\0
    programmer
    liping\0\0
    analyst\0\0\0
    sureerat
    manager\0\0\0
    
这些格式中的任意一种都是可行的 - 但是消息交换协议必须指定使用哪一种(格式)，或者约定在运行期再做决定。


交互协议
----------

前一小节总结了在数据序列化过程中可能遇到的各种问题。而在实际操作中，需要考虑的细节还更多一些，例如：先考虑下面这个问题，如何将下面这个表编组成流。

.. code-block:: go
    3
    4 fred
    10 programmer
    6 liping
    7 analyst
    8 sureerat
    7 manager

许多问题冒出来了。例如：这个表格可能有多少行？- 即我们需要多大的整数来表示表格的大小，如果它只有255行或者更少，那么一个字节就够了，如果更大一些，就可能需要short，integer或者long来表示了。对于字符串的长度也存在同样的问题，对字符本身来说，它们属于哪种字符集? 7位的ASCII？16位的Unicode？字符集的问题将会在后面的章节里详细讨论。

上面的序列化是不透明的或者被称为隐式的，如果采用这种格式来编组数据，那么序列化后的数据中没有包含任何指示它应该被如何解组的信息。为了正确的解组，解组的一端需要精确的知晓编组的方式。如果数据的行数以8位整型数的方式编组，却以16位整型的方式解组，那么接收者将得到错误的解码结果。比如接受者尝试将3与4当作16位整型解组，在后续的程序运行的时候肯定会失败。

早期比较出名的序列化方法是Sun公司的RPC中使用的XDR(外部资料表示法)。后来就是ONC(开放式网络运算)。XDR由 RFC 1832定义，阅读一下这个规范的详细定义是有意义的，即便如此，由于序列化的数据中不包含类型信息，XDR是天生不安全的。ONC中主要通过由编译器为编、解组生成额外的代码来确保数据的正确性。

Go没有为编、解组不透明的序列化数据提供显式的支持,标准包中的RPC包也没有使用XDR，而是使用了这一章后面的小节中将要介绍的gob来作为替代方案。

自描述数据
-----------

自描述数据在最终的结果数据中附带了类型信息,例如，前面提到的数据可能被编码为：

.. code-block:: go
    table
       uint8 3
       uint 2
    string
       uint8 4
       []byte fred
    string
       uint8 10 
       []byte programmer
    string
       uint8 6 
       []byte liping
    string
       uint8 7 
       []byte analyst
    string
       uint8 8 
       []byte sureerat
    string
       uint8 7
       []byte manager

当然，实际使用的编码方式不会如此啰嗦。小整数可能被用作类型标记，并且整个数据编码后的字节数组会尽量的小（XML是一个反例）。原则就是编组器会在序列化后的数据中包含类型信息。解组器知道类型生成的规则，并使用此规则重组出正确的数据结构。

抽象语法表示法
-------------------
    
抽象语法表示法/1(ASN.1)最初出现在1984年，它是一个为电信行业设计的复杂标准，Go的标准包asn1实现了它的一个子集，它可以将复杂的数据结构序列化成自描述的数据。在当前的网络系统中，它主要用于对认证系统中普遍使用的X.509证书的编码。Go对ASN.1的支持主要是X.509证书的读写上。

以下两个函数用以对数据的编、解组

.. code-block:: go
    func Marshal(val interface{}) ([]byte, os.Error)
    func Unmarshal(val interface{}, b []byte) (rest []byte, err os.Error)

前一个将数据值编组成序列化的字节数组，后一个将其解组出来，需要对interface类型的参数进行更多的类型检查。编组时，我们只需要传递某个类型的变量的值即可，解组它，则需要一个与被序列化过的数据匹配的确定类型的变量，我们将在后面讨论这部分的细节 。除了有确定类型的变量外，我们同时需要保证那个变量的内存已经被分配，以使被解组后的数据能有实际被写入的地址。


我们将举一个整数编、解组的小例子。在这个例子中。我们先将一个整数传递给Marshal得到一个字节数组，然后又将此数组解组成一个整数。

.. code-block:: go
    /* ASN.1
     */

    package main

    import (
            "encoding/asn1"
            "fmt"
            "os"
    )

    func main() {
            mdata, err := asn1.Marshal(13)
            checkError(err)

            var n int
            _, err1 := asn1.Unmarshal(mdata, &n)
            checkError(err1)

            fmt.Println("After marshal/unmarshal: ", n)
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }
    
当然，被解组后的值，是13。

一旦我们越过了这个小关卡，事情开始变得复杂。为了管理更复杂的数据类型，我们需要更深入的了解ASN.1支持的数据类型，以及Go是如何支持ASN.1的。

任何序列化方法都只能处理某些数据类型，而对其他的数据类型无能为力。因此为了评估类似ASN.1等序列化方案的可行性，你必须先将要在程序中使用的数据类型与它们支持的数据类型做个比较，下面是ASN.1支持的数据类型，它们来自于 http://www.obj-sys.com/asn1tutorial/node4.html 。
    
简单数据类型有：

- BOOLEAN：两态变量值
- INTEGER：表征整型变量值
- BIT STRING：表征任意长度的二进制数据
- OCT STRING：表征长度是8的倍数的二进制数据
- NULL：指示一个没有有效数据的序列
- OBJECT IDENTIFIER：命名信息对象
- REAL：表征一个real变量值
- ENUMERATED：表征一个至少有三个状态的变量值
- CHARACTER STRING：表征一个字符串值

字符串可以来自于确定的字符集
    
- NumericString: 0,1,2,3,4,5,6,7,8,9, 与空格(space)
- PrintableString: 大、小写字母，数字，空格，省略号，左、右小括号，加号，逗号，连字符，句号，斜线，冒号，等号，问号
- TeletexString(T61String): CCITT的Teletex字符集中的T61，空格和删除(delete)
- VideotexString:CCITT的Videotex字符集中的T.100与T.101, 空格和删除(delete)
- VisibleString (ISO646String):国际ASCII中的打印字符集和空格
- IA5String:国际字母表5(国际ASCII)
- GraphicString:所有被注册的G集和空格

最后,以下是结构化的类型：

- SEQUENCE:表征不同类型变量构成的有序集合
- SEQUENCE OF: 表征相同类型的变量构成的有序集合
- SET: 表征不同类型的变量构成的无序集合
- SET OF:表征相同类型的变量构成的有序集合
- CHOICE:从一个不同类型构成的特定集合中选出一个类型
- SELECTION: 从一个特定的CHOICE类型中选取一个组件类型
- ANY:启用一个用以指定类型的应用. 注意:ANY是一个弃用的ASN.1结构类型,它被x.680的 Open Type所替代

不是以上所有的类型、可能的值都被Go支持，在Go 'asn1'包文档中定义的规则如下：

- ASN.1 INTEGER 可以被写入int或者int64中. 如果被编码的值与Go类型不匹配,Unmarshal将返回一个解析错误.
- ASN.1 BIT STRING 可以被写入BitString中.
- ASN.1 OCT STRING可以被写入[]byte中.
- ASN.1 OBJECT IDENTIFIER 可以被写入ObjectIdentifier中.
- ASN.1 ENUMERATED 可以被写入Enumerated中.
- ASN.1 UTCTIME或者GENERALIZEDTIME可以被写入*time.Time中.
- ASN.1 PrintableString 或者 IA5String可以被写入string中.
- 以上的任何ASN.1类型的值都可以作为对应的Go类型的值写入interface{}中。比如整数放入interface{}的话，它对应的类型是int64。
- 如果一个变量x可以被当做某个类型写入，那么ASN.1中的x构成的有序列或者集合就可以当做这个类型的slice写入了。
- 如果某个有序列或者集合中的所有元素都可以被写入到某个结构里与之对应的元素中，那么此ASN.1 SEQUENCE 或者SET就可以写入到这个结构中。

Go在实现上,为ASN.1添加了一些约束。例如ASN.1允许任意大小的整数,而GO只允许最大为64位有符号整数能表示的值.另一方面，Go区分有符号类型与无符号类型,而在ASN.1则没有分别.因此传递一个大于int64最大值能表示的uint64的值，则可能会失败。

同理，ASN.1允许多个不同的字符集,而Go只支持PrintableString和IA5String(ASCII). ASN.1不支持Unicode字符(它需要BMPString ASN.1扩展)，连Go中的基本Unicode字符集它都不支持，如果应用程序需要传输Unicode字符，则可能需要类似UTF-7的编码。有关编码的内容将会在后边字符集相关的章节来讨论。

我们已经看到，整型的值很容易被编、解组。类似的boolean与real等基本类型处理手法也类似。由ASCII字符构成的字符串也很容易。但当处理 "hello \u00bc"这种含有 '¼'这个非ASCII字符的字符串，则会出现错误：“ASN.1 结构错误:PrintableString包含非法字符”。以下的代码仅在处理由可打印字符（printable characters）构成的字符串时,工作良好。

.. code-block:: go
    s := "hello"
    mdata, _ := asn1.Marshal(s)

    var newstr string
    asn1.Unmarshal(mdata, &newstr)
    
ASN.1还包含一些未在上边列表中出现的“有用的类型(useful types)”, 比如UTC时间类型，GO支持此UTC时间类型。就是说你可以用一种特有的类型来传递时间值。ASN.1不支持指针,Go中却有指向时间值的指针。比如函数GetLocalTime返回*time.Time。asn1包编组这个time结构，也使用这个包解组到一个time.Time对象指针中。代码如下：

.. code-block:: go
    t := time.LocalTime()
    mdata, err := asn1.Marshal(t)

    var newtime = new(time.Time)
    _, err1 := asn1.Unmarshal(&newtime, mdata)
    
LocalTime与new函数都返回的是*time.Time类型的指针，GO将内部对这些特殊类型进行处理。

除了time这种特殊情况外，你可能要编、解组结构类型。除了上面提到的Time结构外，其他的结构Go还是很好处理的。类以new的操作将会创建指针，因此在编、解组之前，你需要解引用它。通常，Go会随需自动对指针进行解引用，但是下面这个例子并不是这么个情况。对于类型T，以下两种方式均可。

.. code-block:: go
    // using variables
    var t1 T
    t1 = ...
    mdata1, _ := asn1.Marshal(t)

    var newT1 T
    asn1.Unmarshal(&newT1, mdata1)

    /// using pointers
    var t2 = new(T)
    *t2 = ...
    mdata2, _ := asn1.Marshal(*t2)

    var newT2 = new(T)
    asn1.Unmarshal(newT2, mdata2)
    
恰当地的使用指针与变量能让代码工作得更好。

结构的所有字段必须是公共的，即字段名必须以大写字母开头。Go内部实际是使用reflect包来编、解组结构，因此reflect包必须能访问所有的字段。比如下面这个类型是不能被编组的：

.. code-block:: go
    type T struct {
        Field1 int
        field2 int // not exportable
    }

ASN.1只处理数据类型，它并不关心结构字段的名字。因此只要对应的字段类型相同那么下面的T1类型将可以被解、解组到T2类型中。
    
.. code-block:: go
    type T1 struct {
        F1 int
        F2 string
    }

    type T2 struct {
        FF1 int
        FF2 string
    }
    
不仅每个字段的类型必须匹配，而且字段数目也要相等，下面两个类型将不能互编、解码：

.. code-block:: go
    type T1 struct {
        F1 int
    }

    type T2 struct {
        F1 int
        F2 string // too many fields
    }
    
ASN.1 日期查询服务客户端与服务器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现在（最后）让我们使用ASN.1来跨网络传输数据

我们可以使用上一章的技术来编写一个将当前时间作为ASN.Time类型时间来传送的TCP服务器。服务器是：

.. code-block:: go
    /* ASN1 DaytimeServer
     */
    package main

    import (
            "encoding/asn1"
            "fmt"
            "net"
            "os"
            "time"
    )

    func main() {

            service := ":1200"
            tcpAddr, err := net.ResolveTCPAddr("tcp", service)
            checkError(err)

            listener, err := net.ListenTCP("tcp", tcpAddr)
            checkError(err)

            for {
                    conn, err := listener.Accept()
                    if err != nil {
                            continue
                    }

                    daytime := time.Now()
                    // Ignore return network errors.
             mdata, _ := asn1.Marshal(daytime)
                    conn.Write(mdata)
                    conn.Close() // we're finished
     }
    }

    func checkError(err error) {
            if err != nil {
                    fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
                    os.Exit(1)
            }
    }
        
    
    
---------------------------------------------

    
    
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

    
    
    
    
    