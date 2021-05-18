# 第11章：加密的实现和攻击

没有密码学的安全通信是不完整的。使用加密能帮助保护信息和系统的完整性，机密性和真实性。作为工具开发人员，可能需要实现加密功能，可能用于 **SSL/TLS** 通信，相互身份验证，对称密钥加密或密码哈希。但是开发人员通常不安全地实现加密功能，这意味着有攻击意识的人可以利用这些弱点来破坏敏感的，有价值的数据，例如社保或信用卡号。

本章演示了Go中加密的各种实现，并讨论可以利用的常见弱点。虽然我们介绍不同的密码函数和代码块，但我们不探索密码算法或数学原理的细微差别。坦率地说，这远远超出了我们对密码学的兴趣（或知识）。如前所述，在未经所有者明确许可的情况下，请勿在本章中对资源或资产做任何事情。我们研究这些是出于学习目的，而不是为了协助进行非法活动。

## 复习密码学的基本概念

在探讨Go语言中的加密之前，让我们复习一些基本的加密概念。长话短说吧。

首先，加密（出于维护机密性的目的）只是加密的任务之一。_Encryption_，通常来说是双向的，可以对数据加密，然后解密恢复初始的输入。加密数据的过程使得它在被解密之前毫无意义。

加密和解密都涉及到将数据和附带的密钥传递到加密函数。该函数输出加密的数据（称为密文）或原始的可读数据（称为明文）。有多种算法实现。`Symmetric` 算法在加解密时使用相同的秘钥，而 `asymmetric` 使用不同的秘钥。您可能会使用加密来保护传输中的数据或存储敏感信息（例如信用卡号），以便日后解密，这可能是为了方便将来的购买或欺骗监控。

另一方面，`hashing`是用于对数据进行数学扰乱的单向过程。可以将敏感信息传递到哈希函数来生成固定长度的输出。当使用强大的算法（例如SHA-2系列算法）时，不同输入产生相同输出的可能性非常低。即，发生碰撞的可能性低。由于哈希是不可逆的，因此通常用作在数据库中存储明文密码或执行完整性校验以确定数据是否被更改过。如果需要模糊或随机化两个相同输入的输出，可以使用 `salt`，这是一个随机值，用于在哈希过程中区分两个相同的输入。`salt` 通常用于密码存储，因为允许同时使用相同密码的多个用户仍然生成不同的哈希值。

密码学还提供了对消息进行身份验证的方法。`message authentication code (MAC)` 是由一个特殊的单向加密函数产生的输出。这个函数使用数据本身、一个密钥和一个初始化向量，并生成一个不太可能发生冲突的输出。消息的发送者执行生成MAC的功能，然后将MAC作为消息的一部分。接收方在本地计算MAC并将其与接收到的MAC进行比较。匹配成功表明发送方拥有正确的密钥（即发送方是可信的），并且消息没有被更改（即保持了完整性）。

现在到这，应该对密码学有足够的了解了，可以理解本章的内容。必要时，我们将讨论与给定主题相关的更多细节。先从Go的标准加密库开始吧。

## 搞懂标准加密库

在Go中实现加密的妙处在于，使用的大多数加密功能都来自于标准库。其他语言通常依赖于OpenSSL或其他第三方库，而Go的加密功能是官方库的一部分。这使得加密的实现相对简单，因为不必安装会污染开发环境的笨重的依赖项。有两个独立的库。

标准库的`crypto`包中有各种常见的加密和算法相关的子包。例如，可以使用 `aes`，`des`和`rc4`子包来实现对称密钥算法。用于非对称加密的有`dsa`和`rsa`子包；以及用于哈希的`md5`，`sha1`，`sha256`和`sha512`子包。这不是全部；另外还有用于其他加密的子包。

除了标准的 `crypto` 包，Go还有一个官方的扩展包，包含各种加密功能：`golang.org/x/crypto`。功能包括其他哈希算法，加密算法和通用功能。例如，该包中有用于_bcrypt hashing_的`bcrypt`子包（一种用于哈希密码和敏感数据的更好，更安全的替代方法），用于生成合法证书的`acme/autocert` 以及方便SSH协议通信的SSH子包。

内置`crypto`包和补充的`golang.org/x/crypto`包之间唯一真正的区别是，`crypto`包遵循更严格的兼容性要求。另外，要使用`golang .org/x/crypto`中的子包，则首先需要输入以下内容来安装该软件包：

```bash
$ go get -u golang.org/x/crypto/bcrypt
```

Go官方的 `crypto`包中所有功能和子包的完整列表，请参阅 `https://golang.org/pkg/crypto/` 和 `https://godoc.org/golang.org/x/crypto/`。

下一节将深入探讨各种加密实现。将会展示如何使用Go的加密功能来做一些邪恶的事情，例如破解密码哈希，使用静态密钥解密敏感数据以及暴力破解弱加密密码。还将使用该功能来创建使用TLS来保护传输中的通信，检查数据的完整性和真实性以及执行相互身份验证的工具。

## 探索哈希

如前所述，哈希是一种单向函数，用于根据变长输入生成固定长度、概率唯一的输出。不能反向哈希值来恢复原始输入数据。哈希通常用于存储原始明文数据，以后不再处理或验证数据的完整性。例如，糟糕的做法是存储明文密码；相反，应该存储哈希（最好是加点佐料，即随机值，以确保重复值之间的随机性）。

通过两个例子来演示Go中的哈希。第一个例子是尝试使用离线字典来破解给定的MD5或SHA-512哈希。第二个例子是演示 `bcrypt` 的实现。如前所述，`bcrypt` 是一种用于哈希敏感数据（例如密码）的更安全的算法。该算法还有降低其速度的功能，这使得破解密码更加困难。

### 破解MD5或SHA-512哈希

清单11-1是哈希破解代码。（`/`根目录中的所有代码都在github 仓库 `https://github.com/blackhat-go/bhg/` 中。）由于哈希不是可逆的，因此代码会尝试通过生成常见单词（从单词列表中提取）的哈希，然后将生成的哈希值与当前的哈希进行比较，来猜测哈希的明文。如果两个哈希值匹配，则可能就猜到了明文值。

```go
var md5hash = "77f62e3524cd583d698d51fa24fdff4f"
var sha256hash = "95a5e1547df73abdd4781b6c9e55f3377c15d08884b11738c2727dbd887d4ced"
func main() {
    f, err := os.Open("wordlist.txt")
    if err != nil {
        log.Fatalln(err) 
    }
    defer f.Close()
    scanner := bufio.NewScanner(f) 
    for scanner.Scan() {
        password := scanner.Text()
        hash := fmt.Sprintf("%x", md5.Sum([]byte(password))x) 
        if hash == md5hash {
            fmt.Printf("[+] Password found (MD5): %s\n", password) 
        }
        hash = fmt.Sprintf("%x", sha256.Sum256([]byte(password))) 
        if hash == sha256hash {
            fmt.Printf("[+] Password found (SHA-256): %s\n", password) 
        }
    }
    if err := scanner.Err(); err != nil { 
        log.Fatalln(err)
    } 
}
```

清单 11-1: 破解 MD5 和 SHA-256 哈希 \(/ch-11/hashes/main.go\)

