# 第8章：原始网络包的处理

在本章要学会抓包和处理网络包。包处理用在很多地方，包括抓取明文身份验证凭证、更改包的应用程序功能，或者欺骗和毒害流量。还可以将其用于SYN扫描和通过SYN-floor保护进行端口扫描等。

我们将介绍谷歌的优秀的 gopacket 包，该包能够解码数据包并重新组装通信流。该包可以使用 Berkeley Packet Filter \(BPF\) 来过滤流量，也称为tcpdump语法；读写 _.pcap_ 文件；检查各个层和数据；还有操作包。

我们将通过几个示例来演示如何识别设备、过滤结果和创建可以绕过 SYN-flood 保护的端口扫描器。

## EXPLOIT设置开发环境

在完成本章的代码之前，需要设置开发环境。首先，输入以下命令安装gopacket：

```text
$ go get github.com/google/gopacket
```

现在，gopacket 依赖外部库和驱动程序绕过操作系统的协议栈。如果打算在 Linux 或 macOS 编译本章中的例子的话，需要安装 _libpcap-dev_ 。使用大多数的包管理工具（如apt，yum，或 brew）来安装。下面使用 _apt_ 来安装（其余两个安装过程类型）：

```text
$ sudo apt-get install libpcap-dev
```

如果在Windows上编译运行本章中的例子，根据是否进行交叉编译，您有两个选项。如果不交叉编译的话设置开发环境相对简单点，但是在这种情况下，必须在 Windows 上创建 Go 开发环境，如果不想让另一个环境变得混乱，那么这个环境可能没有吸引力。目前，我们假设您有一个可以用来编译Windows二进制文件的工作环境。在该环境下需要安装 WinPcap 。从 [https://www.winpcap.org/](https://www.winpcap.org/) 下载免费版。

## 使用pcap子包识别设备

在抓包之前，必须确定可以监听的可用设备。可以使用 _gopacket/pcap_ 子包中的 `pcap.Find AllDevs() (ifs []Interface, err error)` 函数获取这些信息。清单8-1演示使用该函数列出所有可用的接口。

```go
package main

import (
    "fmt"
    "log"

    "github.com/google/gopacket/pcap"
)

func main() {
    devices, err := pcap.FindAllDevs()
    if err != nil {
        log.Panicln(err)
    }

    for _, device := range devices {
        fmt.Println(device.Name)
        for _, address := range device.Addresses {
            fmt.Printf("    IP:      %s\n", address.IP)
            fmt.Printf("    Netmask: %s\n", address.Netmask)
        }
    }
}
```

清单 8-1: 列出可用的网络设备 \([https://github.com/blackhat-go/bhg/ch-8/identify/main.go/](https://github.com/blackhat-go/bhg/ch-8/identify/main.go/)\)

调用 `pcap.FindAllDevs()` 来枚举设备。然后循环遍历找到的设备。访问每个设备的属性，包括 _device.Name_ 。通过属性 _Addresses_ 也能访问IP地址，该属性是 _pcap.InterfaceAddress_ 类型的切片。遍历循环他们的地址，将IP地址和掩码显示在屏幕上。

执行程序将产生类似于清单8-2的输出。

```text
$ go run main.go 
enp0s5
    IP: 10.0.1.20
    Netmask: ffffff00
    IP: fe80::553a:14e7:92d2:114b 
    Netmask: ffffffffffffffff0000000000000000
any 
lo
    IP: 127.0.0.1
    Netmask: ff000000
    IP: ::1
    Netmask: ffffffffffffffffffffffffffffffff
```

清单8-2：显示可用网络接口的输出

输出列出了可用的网络接口—— `enp0s5，any 和 lo` ——也就是他们的IPv4和IPv6地址和掩码。每个系统上的输出可能与这些网络细节不同，但应该足够相似，以便您能够理解这些信息。

## 实时抓包和过滤结果

既然您已经知道如何查询可用设备，那么就可以使用 `gopacket` 的特性来实时抓取数据包。在此过程中，还将使用**BPF** 语法过滤数据包集。**BPF** 能够限制抓取和显示的内容，以便只看相关的流量。通常根据协议和端口过滤流量。例如，可以创建一个过滤器来查看发送到端口80的所有TCP流量。还可以根据目标主机过滤流量。对BPF语法完整的论述超出了本书的范围。有关使用BPF的其他方法，请查看 [http://www.tcpdump.org/manpages/pcap-filter.7.html](http://www.tcpdump.org/manpages/pcap-filter.7.html) 。

清单8-3显示了过滤流量的代码，以便只抓取发送到端口80或从端口80发送的TCP流量。

```go
package main

import (
    "fmt"
    "log"

    "github.com/google/gopacket"
    "github.com/google/gopacket/pcap"
)

var (
    iface    = "enp0s5"
    snaplen  = int32(1600)
    promisc  = false
    timeout  = pcap.BlockForever
    filter   = "tcp and port 80"
    devFound = false
)

func main() {
    devices, err := pcap.FindAllDevs()
    if err != nil {
        log.Panicln(err)
    }

    for _, device := range devices {
        if device.Name == iface {
            devFound = true
        }
    }
    if !devFound {
        log.Panicf("Device named '%s' does not exist\n", iface)
    }

    handle, err := pcap.OpenLive(iface, snaplen, promisc, timeout)
    if err != nil {
        log.Panicln(err)
    }
    defer handle.Close()

    if err := handle.SetBPFFilter(filter); err != nil {
        log.Panicln(err)
    }

    source := gopacket.NewPacketSource(handle, handle.LinkType())
    for packet := range source.Packets() {
        fmt.Println(packet)
    }
}
```

清单 8-3: 使用 BPF 过滤抓取特定的网络流量 \([https://github.com/blackhat-go/bhg/ch-8/filter/main.go/](https://github.com/blackhat-go/bhg/ch-8/filter/main.go/)\)

代码首先定义设置抓包所需的几个变量。其中包括要抓取数据接口的名称，快照长度（每帧捕获的数据量），`promisc` 变量（表示是否混杂模式），和 `time-out`。还有BPF过滤器：`tcp and port 80`。这样就能只抓取符合这些条件的包。

在 `main()` 函数内，枚举可用的设备，遍历它们以确定所要的捕获接口是否存在于设备列表中。如果接口名不存在就 `panic`，说明这是无效的。

`main()` 函数中剩余的代码是抓包逻辑。从高层次的角度来看，首先需要获得或创建一个可以读取和注入包的 `*pcap.Handle`。使用这个句柄，就可以应用 **BPF** 过滤器并创建一个新的包数据源，可以从中读取包。

调用 `pcap.OpenLive()` 创建 `*pcap.Handle`（代码中命名为 `handle`）。该函数参数为接口名称，快照长度，一个定义的是否混杂的布尔值，和一个超时时间。这些变量都在 `main()` 函数的开始处定义，如前所述。调用 `handle.SetBPFFilter(filter)` 为句柄设置 **BPF** 过滤器，然后当调用 `gopacket .NewPacketSource(handle, handle.LinkType())` 时使用 `handle` 来创建新包数据源。第二个参数为 `handle.LinkType()` ，当处理包时用于解码。最后，循环遍历 `source.Packets()` 从网络中读取包，该函数返回的是一个管道。

可能还记得在本书前面的例子中，如果管道中没有数据的话，循环遍历管道来读取数据会阻塞。当收到包时，读取并输出其中的内容。

输出的内容类似于清单 8-4。请注意，该程序需要权限，因为是从网络读取原始内容。

```text
$ go build -o filter && sudo ./filter
PACKET: 74 bytes, wire length 74 cap length 74 @ 2020-04-26 08:44:43.074187 -0500 CDT 
- Layer 1 (14 bytes) = Ethernet {Contents=[..14..] Payload=[..60..] SrcMAC=00:1c:42:cf:57:11 DstMAC=90:72:40:04:33:c1 EthernetType=IPv4 Length=0}
- Layer 2 (20 bytes) = IPv4 {Contents=[..20..] Payload=[..40..] Version=4 IHL=5 TOS=0 Length=60 Id=998 Flags=DF FragOffset=0 TTL=64 Protocol=TCP Checksum=55712 SrcIP=10.0.1.20 DstIP=54.164.27.126 Options=[] Padding=[]}
- Layer 3 (40 bytes) = TCP {Contents=[..40..] Payload=[] SrcPort=51064 DstPort=80(http) Seq=3543761149 Ack=0 DataOffset=10 FIN=false SYN=true RST=false PSH=false ACK=false URG=false ECE=false CWR=false NS=false Window=29200 Checksum=23908 Urgent=0 Options=[..5..] Padding=[]}

PACKET: 74 bytes, wire length 74 cap length 74 @ 2020-04-26 08:44:43.086706 -0500 CDT - Layer 1 (14 bytes) = Ethernet {Contents=[..14..] Payload=[..60..] SrcMAC=00:1c:42:cf:57:11 DstMAC=90:72:40:04:33:c1 EthernetType=IPv4 Length=0}
- Layer 2 (20 bytes) = IPv4 {Contents=[..20..] Payload=[..40..] Version=4 IHL=5 TOS=0 Length=60 Id=23414 Flags=DF FragOffset=0 TTL=64 Protocol=TCP Checksum=16919 SrcIP=10.0.1.20 DstIP=204.79.197.203 Options=[] Padding=[]}
- Layer 3 (40 bytes) = TCP {Contents=[..40..] Payload=[] SrcPort=37314 DstPort=80(http) Seq=2821118056 Ack=0 DataOffset=10 FIN=false SYN=true RST=false PSH=false ACK=false URG=false ECE=false CWR=false NS=false Window=29200 Checksum=40285 Urgent=0 Options=[..5..] Padding=[]}
```

清单 8-4: 抓包日志

虽然原始输出不是很容易理解，但却有很好的分层。现在可以使用函数，如 `packet.ApplicationLayer() 和 packet.Data()` 来检索单个层或整个包的原始字节。当使用 `hex .Dump()` 结合输出时，就可以以易读的方式显示内容。可以自己尝试一下。

## 嗅探并明文显示用户凭证

现在编译代码。复用其他工具的一些功能，以嗅探并明文显示用户凭证。

现在，大多数组织使用交换网络进行操作，交换网络直接在两个端点之间发送数据，而不是广播发送，这使得在企业环境中被动地抓包变得更加困难。但是，接下来的明文嗅探攻击配合像 `Address Resolution Protocol (ARP)` 毒剂这样的东西非常有用。一种可以强制端点与交换网络上的恶意设备通信的攻击，或者当偷偷嗅探从被害的用户工作站发出的出站流量时。本例中，假定已经攻击了用户工作站，并只抓取使用FTP的流量来保持代码的简洁。

除了一些小的更改外，清单8-5中的代码几乎与清单8-3中的代码相同。

```go
package main

import (
    "bytes"
    "fmt"
    "log"

    "github.com/google/gopacket"
    "github.com/google/gopacket/pcap"
)

var (
    iface    = "enp0s5"
    snaplen  = int32(1600)
    promisc  = false
    timeout  = pcap.BlockForever
    filter   = "tcp and dst port 21"
    devFound = false
)

func main() {
    devices, err := pcap.FindAllDevs()
    if err != nil {
        log.Panicln(err)
    }

    for _, device := range devices {
        if device.Name == iface {
            devFound = true
        }
    }
    if !devFound {
        log.Panicf("Device named '%s' does not exist\n", iface)
    }

    handle, err := pcap.OpenLive(iface, snaplen, promisc, timeout)
    if err != nil {
        log.Panicln(err)
    }
    defer handle.Close()

    if err := handle.SetBPFFilter(filter); err != nil {
        log.Panicln(err)
    }

    source := gopacket.NewPacketSource(handle, handle.LinkType())
    for packet := range source.Packets() {
        appLayer := packet.ApplicationLayer()
        if appLayer == nil {
            continue
        }
        payload := appLayer.Payload()
        if bytes.Contains(payload, []byte("USER")) {
            fmt.Print(string(payload))
        } else if bytes.Contains(payload, []byte("PASS")) {
            fmt.Print(string(payload))
        }
    }
}
```

清单 8-5: 抓取FTP身份验证凭据\([https://github.com/blackhat-go/bhg/ch-8/ftp/main.go/](https://github.com/blackhat-go/bhg/ch-8/ftp/main.go/)\)

只更改了大约10行代码。首先，更改 **BPF** 过滤器为只抓取发送到21端口的流量（该端口一般用于FTP）。在处理包之前的剩余代码保持不变。

要想处理包，先提取包的应用层并检查其是否存在，因为应用层含有FTP命令和数据。通过检查 `packet.ApplicationLayer()` 的响应值是否是nil来查找应用层。假设包中存在应用层，通过调用 `appLayer.Payload()` 从应用层中提取有效值（FTP命令/数据）。（提取并检查其他层和数据也用类似的方法，但是只需要应用层的值。）提取数据后检查是否有 `USER` 或 `PASS` 命令，表明这是登录序列的一部分。如果是就再屏幕上输出数据。

下面是抓取一个FTP登录的示例：

```text
$ go build -o ftp && sudo ./ftp 
USER someuser
PASS passw0rd
```

当然也可以优化代码。本例中，如果数据中含有 `USER` 或 `PASS` 就输出该数据。实际上，代码应该只搜索有效数据的开头部分，以消除当这些关键字作为在客户端和服务器之间传输的文件内容的一部分出现，或者是更长的单词\(如PASSAGE或ABUSER\)的一部分出现时的误报。鼓励您将该改进作为学习。

## 通过SYN-flood保护的端口扫描

在第2章中已经了解了端口扫描器的创建过程。通过多次迭代来改进代码，直到实现可以高性能产出准确的结果。然而，某些情况下，扫描器任然会产出错误的结果。特别地，当组织使用 **SYN-flood** 保护时，通常所有的端口——打开、关闭和过滤的相似点——都会产生相同的包交换，表示端口是打开的。这些保护措施被称为 **SYN cookie** ，防止 **SYN-flood** 攻击和模糊攻击面，产生误报。

当目标使用 **SYN cookie** 时，如何确定是服务监听的端口，还是设备错误地显示端口是打开的？毕竟这两种情况下，都能完成TCP的三次握手。大多数的工具和扫描器（包括Nmap）都会查看这个序列（或者它的一些变体，基于所选择的扫描类型），以确定端口的状态。因此，不能依赖这些工具来产生准确的结果。

然而，如果考虑在建立了一个连接——数据交换——之后会发生什么，也许是以服务标语的形式——可以推断出实际的服务是否正在响应。**SYN-flood** 保护一般不会交换除最初三次握手之外的包，除非有服务正在监听，因此，存在任何附加包的话表明可能存在服务。

### 检查TCP Flag

为了解释 **SYN cookie**，必须扩展端口扫描功能，当建立连接后通过查看是否从目标处收到了除三次握手之外的包。可以通过嗅探数据包来完成，查看是否有数据包使用TCP Flag 值进行传输，该值指示附加的合法服务通信。

TCP Flag表示关于包传输状态的信息。如果查看TCP手册，会发现 flag 存储在包头部第14个位置的单个字节中。该字节的每一位代表一个Flag值。如果该位置的位设置为1，则flag为“on”;如果该位设置为0，则flag为“off”。根据TCP规范，表8-1显示了flag在字节中的位置。

**表 8-1：** TCP Flag 及在字节中的位置

| **Bit** | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
| :---: | :---: | :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| **Flag** | CWR | ECE | URG | ACK | PSH | RST | SYN | FIN |

知道了Flag 的位置就能创建过滤器检测他们。例如，可以查找包含下面 Flag 的包，这些 Flag 可能指示监听服务：

* ACK 和 FIN
* ACK
* ACK 和 PSH

因为使用 `gopacket` 库可以抓取并过滤某些包，可以构建连接远程服务的程序，嗅探数据包，并仅显示与这些TCP头通信的数据包的服务。假定所有其他服务由于SYN cookie而被错误地“打开”。

### 构建 BPF 过滤器

BPF 过滤器需要检查指示包传输的特定 flag 值。假如前面提到的 flag 是开启的，则 flag 字节有以下值：

* ACK 和 FIN: 00010001 \(0x11\)
* ACK: 00010000 \(0x10\)
* ACK 和 PSH: 00011000 \(0x18\)

为了清晰起见，使用了和二进制值相等的十六进制，因为在过滤器中使用十六进制值。

总而言之，需要检查TCP报头的第14字节\(基于0的索引的偏移量为13\)，仅过滤flag为0x11、0x10或0x18的数据包。下面是BPF过滤器的样子：

```go
tcp[13] == 0x11 or tcp[13] == 0x10 or tcp[13] == 0x18
```

太棒了，现在有过滤器了。

### 编写端口扫描器

现在，您将使用过滤器来构建实用程序，该实用程序将建立完整的TCP连接，并检查除三次握手之外的数据包，以查看是否传输了其他数据包，从而表明有真实的服务正在监听。程序如清单8-6所示。为了简单起见，没有对代码的性能做优化。但是，可以通过第2章中类似的优化来改进代码。

```go
package main

import (
    "fmt"
    "log"
    "net"
    "os"
    "strings"
    "time"

    "github.com/google/gopacket"
    "github.com/google/gopacket/pcap"
)

var (
    snaplen  = int32(320)
    promisc  = true
    timeout  = pcap.BlockForever
    filter   = "tcp[13] == 0x11 or tcp[13] == 0x10 or tcp[13] == 0x18"
    devFound = false
    results  = make(map[string]int)
)

func capture(iface, target string) {
    handle, err := pcap.OpenLive(iface, snaplen, promisc, timeout)
    if err != nil {
        log.Panicln(err)
    }
    defer handle.Close()

    if err := handle.SetBPFFilter(filter); err != nil {
        log.Panicln(err)
    }

    source := gopacket.NewPacketSource(handle, handle.LinkType())
    fmt.Println("Capturing packets")
    for packet := range source.Packets() {
        networkLayer := packet.NetworkLayer()
        if networkLayer == nil {
            continue
        }
        transportLayer := packet.TransportLayer()
        if transportLayer == nil {
            continue
        }

        srcHost := networkLayer.NetworkFlow().Src().String()
        srcPort := transportLayer.TransportFlow().Src().String()

        if srcHost != target {
            continue
        }
        results[srcPort] += 1
    }
}

func main() {

    if len(os.Args) != 4 {
        log.Fatalln("Usage: main.go <capture_iface> <target_ip> <port1,port2,port3>")
    }

    devices, err := pcap.FindAllDevs()
    if err != nil {
        log.Panicln(err)
    }

    iface := os.Args[1]
    for _, device := range devices {
        if device.Name == iface {
            devFound = true
        }
    }
    if !devFound {
        log.Panicf("Device named '%s' does not exist\n", iface)
    }

    ip := os.Args[2]
    go capture(iface, ip)
    time.Sleep(1 * time.Second)

    ports, err := explode(os.Args[3])
    if err != nil {
        log.Panicln(err)
    }

    for _, port := range ports {
        target := fmt.Sprintf("%s:%s", ip, port)
        fmt.Println("Trying", target)
        c, err := net.DialTimeout("tcp", target, 1000*time.Millisecond)
        if err != nil {
            continue
        }
        c.Close()
    }
    time.Sleep(2 * time.Second)

    for port, confidence := range results {
        if confidence >= 1 {
            fmt.Printf("Port %s open (confidence: %d)\n", port, confidence)
        }
    }
}

func explode(portString string) ([]string, error) {
    ret := make([]string, 0)

    ports := strings.Split(portString, ",")
    for _, port := range ports {
        port := strings.TrimSpace(port)
        ret = append(ret, port)
    }

    return ret, nil
}
```

Listing 8-6: Scanning and processing packets with SYN-flood protections \([https://github.com/blackhat-go/bhg/ch-8/syn-flood/main.go/](https://github.com/blackhat-go/bhg/ch-8/syn-flood/main.go/)\)

一般来说，代码中要维护数据包的计数，根据端口分组，以表示端口确实是打开的。使用过滤器只选择设置了适当 flag 的包。匹配数据包的数量越多，越能确定服务正在监听端口。

代码首先定义几个变量，以便在整个过程中使用。这些变量包括过滤器和map 类型的 `results`，使用它来跟踪对端口是否打开的确定级别。 目标端口作为key，并维护匹配包的计数作为map的值。

接下来定义函数 `capture()`，参数为接口名称和要测试的目标IP。函数本身以与前面示例相同的方式引导数据包抓取。然而，必须用不同的代码处理每个包。利用 `gopacket` 来提取包的网络和传输层。如果缺少这两层就忽略掉；这是因为下一步要检查包的源IP和端口，如果没有传输层或网络层的话就没有这些信息。然后再确定包的源机器IP地址是目标的。如果包的来源和IP地址不匹配就跳过处理。如果匹配就递增端口的确定等级。对后续的每个包重复此过程。匹配时就递增确定等级。

`main()`函数中使用一个协成调用 `capture()` 函数。使用协成的目的是确保抓包和处理逻辑物阻塞地并发执行。同时，`main()` 函数继续解析目标端口，一个接一个地遍历，然后调用 `net.DialTimeout` 尝试对每个进行TCP连接。协成在运行中，积极地监视这些连接尝试，寻找表示服务正在监听的包。

尝试连接每个端口后，处理所有结果时，只显示确定等级为1或更高的端口（这意味着至少有一个包与该端口的过滤器匹配）。代码中有几处调用 `time.Sleep()` 确保留有足够的时间来建立嗅探和处理包。

让我们看一下程序的示例运行，如清单8-7所示。

```text
$ go build -o syn-flood && sudo ./syn-flood enp0s5 10.1.100.100 80,443,8123,65530
Capturing packets
Trying 10.1.100.100:80
Trying 10.1.100.100:443 
Trying 10.1.100.100:8123 
Trying 10.1.100.100:65530 
Port 80 open (confidence: 1) 
Port 443 open (confidence: 1)
```

清单 8-7: 带有可信等级的端口扫描结果

测试成功地确定了端口80和443是开启的。同时也确定没有服务监听在端口8123 和 65530（注意，我们在示例中更改了IP地址以保护无辜者。）

可以用几种方式改进代码。作为学习练习，建议添加以下增强功能：

1. 从 `capture()` 函数中移除网络层和传输层逻辑和来源检查。相反，在BPF过滤器中添加额外参数来确保只抓取目标IP和端口的数据包。
2. 用并发替代端口扫描的顺序逻辑，类似于前几章演示的那样。这能提高效率。
3. 不是将代码限制为单个目标IP，而是允许用户提供IP或网络块的列表。

## 总结

我们已经完成了有关抓包的讨论，主要集中于被动嗅探活动。在下一章中，将重点介绍漏洞利用开发。

