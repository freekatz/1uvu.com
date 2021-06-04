<h1 align="center">项目名称：go-storage 的 ipfs</h1>

# 一、项目详细方案

## 1 技术解读

第一部分会对整个项目涉及到的几个关键技术做一些笼统的概述，重点介绍与项目实现息息相关的部分。

### 1.1 go-storage

`Beyond Storage` 是一个处于已存在存储服务之上的存储抽象[^1]，而 `go-storage` 是它的 golang 实现。

Beyond Storage 提供强大的扩展能力和伸缩能力，开发者可以通过 `Beyond Storage` 基于统一的接口，去访问不同的存储服务中的不同类型的数据，这免去了开发者有关适配各个平台存储服务的工作量。

提供如下两方面的操作支持[^2]：

-   [Servicer](https://beyondstorage.io/zh-CN/docs/go-storage/operations/servicer/index)：服务级别的管理操作
-   [Storager](https://beyondstorage.io/zh-CN/docs/go-storage/operations/storager/index)：基础存储对象操作
    -   [Copier](https://beyondstorage.io/zh-CN/docs/go-storage/operations/copy): 在 Storager 中复制一个对象
    -   [Mover](https://beyondstorage.io/zh-CN/docs/go-storage/operations/move): 在 Storager 中移动一个对象
    -   [Reach](https://beyondstorage.io/zh-CN/docs/go-storage/operations/reach): 给对象生成一个可公开访问的 url
    -   [Multiparter](https://beyondstorage.io/zh-CN/docs/go-storage/operations/multiparter): 允许进行分段上传
    -   [Appender](https://beyondstorage.io/zh-CN/docs/go-storage/operations/appender): 允许追加写入到对象 （Append）
    -   [Block](https://beyondstorage.io/zh-CN/docs/go-storage/operations/blocker): 允许使用 Block 来组合一个对象
    -   [Page](https://beyondstorage.io/zh-CN/docs/go-storage/operations/pager): 允许随机写入操作
    -   [Fetcher](https://beyondstorage.io/zh-CN/docs/go-storage/operations/fetch): fetch from a given url to path

Servicer 和 Storager 的接口定义均在：https://github.com/beyondstorage/go-storage/blob/master/types/operation.generated.go，其中，接口定义特别要求实现接口时要混合一个 `Unimplemented `的 已实现类型，为了使得声明接口变量时不会 panic，且可以检查出哪些接口的方法尚未被实现。

下面，简单地分析一下 go-storage 的仓库源码[^3]：

-   [cmd/definitions](https://github.com/beyondstorage/go-storage/tree/master/cmd/definitions)：提供了很多种类的模板代码，及模板生成代码，其中 `tmpl/service.tmpl` 是对于不同平台存储服务的模板代码。
-   [pairs](https://github.com/beyondstorage/go-storage/tree/master/pairs)：定义一些常用的 pairs getter。
-   [pkg](https://github.com/beyondstorage/go-storage/tree/master/pkg)：工具库
    -   [credential](https://github.com/beyondstorage/go-storage/tree/master/pkg/credential)：包括加密、签名等的认证工具库。
    -   [endpoint](https://github.com/beyondstorage/go-storage/tree/master/pkg/endpoint)：根据配置生成 endpoint 地址信息。
    -   [fswrap](https://github.com/beyondstorage/go-storage/tree/master/pkg/fswrap)：在 File 层面，从 I/O、FileInfo 及 HTTP 三个方面对 `Storager ` 的封装。
    -   [headers](https://github.com/beyondstorage/go-storage/tree/master/pkg/headers)：以 HTTP/2 的部分字段为 Object 添加首部信息。
    -   [httpclient](https://github.com/beyondstorage/go-storage/tree/master/pkg/httpclient)：与 HTTP 客户端和连接相关的封装。
    -   [iowrap](https://github.com/beyondstorage/go-storage/tree/master/pkg/iowrap)：对底层 I/O 操作的封装。
    -   [randbytes](https://github.com/beyondstorage/go-storage/tree/master/pkg/randbytes)：随机 []byte 生成。
-   [services](https://github.com/beyondstorage/go-storage/tree/master/services)：这里的服务指的是 go-storage 提供的服务，即获取和保存 Servicer 和 Storager 实例的函数。
-   [types](https://github.com/beyondstorage/go-storage/tree/master/types)：定义了各类数据类型的接口、结构体，其中 `operation.generated.go` 定义了 go-storage 向外提供的操作支持接口。

### 1.2 go-service-example

https://github.com/beyondstorage/go-service-example

`go-service-example` 是为开发不同平台 service 提供的模板仓库，下面详细分析一下仓库结构[^4]：

-   [generated.go](https://github.com/beyondstorage/go-service-example/blob/master/generated.go)：由 [cmd/definitions](https://github.com/beyondstorage/go-storage/tree/master/cmd/definitions) 下的 `tmpl/service.tmpl` 自动生成。
-   [service.toml](https://github.com/beyondstorage/go-service-example/blob/master/service.toml)：定义开发相关配置，用于生成代码。
-   [storage.go](https://github.com/beyondstorage/go-service-example/blob/master/storage.go)：需要实现的 `Storager `接口声明的相关方法。
-   [tools.go](https://github.com/beyondstorage/go-service-example/blob/master/tools.go)：导入一些工具库。
-   [utils.go](https://github.com/beyondstorage/go-service-example/blob/master/utils.go)：定义 `Storage `结构体，及一些工具函数。

如果有需要，还可以额外增加其它源文件，如包含额外 `error `类型定义的 `error.go`。

另外，如果，需要实现其它接口，如 Srevicer，同样在对应位置增加代码，并新增源文件 `service.go`。

### 1.3 IPFS

这里会简要的介绍一下 IPFS 的原理和工作模式，着重探讨访问 IPFS 网络的两种方式。

首先，IPFS 是什么？

>   IPFS[^5] 是一个点对点的超媒体协议，致力于让网络更加快速、安全和开放。

它融合了很多经过时间检验的优秀思想和设计，如 BT 协议、DHT 分布式哈希表等等，有些人将其视为 Web3.0 的划时代颠覆性技术。暂时不谈 IPFS 想要取代 HTTP 作为新一代媒体传输协议的野心，IPFS 本身还可以被视为一种去中心化的分布式文件系统，与区块链相结合，可以提供一种十分优秀的文件分布式存储方案。

IPFS 分为公网和私网，持有相同 swarm key 的一组节点被视为处于相同的网络（前提是这一群节点都各自批准加入，一般会通过 ipfs cluster 批量处理）。

IPFS 提供了多种通过编程语言访问 IPFS 网的途径，包括 [^6]：

-   [interface-go-ipfs-core · pkg.go.dev](https://pkg.go.dev/github.com/ipfs/interface-go-ipfs-core)：一个相对稳定、简单但是比较陈旧的 api 库实现。
-   [coreapi · pkg.go.dev](https://pkg.go.dev/github.com/ipfs/go-ipfs/core/coreapi)：直接使用 `go-ipfs` 的 `core/coreapi` 来与 IPFS 交互，不稳定，不推荐，但是不要求目标 IPFS 网必须开放 5001 API 端口。
-   [go-ipfs-api · pkg.go.dev](https://pkg.go.dev/github.com/ipfs/go-ipfs-api)：基于 HTTP 访问的与 IPFS 网交互的库实现，较稳定。
-   [go-ipfs-http-client · pkg.go.dev](https://pkg.go.dev/github.com/ipfs/go-ipfs-http-client)：基于 HTTP API 实现的 `interface-go-ipfs-core` 库，提供更多、更新的特性，不太稳定。如果倾向于做更少的修改且不追求 IPFS 的新特性，则推荐使用 `go-ipfs-api`。

在仔细斟酌之后，个人认为 `go-ipfs-api` 是最好的选择。

## 2 实现Storager

据社区文档的介绍[^7]及相关源码，实现 Storager 需要完成如下操作的实现。

### 2.1 Basic Operations

第一部分是 Storager 的基础操作。

-   List：List will list objects under storage.
-   Create：Create will create an object struct without any API call.
-   Delete：Delete will delete an Object from storage.
-   Metadata：Metadata will return current storager’s metadata.
-   Read：Read will read the Object data.
-   Stat：Stat will stat a path to get info of an object.
-   Write：Write will write data into an Object.

### 2.2 Extended operations

第二部分是额外扩展的操作，用于实现其它属于 Storager 的接口，即 Copier、Mover 等等。

-   Copy：Copy will copy an Object in Storage.
-   Move：Move will move an Object in Storage.
-   Reach：Reach will get a public url to reach the object.
-   Multipart：Multipart will construct an Object via multiparts.
-   Append：Append will construct an Object via append operations.
-   Block：Block will construct an Object via blocks.
-   Page：Page will construct an Object via pages.

# 二、项目开发时间规划

这里将整个项目开发划分为四个阶段：

**准备**

-   [ ] 2021-07-01 => 2021-07-15：熟悉项目相关工具和库，项目结构和开发规范设计。

**基础开发**

-   [ ] 2021-07-16 => 2021-08-05：完成 Storager 基础操作的开发。
-   [ ] 2021-08-05 => 2021-08-15：调试和测试 Storager 基础操作。

**扩展开发**

-   [ ] 2021-08-16 => 2021-09-00：完成 Storager 部分扩展操作的开发。
-   [ ] 2021-09-01 => 2021-09-10：调试和测试 Storager 的所有操作。

**文档书写**

-   [ ] 2021-09-11 => 2021-09-20：书写文档、代码注释。

**其余时间留作备用，以防意外**

# 参考文献

[^1]: [介绍 | BeyondStorage](https://beyondstorage.io/zh-CN/docs/)
[^2]: [go-storage | BeyondStorage](https://beyondstorage.io/zh-CN/docs/go-storage/index#全面的操作支持)
[^3]: [beyondstorage/go-storage: A unified storage layer for Golang, let's go beyond storage. ](https://github.com/beyondstorage/go-storage)
[^4]: [beyondstorage/go-service-example: The example for creating a go-storage service.](https://github.com/beyondstorage/go-service-example)
[^5]: [IPFS Powers the Distributed Web](https://ipfs.io/)
[^6]: [Go-IPFS API | IPFS Docs](https://docs.ipfs.io/reference/go/api/)

[^7]: [operations/storager | BeyondStorage](https://beyondstorage.io/docs/go-storage/operations/storager/index)