首先定义两个保存目标哈希值的变量。一个是MD5 哈希，另一个是SHA-256 哈希。想象一下，您是在后漏洞阶段获取了这两个哈希，并试图通过运行散列算法生成它们的输入（明文密码）。通常可以通过检查哈希值的长度来确定算法。找到与目标匹配的哈希后，就知道是正确的输入了。

使用之前创建的字典文件作为输入列表。另外，谷歌一下可以帮助找到常用密码的字典文件。要检查MD5哈希，请打开字典文件并通过在文件描述符上创建 `bufio.Scanner` 逐行读取。每行是一个要检查的密码。将当前密码传递给 `md5.Sum(input [] byte)` 函数。此函数生成原生的MD5哈希值，因此可以使用 `fmt.Sprintf()` 函数和格式字符串 `％x`将其转换为十六进制字符串。毕竟，`md5hash` 变量由目标哈希的十六进制格式的字符串组成。转换该值可确保随后可以比较目标哈希值和计算得出的哈希值。如果这些哈希匹配，则程序会向stdout输出一条成功消息。

执行类似的过程来计算和比较SHA-256哈希。该实现与MD5代码非常相似。唯一真正的区别是，`sha256`包中有计算各种SHA散列长度的附加函数。与其调用 `sha256. sum()` （一个不存在的函数），不如调用 `sha256.Sum256(input []byte)` 强制使用SHA-256算法计算哈希值。就像在MD5示例中所做的一样，将原始字节转换为十六进制字符串，并比较SHA-256哈希值以查看是否有匹配项。

#### 实现 **bcrypt**

下一个示例展示了如何使用 `bcrypt` 加密和验证密码。与SHA和MD5不同，`bcrypt` 是为密码哈希设计的，与SHA或MD5系列相比，它成为程序员的更好选择。默认情况下， `bcrypt` 包含一个”佐料“，以及一个运行该算法更加耗费资源的成本因素。该成本因素控制着内部加密函数的迭代次数，从而增加了破解密码哈希所需的时间和精力。尽管仍然可以使用字典或暴力来破解密码，但是成本（时间）会显着增加，从而阻止了对时间敏感的漏洞后的破解。随着时间的推移，还可能会增加成本，以应对计算能力的提高。这使其可以适应将来的破解攻击。

清单11-2创建了一个bcrypt哈希，然后验证明文密码是否与给定的bcrypt哈希匹配。

```go
import ( 
    "log"
     "os"
    "golang.org/x/crypto/bcrypt"
)
var storedHash = "$2a$10$Zs3ZwsjV/nF.KuvSUE.5WuwtDrK6UVXcBpQrH84V8q3Opg1yNdWLu"

func main() {
    var password string
    if len(os.Args) != 2 {
        log.Fatalln("Usage: bcrypt password") 
    }
    password = os.Args[1]
    hash, err := bcrypt.GenerateFromPassword( 
        []byte(password),
        bcrypt.DefaultCost, 
    )
    if err != nil { 
        log.Fatalln(err)
    }
    log.Printf("hash = %s\n", hash)
    err = bcrypt.CompareHashAndPassword([]byte(storedHash), []byte(password)) 
    if err != nil {
        log.Println("[!] Authentication failed")
        return 
    }
    log.Println("[+] Authentication successful")
}
```

清单 11-2: 比较 bcrypt 哈希 \(/ch-11/bcrypt/main.go\)

对于本书中的大多数代码示例，都省略了包的导入。在此示例中包含了它们，以明确表示正在使用补充的Go包`golang.org/x/crypto/bcrypt`，因为Go的标准包中没有 `bcrypt` 相关功能。然后，初始化变量 `storedHash`，该变量包含一个预先计算的，`bcrypt` 编码的哈希。这是为了演示设计的例子;的目的，硬编码一个值，而不是将示例代码连接到数据库来获取值。例如，该变量可能表示数据库中某一行值，该行存储了前端Web程序的用户身份验证信息。

接下来，根据明文密码值生成一个经过bcrypt编码的哈希。main函数读取密码值作为命令行参数，然后继续调用两个单独的bcrypt函数。第一个函数 `bcrypt.GenerateFromPassword()`接受两个参数：一个字节切片\(表示明文密码\)和一个成本。在此示例中，使用包里的默认值 `bcrypt.DefaultCost` 常量，在撰写本文时，该默认值为10。该函数返回编码的哈希值和产生的任何错误。

第二个调用的bcrypt函数是`bcrypt.CompareHashAndPassword()` ， 用来进行哈希比较。它接受bcrypt编码的哈希和明文密码作为字节切片。该函数解析编码的哈希，以确定成本和随机值。然后，将这些值与明文密码值一起生成bcrypt哈希。如果生成的哈希值与从已编码的`storedHash` 值提取的哈希值匹配，则提供的密码与用于创建 `storedHash`的密码匹配。这是用于对SHA和MD5进行密码破解的方法，即通过哈希函数运行给定密码并将结果与存储的哈希进行比较。这里，不像SHA和MD5那样明确地比较生成的哈希，而是检查`bcrypt.CompareHashAndPassword()` 是否返回错误。如果有错误，则说明计算出的哈希值是正确的，因此用于计算它们的密码不匹配。

以下是两个示例程序运行。第一个是不正确密码的输出：

```bash
$ go run main.go someWrongPassword
2020/08/25 08:44:01 hash = $2a$10$YSSanGl8ye/NC7GDyLBLUO5gE/ng51l9TnaB1zTChWq5g9i09v0AC 
2020/08/25 08:44:01 [!] Authentication failed
```

第二个是正确密码的输出：

```bash
$ go run main.go someC0mpl3xP@ssw0rd
2020/08/25 08:39:29 hash = $2a$10$XfeUk.wKeEePNAfjQ1juXe8RaM/9EC1XZmqaJ8MoJB29hZRyuNxz. 
2020/08/25 08:39:29 [+] Authentication successful
```

细心的可能会注意到，身份验证成功的显示的哈希与 `storedHash` 变量硬编码的值不匹配。回想下，代码调用了两个独立的函数。`GenerateFromPassword()` 函数使用随机值生成编码的哈希。使用不同的随机值，相同的密码将产生不同结果的哈希。因此,不同。`CompareHashAndPassword()` 函数通过使用与存储的哈希相同的随机值和成本来执行哈希算法，因此生成的哈希与`storedHash`变量中的哈希相同。

## 验证消息

现在将重点转向消息验证。交换消息时，需要验证数据的完整性和远程服务的真实性，以确保数据是真实且未被篡改的。消息在传输过程中是否被未经授权的来源更改？消息是由授权方发送的，还是由别的实体伪造的?

可以使用Go的`crypto/hmac` 包解决这些问题，该包实现了_Keyed-Hash Message Authentication Code_ \(HMAC\)标准。HMAC是一种加密算法，可让检查消息是否被篡改并验证源身份。它使用哈希函数并使用共享的密钥，只有被授权产生有效消息或数据的各方才应拥有该密钥。没有共享此秘钥的攻击者无法伪造有效的HMAC值。在某些编程语言中实现HMAC可能会有些棘手。例如，某些语言会强制逐字节手动比较接收到的哈希值和计算得出的哈希值。如果开发人员过早地逐字节比较其结果，则开发人员可能会在此过程中无意中引入时序差异。攻击者可以通过测量消息处理时间来推断出预期的HMAC。此外，开发人员有时会认为HMAC（消耗一条消息和密钥）与消息之前的密钥的哈希值相同。但是，HMAC的内部不同于纯哈希功能。通过不明确地使用HMAC，开发人员会将应用程序暴露于长度扩展攻击中，在这种攻击中，攻击者会伪造消息和有效的MAC。

对我们Gophers来说幸运的是，crypto/hmac包使安全的方式实现HMAC功能变得相当容易。来看一个实现。注意，以下程序比典型的用例简单得多，后者可能涉及某种类型的网络通信和消息传递。在大多数情况下，可以根据HTTP请求参数或通过网络传输的其他消息来计算HMAC。在清单11-3所示的示例中，省略了客户端与服务器之间的通信，只看HMAC功能。

```go
var key = []byte("some random key")
func checkMAC(message, recvMAC []byte) bool {
    mac := hmac.New(sha256.New, key)
    mac.Write(message)
    calcMAC := mac.Sum(nil)
    return hmac.Equal(calcMAC, recvMAC)
}
func main() {
    // In real implementations, we’d read the message and HMAC value from network source
    message := []byte("The red eagle flies at 10:00")     
    mac, _ := hex.DecodeString("69d2c7b6fbbfcaeb72a3172f4662601d1f16acfb46339639ac8c10c8da64631d")
    if checkMAC(message, mac) {
        fmt.Println("EQUAL") 
    } else {
        fmt.Println("NOT EQUAL") 
    }
}
```

清单 11-3: 使用HMAC进行消息身份验证 \(/ch-11/hmac/main.go\)

程序首先定义要用于HMAC加密的密钥。也就在这里使用硬编码，但在实际的实现中，此密钥将受到充分保护并且是随机的。该秘钥也将在端点之间共享，意味着消息发送方和接收方使用此相同的秘钥。由于未实现完整的客户端-服务器功能，所以使用这个变量，就好像它已被充分共享一样。

接下来，定义 `checkMAC()` 函数，以消息和接收到的HMAC作为参数。消息接收者将调用此函数以检查他们接收到的MAC值是否与他们在本地计算的值匹配。首先，调用 `hmac.New()` ，实参为 `sha256.New` 和 `key`，返回 `hash.Hash` 实例。在这个例子中，`hmac.New()` 函数通过使用SHA-256算法和密钥来初始化HMAC，并将结果分配给名为mac的变量。然后，可以使用此变量来计算HMAC哈希值，就像在前面的哈希示例中所做的那样。在这里，分别调用 `mac.Write(message)` 和 `mac.Sum(nil)` 。将本地计算的HMAC结果存储在名为`calcMAC` 的变量中。

下一步是比较本地计算的HMAC值是否等于收到的HMAC值。要以一种安全的方式做到这一点，可以调用 `hmac.Equal(calcMAC, recvMAC)` 。许多开发人员倾向于通过调用 `bytes.Compare(calcMAC, recvMAC)` 来比较字节片。问题是，`bytes.Compare()` 执行字典比较，遍历并比较给定切片的每个元素，直到找到差异或到达切片的末尾。比较所需的时间将根据`bytes.Compare()`在第一个元素，最后一个元素或两者之间的某个地方是否有所不同而不同。攻击者可能会及时测量这种变化，以确定预期的HMAC值并伪造合法处理的请求。`hmac.Equal()` 函数通过几乎相同的时间的方式比较切片来解决此问题。函数在何处发现差异都无所谓，因为处理时间变化不大，不会产生明显或可察觉的形式。

`main()` 函数模拟从客户端接收消息的过程。如果确实收到了一条消息，则必须从传输中读取并解析HMAC和消息。由于这只是一个模拟，因此可以对接收到的消息和接收到的HMAC进行硬编码，然后对HMAC十六进制字符串进行解码，以便将其转换成 `[]byte`。可以使用if语句来调用 `checkMAC()`函数，并将接收到的消息和HMAC传递给该函数。如前所述，\`checkMAC\(\) 函数通过使用接收到的消息和共享密钥来计算HMAC，并返回布尔值来确定接收到的HMAC和计算出的HMAC是否匹配。

尽管HMAC确实提供了真实性和完整性保证，但它不能确保私密性。无法确定是否未经授权的源看不到该消息。下一部分将通过探索和实现各种类型的加密来解决此问题。

## 加密数据

加密可能是最著名的加密概念。毕竟，隐私和数据保护由于备受瞩目的数据泄露而获得了大量新闻报道，这通常是由于未以加密格式存储了用户密码和其他敏感数据。即使没有媒体的注意，加密也应该引起黑帽和开发人员的关注。毕竟，了解基本过程和实现可能是有利可图的数据泄露与令人沮丧的攻击终止链之间的区别。下一节介绍了各种加密形式，包括有用的应用程序和每种用例。

### 对称密钥加密

以最直接的形式——对称密钥加密，进入加密之旅。这种模式是加密和解密都使用相同的密钥。Go使对称密码学变得非常简单，因为默认包或扩展包支持大多数常用算法。

为了简洁，在一个示例中讲解对称密钥加密。假设要攻击一个组织。已经执行了必要的权限升级，横向移动和网络侦察，才能访问电子商务Web服务器和后端数据库。该数据库中有金融交易；但是，显然这些交易中使用的信用卡号已加密。检查Web服务器上的应用程序源代码，并确定组织正在使用高级加密标准（AES）的加密算法。AES支持多种操作方式，每种方式都有不同的注意事项和实现细节。方式间是不可互换的。用于解密的方式必须与用于加密的方式相同。

在这种情况下，假设确定应用程序正在以密码块链接（CBC）方式使用AES。因此，让我们编写一个解密这些信用卡的功能（清单11-4）。假设对称密钥已在应用程序中进行了硬编码或在配置文件中进行了静态设置。在阅读本示例时，请记住，需要针对其他算法或密码调整此实现，但这是一个很好的起点。

```go
func unpad(buf []byte) []byte {
    // Assume valid length and padding. Should add checks padding := int(buf[len(buf)-1])
    return buf[:len(buf)-padding]
}

func decrypt(ciphertext, key []byte) ([]byte, error) {
    var (
        plaintext []byte
        iv []byte
        block cipher.Block 
        mode cipher.BlockMode
        err error 
    )

    if len(ciphertext) < aes.BlockSize {
        return nil, errors.New("Invalid ciphertext length: too short")
    }

    if len(ciphertext)%aes.BlockSize != 0 {
        return nil, errors.New("Invalid ciphertext length: not a multiple of blocksize")
    }

    iv = ciphertext[:aes.BlockSize]
    ciphertext = ciphertext[aes.BlockSize:]

    if block, err = aes.NewCipher(key); err != nil {
        return nil, err
    }
    mode = cipher.NewCBCDecrypter(block, iv)
    plaintext = make([]byte, len(ciphertext)) 
    mode.CryptBlocks(plaintext, ciphertext)
    plaintext = unpad(plaintext)}

    return plaintext, nil
}
```

清单 11-4: AES填充和解密 \(/ch-11/aes/main.go\)

代码中定义了两个函数：`unpad()` 和 `decrypt()` 。`unpad()` 函数是一个整合在一起的通用函数，用于处理解密后填充数据的删除。这是必要的步骤，但超出了此讨论的范围。有关更多信息请查阅 Public Key Cryptography Standards \(PKCS\) \#7 。这是AES的一个相关主题，因为它用于确保我们的数据具有正确的块对齐方式。对于此示例，只知道以后需要使用该功能来清理数据。该函数本身假设要在实际场景中已明确验证了一些事实。具体来说，需要确认填充字节的值是否有效，切片偏移量是否有效以及结果的长度是否合适。

最有趣的逻辑存在于 `delete()` 函数中，该函数使用两个字节切片：需要解密的密文和用于执行此操作的对称密钥。该函数先执行一些验证，至少确认密文与块大小长度相等。这是必要的步骤，因为CBC模式加密使用初始向量（IV）来实现随机性。该IV就像密码哈希的随机值一样，不需要保密。IV与单个AES块的长度相同，在加密过程中会加在密文上。如果密文长度小于预期的块大小，则说明密文有问题或缺少IV。还检查密文长度是否为AES块大小的倍数。否则，解密将失败，因为CBC模式期望密文长度为块大小的倍数。

完成验证检查后，可以继续解密密文。如前所述，IV是加在密文之前的，因此要做的第一件事是从密文中提取IV。可以使用 `aes.BlockSize` 常量来检索IV，然后通过 `ciphertext = [aes.BlockSize:]` 将密文变量重新定义为密文的其余部分。现在，已将加密数据与IV分开了。

接下来，调用 `aes.NewCipher()` ，并向其传递对称密钥。这将初始化AES块模式密码，并将其分配给名为 `block` 的变量。然后，通过调用 `cipher.NewCBCDecryptor(block,iv) 来明确AES密码以CBC模式运行。将结果赋值给名为`mode`的变量。（crypto/cipher包中有用于其他AES模式的初始化功能，但此处仅使用CBC解密。）然后，调用`mode.CryptBlocks\(plaintext，ciphertext\)`，用来解密密文的内容，并将结果存储在纯文本字节切片中。最后，通过调用`unpad\(\)\` 通用函数来删除PKCS＃7填充。返回结果。如果一切正常，这应该是信用卡号的纯文本值。

运行该示例，生成预期的结果：

```bash
$ go run main.go
key = aca2d6b47cb5c04beafc3e483b296b20d07c32db16029a52808fde98786646c8
ciphertext = 7ff4a8272d6b60f1e7cfc5d8f5bcd047395e31e5fc83d062716082010f637c8f21150eabace62 
--snip--
plaintext = 4321123456789090
```

注意，没有在此示例代码中定义 `main()` 函数。为什么没有呢？嗯，在陌生环境中解密数据会产生各种潜在的细微差别和变化。例如，密文和密钥值是编码的还是原始二进制的？如果已编码，它们是十六进制字符串还是Base64？数据是否在本地可访问，还是需要从数据源中提取数据或与硬件安全模块进行交互？键是，解密很少是复制粘贴的工作，通常需要对算法，模式，数据库交互和数据编码有一定程度的了解。因此，我们选择引导您找到答案，期望您在适当的时候弄清楚。

只需了解一点对称密钥加密，就可以使渗透测试更加成功。例如，在我们窃取客户端源代码存储库的经验中，我们发现人们经常在CBC或Electronic Codebook（ECB）模式下使用AES加密算法。ECB模式有一些固有的弱点，如果使用不正确，CBC也不会更好。加密可能很难理解，因此开发人员经常认为所有加密密码和模式都同样有效，并且不了解其细微之处。尽管我们不认为自己是密码学家，但我们了解的知识足以在Go中安全地使用加密，并可以利用其他人的缺陷实现。

尽管对称密钥加密比非对称加密更快，但是它遭受内在的密钥管理挑战。毕竟，要使用它，必须将相同的密钥分发给对数据执行加密或解密的系统或应用程序。必须经常遵循严格的流程和审核机制，才能安全地分发密钥。此外，例如，仅依赖对称密钥加密可以防止任意客户机与其他节点建立加密通信。没有一种很好的方法来协商密钥，也没有许多常见算法和模式的身份验证或完整性保证。这意味着获得密钥的任何人，无论是经过授权的还是恶意的，都可以继续使用它。

这就是非对称密码学可以使用的地方。

### 非对称加密

与对称密钥加密相关的许多问题都可以通过非对称（或公钥）加密技术解决，该加密技术使用两个独立但在数学上相关的密钥。一个公开，而另一个私密。用私钥加密的数据只能用公钥解开，而用公钥加密的数据只能用私钥解开。如果私钥保护得当，且是私密的，那么使用公钥加密的数据仍然是私密的，因为需要严密保护的私钥来解密。不仅如此，还可以使用私钥对用户进行身份验证。用户可以使用私钥对消息签名，例如，公众可以使用公钥解密。

因此，有人可能会问：“有什么收获呢？如果公钥加密提供了所有这些保证，那么为什么我们还要使用对称密钥加密呢？”，这是个好问题！公钥加密的问题在于它的速度。它比对称加密要慢得多。为了获得两全其美的效果（并避免最坏的情况），通常混合使用：初始通信时使用非对称加密，建立加密通道以创建和交换对称密钥（通常称为 _会话秘钥_）。由于会话密钥非常小，因此使用公钥加密进行此过程几乎不需要任何开销。然后，客户端和服务器都有会话密钥的副本，可使后面的通信更快。

来看下公钥加密的几个常见用例。具体来说，加密，签名验证和相互认证。

#### 加密和签名验证

对于第一个示例，对消息使用公钥进行加密和解密。还将创建逻辑以对消息签名并验证该签名。为简单起见，将所有逻辑都放在 `main()`函数中。这旨在展示核心功能和逻辑，以便可以实现。在现实场景中，因为可能有两个远程节点相互通信，此过程可能要复杂一些。这些节点必须交换公钥。幸运的是，此交换过程不需要与交换对称密钥相同的安全保证。回想下，用公钥加密的任何数据只能由相关的私钥解密。因此，即使中间有人攻击来拦截公钥交换和之后的通信，也无法解密使用同一公钥加密的任何数据。只有私钥可以解密。

来看一下清单11-5所示的实现。在查看示例时，我们将详细解释逻辑和加密功能。

```go
func main() {
    var (
        err                                             error
        privateKey                                         *rsa.PrivateKey
        publicKey                                        *rsa.PublicKey
        message, plaintext, ciphertext, signature, label []byte
    )
    if privateKey, err = rsa.GenerateKey(rand.Reader, 2048); err != nil { 
        log.Fatalln(err)
    }
    publicKey = &privateKey.PublicKey

    label = []byte("")
    message = []byte("Some super secret message, maybe a session key even")
    ciphertext, err = rsa.EncryptOAEP(sha256.New(), rand.Reader, publicKey, message, label) 
    if err != nil {
        log.Fatalln(err) 
    }
    fmt.Printf("Ciphertext: %x\n", ciphertext)

    plaintext, err = rsa.DecryptOAEP(sha256.New(), rand.Reader, privateKey, ciphertext, label) 
    if err != nil {
        log.Fatalln(err) 
    }
    fmt.Printf("Plaintext: %s\n", plaintext)

    h := sha256.New()
    h.Write(message)
    signature, err = rsa.SignPSS(rand.Reader, privateKey, crypto.SHA256, h.Sum(nil), nil) 
    if err != nil {
        log.Fatalln(err) 
    }
    fmt.Printf("Signature: %x\n", signature)

    err = rsa.VerifyPSS(publicKey, crypto.SHA256, h.Sum(nil), signature, nil)
    if err != nil {
        log.Fatalln(err) 
    }
    fmt.Println("Signature verified")
}
```

清单 11-5: 非对称或公钥加密 \(/ch-11/public-key/main.go/\)

该程序演示了两个独立但相关的公钥加密功能：加密/解密和消息签名。首先，通过调用 `rsa.GenerateKey()` 函数来生成一个公钥/私钥对。使用随机reader和key的长度作为函数的参数。假设随机reader和key的长度足以生成密钥，则结果为 `*rsa.PrivateKey` 实例，该实例包含其值为公钥的字段。现在有了一个有效的密钥对。为了方便起见，将公钥分配给它自己的变量。

程序在每次运行时都会生成此密钥对。在大多数情况下，例如SSH通信，将一次生成密钥对保存到磁盘。私钥将保护好，公钥将分发到端点。在这里跳过密钥的分配，保护和管理，仅关注加密功能。

创建密钥后，就可以开始使用它们进行加密了。调用函数 `rsa.EncryptOAEP() ，该函数参数为哈希函数，用于填充和随机的reader，公钥，要加密的消息以及可选标签。函数返回错误（如果输入导致算法失败）和密文。然后，将相同的哈希函数，reader，私钥，密文和标签传递到函数`rsa.DecryptOAEP\(\)\` 中。函数使用私钥解密密文，并返回明文结果。

注意，现在正在使用公钥加密消息。这样可以确保只有私钥持有者才能解密数据。接下来，通过调用 `rsa.SignPSS()` 创建数字签名。再次传递一个随机reader，私钥，使用的哈希函数，消息的哈希值以及代表其他选项的nil值。该函数返回错误和签名值。就像人类的DNA或指纹一样，此签名可以唯一地识别签名者的身份（即私钥）。持有公钥的任何人都可以验证签名，不仅可以确定签名的真实性，还可以验证消息的完整性。将公钥，哈希函数，哈希值，签名和其他选项传递给 `rsa.VerifyPSS()` 来验证签名。注意，在这里是将公钥而不是私钥传递给该函数。希望验证签名的端点将无法访问私钥，如果输入错误的密钥值，验证也不会成功。签名有效时，`rsa .VerifyPSS()` 函数返回nil，而无效时返回错误。

下面是该示例的运行结果。其表现符合预期，使用公钥加密消息，使用私钥解密消息，并验证签名：

```bash
$ go run main.go
Ciphertext: a9da77a0610bc2e5329bc324361b480ba042e09ef58e4d8eb106c8fc0b5
--snip--
Plaintext: Some super secret message, maybe a session key even
Signature: 68941bf95bbc12edc12be369f3fd0463497a1220d9a6ab741cf9223c6793
--snip--
Signature verified
```

接下来，看一下公钥加密的另一种应用：相互认证。

#### 相互认证

相互认证是客户端和服务器相互认证的过程。他们使用公钥加密来完成。客户端和服务器都将生成公钥/私钥对，交换公钥，并使用公钥来验证另一个端点的真实性和身份。要实现这一壮举，客户端和服务器都必须做额外的工作来设置授权，明确定义他们打算用来验证彼此的公钥。此过程的不利方面是管理开销，必须为每个节点创建唯一的密钥对，并确保服务器和客户端节点具有适当的数据以正确进行。

首先，将移除创建密钥对的管理任务。将公钥存储为自签名的，PEM编码的证书。使用openssl 工具创建这些文件。在服务器上，通过输入以下内容来创建服务器的私钥和证书：

```bash
$ openssl req -nodes -x509 -newkey rsa:4096 -keyout serverKey.pem -out serverCrt.pem -days 365
```

openssl命令将提示输入各种输入，在此示例中，可以用任意值。该命令创建两个文件：serverKey.pem和serverCrt.pem。文件serverKey.pem含有私钥，必须保护好。 serverCrt.pem文件包含服务器的公钥，将其分发给每个连接的客户端。

对于每个连接的客户端，运行与上面类似的命令：

```bash
$ openssl req -nodes -x509 -newkey rsa:4096 -keyout clientKey.pem -out clientCrt.pem -days 365
```

命令也生成两个文件：clientKey.pem和clientCrt.pem。与服务器输出一样，保护好客户端的私钥。 clientCrt.pem证书文件将发送到服务器并由程序加载。这就可以配置客户端并将其标识为授权端点。必须为其他每个客户端创建，传输和配置证书，以便服务器可以识别并明确授权它们。

在清单11-6中，创建了一个HTTPS服务器，该服务器要求客户端提供合法的授权证书。

```go
func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Printf("Hello: %s\n", r.TLS.PeerCertificates[0].Subject.CommonName)
    fmt.Fprint(w, "Authentication successful")
}
func main() {
    var (
        err            error
        clientCert    []byte
        pool        *x509.CertPool
        tlsConf        *tls.Config
        server        *http.Server
    )

    http.HandleFunc("/hello", helloHandler)

    if clientCert, err = ioutil.ReadFile("../client/clientCrt.pem"); err != nil { 
        log.Fatalln(err)
    }
    pool = x509.NewCertPool() 
    pool.AppendCertsFromPEM(clientCert) 

    tlsConf = &tls.Config{
        ClientCAs: pool,
        ClientAuth: tls.RequireAndVerifyClientCert,
    }
    tlsConf.BuildNameToCertificate() z

    server = &http.Server{ 
        Addr: ":9443", 
        TLSConfig: tlsConf,
    }
    log.Fatalln(server.ListenAndServeTLS("serverCrt.pem", "serverKey.pem")) 
}
```

清单 11-6: 创建相互认证的服务器 \(/ch-11/mutual-auth/cmd/server/main.go\)

`main()` 函数外定义了 `helloHandler()` 函数。正如在第3章和第4章中讨论的那样，处理程序函数接受 `http.ResponseWriter` 实例和`http.Request` 实例。这个函数非常无聊。记录接收到的客户端证书的通用名。通过检查 `http.Request的TLS` 字段并深入查看证书`PeerCertificates` 数据，可以访问通用名。处理函数还会向客户端发送一条消息，表明身份验证成功。

但是，如何定义授权哪些客户端，以及如何对它们进行身份验证呢？这个过程相当轻松。首先要从客户端先前创建的PEM文件中读取客户端的证书。由于可能有多个授权的客户端证书，因此可以创建一个证书池和调用池。`AppendCertsFromPEM(clientCert)` 可以将客户端证书添加到池中。对要验证的每个其他客户端执行此步骤。

接下来，创建TLS配置。将`ClientCAs`字段明确地设置为池，并将 `ClientAuth` 配置为 `tls.RequireAndVerifyClientCert` 。该配置定义了授权客户端池，并要求客户端在允许继续之前先正确地标识自己。调用 `tlsConf.BuildNameToCertificate()` ，以便客户端的通用名和主题备用名（为其生成证书的域名）将正确映射给定的证书。并通过调用 `server.ListenAndServeTLS()` 启动服务器。将先前创建的服务器证书和私钥文件传递给它。注意的是，服务器中的代码并没有使用客户端的私钥文件。就像之前说过的那样，私钥仍然是私密的；服务器仅使用客户端的公钥来识别和授权客户端。这是公钥加密的特色。

使用 `curl` 来验证服务器。如果生成并提供了伪造的，未经授权的客户证书和密钥，则会收到一条详细的消息：

```bash
$ curl -ik -X GET --cert badCrt.pem --key badKey.pem \ 
    https://server.blackhat-go.local:9443/hello
curl: (35) gnutls_handshake() failed: Certificate is bad
```

同样在服务器也收到一条更详细的消息，如下所示：

```bash
http: TLS handshake error from 127.0.0.1:61682: remote error: tls: unknown certificate authority
```

另一方面，如果使用的是有效的证书和与服务器池中配置的证书相匹配的密钥，则成功验证时会有点自豪：

```bash
$ curl -ik -X GET --cert clientCrt.pem --key clientKey.pem \ 
    https://server.blackhat-go.local:9443/hello
HTTP/1.1 200 OK
Date: Fri, 09 Oct 2020 16:55:52 GMT 
Content-Length: 25
Content-Type: text/plain; charset=utf-8

Authentication successful
```

此消息表明服务器正常。

现在来看下客户端（清单11-7）。可以在与服务器相同或不同的系统上运行客户端。如果在其他系统上，则需要将 `clientCrt.pem` 发送到服务器，将 `serverCrt.pem` 发送到客户端。

```go
func main() {
    var (
        err                 error
        cert                 tls.Certificate
        serverCert, body    []byte
        pool                 *x509.CertPool 
        tlsConf             *tls.Config
        transport             *http.Transport 
        client                 *http.Client 
        resp                 *http.Response
    )
    if cert, err = tls.LoadX509KeyPair("clientCrt.pem", "clientKey.pem"); err != nil {
        log.Fatalln(err)
    }

    if serverCert, err = ioutil.ReadFile("../server/serverCrt.pem"); err != nil {
        log.Fatalln(err)
    }

    pool = x509.NewCertPool()
    pool.AppendCertsFromPEM(serverCert) 

    tlsConf = &tls.Config{
        Certificates: []tls.Certificate{cert}, 
        RootCAs: pool,
    } 
    tlsConf.BuildNameToCertificate()

    transport = &http.Transport{
        TLSClientConfig: tlsConf,
    }
    client = &http.Client{{
        Transport: transport,
    }

    if resp, err = client.Get("https://server.blackhat-go.local:9443/hello"); err != nil {
        log.Fatalln(err)
    }
    if body, err = ioutil.ReadAll(resp.Body); err != nil {
        log.Fatalln(err) 
    }
    defer resp.Body.Close()
    fmt.Printf("Success: %s\n", body) 
}
```

清单 11-7: 相互认证的客户端 \(/ch-11/mutual-auth/cmd/client/main.go\)

许多证书的准备和配置工作与服务器代码相似：创建证书池并准备主题名和通用名。由于不会使用客户端证书和密钥作为服务器，因此调用 `tls.LoadX509KeyPair(“ clientCrt.pem”, “ clientKey.pem”)` 来加载它们以供后面使用。还读取了服务器证书，并将其添加到希望允许的证书池中。然后，使用池和客户端证书来构建TLS配置，并调用 `tlsConf.BuildNameToCertificate()` 将域名绑定到其相应的证书。

由于创建的是HTTP客户端，因此必须定义一种传输方式，并关联TLS配置。然后，使用传输实例来创建http.Client结构。如在第3章和第4章中讨论的那样，使用此客户端通过 `client.Get(“ https://server.blackhat-go.local:9443 / hello”)` 进行HTTP GET请求。

此时，幕后正在执行相互身份验证——客户端和服务器相互进行身份验证。如果身份验证失败，程序将返回错误并退出。否则，读取HTTP响应体并将其显示到stdout。运行客户端代码会产生预期的结果，具体是没有抛出任何错误并且身份验证成功：

```bash
$ go run main.go
Success: Authentication successful
```

服务器输出如下所示。回想一下，已将服务器配置为将问候消息记录到标准输出中。此消息包含从证书中提取的连接客户端的通用名称：

```bash
$ go run main.go
Hello: client.blackhat-go.local
```

现在，已经有了相互身份验证的例子。为加强理解，我们建议使用TCP socket 修改示例。

在下一部分中，竭尽全力达到更高的目标：暴力破解RC2加密的对称密钥。

## 暴力破解RC2

RC2是罗恩·里维斯特（Ron Rivest）于1987年创建的对称密钥分组密码。在政府的建议下，设计人员使用了40位加密密钥，该密码的强度足以使美国政府对密钥进行暴力破解和解密通讯。例如，它为大多数通信提供了充分的机密性，但允许政府窥探与外国实体的通话。当然，早在1980年代，强行破解密钥就需要强大的计算能力，只有资金雄厚的国家或专业组织才能在合理的时间内解密该密钥。快速发展30年的今天，普通家用计算机可以在几天或几周内暴力破解40位密钥。

所以，管他呢，让我们暴力破解40位密钥。

### 开始

在深入研究代码之前，先做好准备。首先，Go的标准库和扩展加密库都没有可用的RC2包。但是，有一个内部Go包。无法直接在外部程序中导入内部包，因此必须找其他的使用方式。

其次，为简单起见，通常假设一些不希望创建的数据。具体是，假定明文数据的长度是RC2块大小（8字节）的倍数，避免处理PKCS \#5填充等管理任务造成逻辑混乱。处理填充类似于本章前面使用AES所做的事情\(请参见清单11-4\)，但是需要更仔细地验证内容，以维护将要处理的数据的完整性。还将假设密文是一个加密的信用卡号。通过验证生成的纯文本数据来检查可能的密钥。在这种情况下，验证数据涉及确保文本为数字，然后对其进行 `Luhn check`，这是验证信用卡号和其他敏感数据的一种方法。

接下来，假设能够（也许是通过窃取文件系统数据或源代码）确定了使用40位密钥以ECB模式加密了数据而没有初始化矢量。RC2支持可变长度的密钥，并且由于它是分组密码，因此可以在不同的模式下运行。在最简单的模式ECB模式下，数据块独立于其他块进行加密。逻辑更简单了。最后，尽管可以并发破解密钥，如果这样做的话，并发实现的性能会好得多。与其先构建非并发版本，再去迭代，不如从一开始就直接构建并发的版本。

现在再安装几个需要的包。首先从 [https://github.com/golang/crypto/blob/master/pkcs12/internal/rc2/rc2.go](https://github.com/golang/crypto/blob/master/pkcs12/internal/rc2/rc2.go) 获取官方Go实现的RC2。将其安装在本地工作区中，以便导入到暴力破解工具中。正如我们前面提到的，该包是内部包，也就意味着在默认情况下，外部包无法导入和使用。虽然点麻烦，但是也就不用使用第三包，或者自己去实现RC2加密了。如果复制到工作区中，就可以访问未导出的函数和类型。

还要安装用于执行Luhn检查的包：

```bash
$ go get github.com/joeljunstrom/go-luhn
```

Luhn检查计算信用卡号码或其他识别数据的校验和来验证是否有效。使用现有的包，该包有非常好的文档，还不用重复造轮子。

现在可以开始写代码了。遍历整个密钥空间（40位）的每种组合，使用每个密钥解密密文，然后验证结果，即结果只有数字并能通过Luhn检查。使用生产者/消费者模型来实现——生产者把密钥发送到管道中，而消费者从管道中读取密钥并执行。工作本身是一个单一的键值。当找到通过正确验证的明文密钥（表明已找到信用卡号）时，向每个goroutine发信号让其停止工作。

这个问题的一个有趣挑战是如何迭代键空间。我们的解决方案是，使用for循环对其进行迭代，遍历表示为uint64值的键空间。正如所看到的，挑战是uint64占用了内存的64位空间。因此，从uint64转换为40位（5字节）\[\] byte RC2密钥要裁剪掉24位（3字节）的不必要数据。希望看到代码就会明白该过程。我们会慢慢来，分解程序的各个部分，一个一个地进行。从清单11-8开始。

```go
import ( 
    "crypto/cipher"
    "encoding/binary" 
    "encoding/hex" 
    "fmt"
    "log"
    "regexp" 
    "sync"

    luhn "github.com/joeljunstrom/go-luhn" 

    "github.com/bhg/ch-11/rc2-brute/rc2"
)
var numeric = regexp.MustCompile(`^\d{8}$`)

type CryptoData struct { 
    block cipher.Block
    key []byte 
}
```

清单 11-8：导入 RC2 暴力破解类型 \(/ch-11/rc2-brute/main.go\)

为了强调第三方 `go-luhn` 包和从Go内部库中克隆的 `rc2` 包，我们在清单中列出了 import 语句。还编译了一个正则表达式，用来检查生成的纯文本块是否为8字节的数据。

请注意，检查的是8字节的数据，而不是16字节的数据，因为这是信用卡号的长度。检查8个字节，还因为这是RC2块的长度。逐块解密密文，因此可以检查解密的第一个块是否为数字。如果该块的8个字节不全是数字，则可以肯定地推断出没有使用信用卡号，完全可以跳过第二个密文块的解密。性能上的微小改进将大大减少执行数百万次所需的时间。

最后，定义名为 `CryptoDatax` 的类型，该类型将用于存储密钥和 `cipher.Block`。使用该`struct` 来定义工作单位，即生产者创建的工作单元，消费者进行操作的单元。

### 生成者

让我们看一下生产者函数（清单11-9）。将该函数放置在前面的代码清单中的类型定义之后。

```go
func generate(start, stop uint64, out chan <- *CryptoData, done <- chan struct{}, wg *sync.WaitGroup) {
    wg.Add(1) 
    go func() {
        defer wg.Done() 
        var (
            block cipher.Block 
            err error
            key []byte
            data *CryptoData
        )
        for i := start; i <= stop; i++ {
            key = make([]byte, 8) 
            select {
            case <- done: 
                return
            default: 
                binary.BigEndian.PutUint64(key, i)
                if block, err = rc2.New(key[3:], 40); err != nil { 
                    log.Fatalln(err)
                }
                data = &CryptoData{
                       block: block,
                       key:   key[3:],
                }
                out <- data 
            }
        }
    }()

    return 
}
```

清单 11-9: RC2 生产者函数 \(/ch-11/rc2-brute/main.go\)

生产者函数名为 `generate()` 。该函数接受两个uint64变量，这些变量用于定义生产者将在其上创建工作的密钥空间的一部分（基本上是他们将产生密钥的范围）。可以分解密钥空间，并将部分分配给每个生产者。

该函数参数还有两个管道：`CryptData` 只写管道，用于将工作推送给消费者，一个普通的`struct` 管道，该管道用于接收来自消费者的信号。第二个管道是必需的，例如，识别出正确密钥的使用者可以明确地通知生产者停止生产。如果已经解决了问题，那么创建更多的工作毫无意义。最后，函数接受一个 `WaitGroup`，用于跟踪和同步生产者的执行。对于每个并发运行的生产者，执行 `wg.Add(1)` 告诉`WaitGroup` 启动了一个新的生产者。

在goroutine中填充工作管道，还有当 goroutine退出时调用 `defer wg.Done() 通知`WaitGroup`。这样就不会出现死锁，`main\(\)`函数可以继续执行。`for`循环中的`start`和`stop\` 来迭代key剩余的部分。循环的每次迭代都会增加i变量，直到达到结束偏移量为止。

如前所述，key大小为40位，而i为64位。这种大小差异对于理解至关重要。Go中没有40位的类型。只有32位或64位类型。由于32位太小而无法容纳40位，因此需要改用64位，并在以后再考虑额外的24位。如果可以使用\[\]byte而不是uint64来迭代整个key，则可以避免。但是这样做可能需要一些位操作，可能会使示例过于复杂。因此，还是处理长度的细微差别。

在循环中使用一个select语句，乍一看可能很愚蠢，因为是操作管道中的数据，不适合典型语法。通过 `case <- done`来检查 `done` 管道是否关闭。如果管道关闭，则使用 `return` 退出goroutine。当 `done` 管未关闭时，执行 `default` 分支创建定义工作所需的加密实例。具体是，调用 `binary.BigEndian.PutUint64(key, i)` 将uint64值（当前密钥）写入名为key 的 \[\]byte中。

尽管之前没有明确指出，但还是将key初始化为8字节切片。那么为什么只处理5字节key，还要将切片定义为8字节呢？这是因为`binary.BigEndian.PutUint64` 带有uint64值，所以它需要8字节的目标切片，否则会抛出超出索引范围的错误。注意剩余的代码中，仅用到了key切片的最后5个字节。即使前3个字节为零，如果包括在内，它们仍将破坏加密函数的紧缩性。这就是为什么最初调用 `rc2.New(key [3：], 40)` 来创建密码的原因。这样就丢弃3个无关的字节，并且还会传入密钥的长度（以位为单位）：40。使用生成的`cipher.Block` 实例和相关的key字节创建 `CryptoData` 对象，并将其写入out 消费者管道。

这是生产者代码。注意在本部分中，仅引导需要的相关关键数据。在这个函数中，没有任何地方是真正解密的。解密工作将会在消费者函数中执行。

### 运行并解密数据

现在来看一下消费者函数（清单11-10）。同样地将此函数添加到与先前代码相同的文件中。

```go
func decrypt(ciphertext []byte, in <- chan *CryptoData, done chan struct{}, wg *sync.WaitGroup) {
    size := rc2.BlockSize
    plaintext := make([]byte, len(ciphertext)) 
    wg.Add(1)
    go func() {
        defer wg.Done()
        for data := range in {
            select { 
            case <- done:
                return 
            default:
                data.block.Decrypt(plaintext[:size], ciphertext[:size]) 
                if numeric.Match(plaintext[:size]) {
                    data.block.Decrypt(plaintext[size:], ciphertext[size:]) 
                    if luhn.Valid(string(plaintext)) && numeric.Match(plaintext[size:]) {
                        fmt.Printf("Card [%s] found using key [%x]\n", plaintext, data.key)
                        close(done)
                        return
                    } 
                }
            } 
        }
    }() 
}
```

清单 11-10: RC2 消费者函数 \(/ch-11/rc2-brute/main.go\)

消费者函数名为 `decrypt()` ，接收几个参数。接收待解密的密文。还接受两个管道：一个名为 `*CryptoData` 的只读管道，用作工作队列），一个名为 `done` 的管道，用于发送和接收明确的取消信号。最后还接受一个名为 `wg` 的 `*sync.WaitGroup`，用于管理消费者，非常像生产者的实现。调用`wg.Add(1)`告诉WaitGroup启动一个消费者。这样，就可以跟踪和管理所有运行中的消费者了。

接下来，在goroutine中，调用 `defer wg.Done()`，以便在goroutine函数结束时，更新WaitGroup状态，从而将正在运行的工作程序数量减少1。这种WaitGroup业务对于跨任意数量的工作者同步程序的执行是必要的。稍后在 `main()` 函数中使用 `WaitGroup` 来等待`goroutines` 执行完成。

消费者在 `for` 循环中重复读取 `in` 管道中的 `CryptoData` 。管道关闭时循环停止。这个管道是由生产者填充的。很快就会看到，在生产者遍历它们的整个key空间子部分并将相应的加密数据推入工作管道道后，该管道关闭。因此，消费者不断循环，直到生产者完成生产为止。

像生产者代码那样，在for循环中使用 `select` 语句来检查 `done` 管道是否关闭，如果已经关闭，则显式地通知消费者停止额外的工作。识别出有效的信用卡号时将关闭通道，稍后我们将对此进行讨论。`default` 分支执行加密的工作。首先，解密密文的第一个块（8个字节），检查生成的明文是否为8字节的数字值。如果是，则可能是卡号，然后继续解密第二个密文块。从管道中读取 `CryptoData` 对象，通过对象中的 `cipher .Block` 字段调用解密函数。回想一下，生产者使用从key空间获取的唯一key实例化了该结构。

最后，根据Luhn算法验证整个明文，并验证第二个明文块是一个8字节的数字。如果这些检查成功，则可以确定找到了一个有效的信用卡号。在stdout显示卡号和输入的key，然后调用 `close(done)`向其他goroutine发出信号，表明已经找到了。

### Main 函数

至此，已经有了生产者和消费者函数，都可以并发执行。现在，在 `main()` 函数（清单11-11）中将这些内容整合在一起，代码也在与前面清单相同的源文件中。

```go
func main() {
    var (
        err            error
        ciphertext    []byte
    )

    if ciphertext, err = hex.DecodeString("0986f2cc1ebdc5c2e25d04a136fa1a6b"); err != nil {
        log.Fatalln(err)
    }

    var prodWg, consWg sync.WaitGrou
    var min, max, prods = uint64(0x0000000000), uint64(0xffffffffff), uint64(75)
    var step = (max - min) / prods

    done := make(chan struct{})
    work := make(chan *CryptoData, 100)
    if (step * prods) < max {
        step += prods 
    }
    var start, end = min, min + step
    log.Println("Starting producers...") 
    for i := uint64(0); i < prods; i++ {
        if end > max { 
            end = max
        }
        generate(start, end, work, done, &prodWg)y 
        end += step
        start += step
    }
    log.Println("Producers started!") log.Println("Starting consumers...") 
    for i := 0; i < 30; i++ {
        decrypt(ciphertext, work, done, &consWg)
    }
    log.Println("Consumers started!") 
    log.Println("Now we wait...") 
    prodWg.Wait()
    close(work)
    consWg.Wait()}
    log.Println("Brute-force complete") 
}
```

清单 11-11: RC2 \*main\(\) 函数 \(/ch-11/rc2-brute/main.go\)

`main()` 函数对密文进行解码，以十六进制字符串表示。接下来定义几个变量。第一个， `WaitGroup` 变量用于追踪生产者和消费者的 goroutine。还定义了几个uint64值，跟踪40位key的最小值（0x0000000000），最大值（0xffffffffff），及打算启动的生产者数量，在代码中为75。使用这些值来计算每个生产者要迭代key的数量的步长或范围，因为要将这些工作均匀地分配给所有生产者。还创建一个 `*CryptoData` 管道道和一个 `done` 管道。将它们传递给生产者和消费者。

因为需要做基本的整数计算来计算生产者的步长值，所以如果key大小不是生产者数量的倍数，则很有可能会丢失一些数据。为了解决这个问题——并避免在转换为浮点数时丢失精度，调用 `math.Ceil()` ——检查最大key \(step \* prods\)是否小于整个key的最大值\(0xffffffffff\)。如果是这样，则不会考虑key中少数的几个值。只需增加 `step` 值即可解决这一不足。初始化两个变量 `start` 和 `end`，维持分割key的起始偏移量和结束偏移量。

无论如何，计算偏移量和步长都不精确，这可能会导致代码搜索超出最大允许的key。但是，可以在每个生产者的 `for` 循环中修复该问题。在循环中，如果该值超出最大key，则可以调整结束步长值 `end`。循环的每次迭代都会调用生产者函数 `generate()`，并且每次迭代时将key的偏移开始（start）和结束（end）传递给它。还将 `work` 和 `done` 管道，`WaitGroup` 传递给它。调用该函数后，移动`start` 和 `end` 变量，准备下次循环时传递给新生产者。这样就把秘钥分成较小的、更易计算的部分，程序可以并发处理，goroutine之间也不会进行重复工作。

生产者启动后，使用 `for` 循环来创建工作。本例中创建了30个。每次迭代都调用 `crypto()` 函数，并将密文，work管道，`done`管道和消费者的 `WaitGroup` 传递给该函数。这样消费者就并发的工作，当生产者创建工作时，并发消费者就开始拉取和处理工作。

遍历整个key花费的时间。如果处理不正确，`main()` 函数肯定会在发现密钥或耗尽密钥空间之前退出。因此，需要确保生产者和消费者有足够的时间来迭代整个密钥空间或发现正确的密钥。这就需要用到 `WaitGroups` 。调用 `prodWg.Wait()` 阻塞 `main()` 函数，直到生产者完成任务。回想一下，如果生产者耗尽了密钥空间，或通过 `done` 管道明确地取消了处理，生产者完成了他们的任务。任务完成后，明确地关闭 `work` 管道，防止消费者从 `work` 管道读取数据时死锁。最后，调用 `consWg.Wait()` 再次阻塞 `main()`，以便为`WaitGroup` 中的使用者提供足够的时间来完成工作通道中的所有剩余的 `work`。

### 运行程序

程序已经完成了！如果运行会有下面的输出：

```bash
$ go run main.go
2020/07/12 14:27:47 Starting producers...
2020/07/12 14:27:47 Producers started!
2020/07/12 14:27:47 Starting consumers...
2020/07/12 14:27:47 Consumers started!
2020/07/12 14:27:47 Now we wait...
2020/07/12 14:27:48 Card [4532651325506680] found using key [e612d0bbb6] 2020/07/12 14:27:48 Brute-force complete
```

程序启动了生产者和消费者，然后等待他们执行。当找到信用卡时，程序将显示明文的卡号和用于解密该卡的密钥。因为我们假设此密钥是所有卡的神奇密钥，所以我们提前中断执行，并通过画一幅自画像来庆祝我们的成功。

当然，根据秘钥的不同，家用计算机上的暴力破解可能会花费大量时间——数天甚至数周。对于前面运行的示例，为了更快地找到密钥而缩小了密钥空间。然而，在2016年的MacBook Pro上完全耗尽秘钥空间大约需要7天。对于在笔记本电脑上运行的快速而肮脏的解决方案来说，这还不算太坏。

## 总结

对于安全从业者来说，加密是一个重要的课题，尽管学习过程可能有些周折。本章介绍了对称和非对称加密、哈希、使用bcrypt处理密码、消息身份验证、相互身份验证和暴力破解RC2。在下一章中，我们将深入探讨如何攻击Microsoft Windows。

