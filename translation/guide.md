# Okapi 参考和指南

Okapi 是一个运行和管理微服务的网关

<!-- Regenerate this as needed by running `make guide-toc.md` and including its output here -->
* [简介](#简介)
* [架构](#架构)
    * [Okapi自身的Web服务](#okapi自身的web服务)
    * [部署和发现](#部署和发现)
    * [请求处理](#请求处理)
    * [状态码](#状态码)
    * [报头合并规则](#报头合并规则)
    * [版本控制和依赖](#版本控制和依赖)
    * [安全](#安全)
    * [未解决问题 ](#未解决问题)
* [实现](#实现)
    * [丢失的特性](#丢失的特性)
* [编译和运行](#编译和运行)
* [使用 Okapi](#使用-okapi)
    * [存储](#存储)
    * [Curl 示例](#curl-示例)
    * [运行 Okapi](#运行-okapi)
    * [示例 1: 发布和使用一个简单模块](#示例-1-发布和使用一个简单模块)
    * [示例 2: 添加认证模块](#示例-2-添加认证模块)
    * [示例 3: 升级、版本、环境以及`_tenant`接口](#示例-3-升级-版本-环境以及_tenant接口)
    * [示例 4: 完整的 ModuleDescriptor](#示例-4-完整的-moduledescriptor)
    * [多接口](#多接口)
    * [移除](#移除)
    * [集群模式](#集群模式)
    * [Securing Okapi](#securing-okapi)
    * [共享模块描述](#共享模块描述)
    * [为每个租户安装模块](#为每个租户安装模块)
    * [为每个租户升级模块](#为每个租户升级模块)
    * [自动部署](#自动部署)
    * [清除持久化数据](#清除持久化数据)
* [参考](#参考)
    * [Okapi](#okapi)
    * [环境变量](#环境变量)
    * [Web Service](#web-service)
    * [内部模块](#内部模块)
    * [部署](#部署)
    * [Docker](#docker)
    * [系统接口](#系统接口)
    * [数据监控](#数据监控)
* [模块相关](#模块相关)
    * [一个模块的生命周期](#一个模块的生命周期)
    * [租户接口](#租户接口)
    * [HTTP](#http)

## 简介

本文旨在介绍Okapi的相关概念及其整个生态系统（如：核心和模块），以及Okapi的实现和使用细节：通过提供具体的Web服务节点和请求处理的详细内容——处理请求、返回实例、状态码、 错误条件等等。

Okapi是在微服务架构中常用的一些不同微服务模式的实现。最核心的则是通Okapi代理服务实现的“API网关”。理论上讲，API网关是一个系统单一入口的服务。这有点类似于面向对象设计模式中的[外观模式](http://en.wikipedia.org/wiki/Facade_pattern)。Okapi会密切跟踪每个按照[标准定义](https://www.nginx.com/blog/building-microservices-using-an-api-gateway/)的API。_API网关封装了内部系统并且提供了一整套接口供客户端使用，包括认证、监控、负载均衡、缓存、请求整形和管理和静态请求处理在内的核心功能_。从消息队列模式中允许向多个服务广播请求（请求可以使同步的或者最终同步的，也可以是异步的），并并返回最终响应。最终，Okapi充当服务发现的工具以促进服务间的通信。服务器A想要与服务B通信只需要知道它的HTTP接口，因为Okapi将检查可用服务的注册表以找到服务的物理实例。

Okapi被设计成可配置可扩展的，它允许公开新的或现有的Web服务端点，而无需对软件本身进行编程更改。通过调用Okapi核心Web服务以注册新服务（如同Okapi的“modules”）。注册y和相关核心管理任务由服务提供管理员提供。这种可配置性和可扩展性对于app商店功能是很有必要的，它可以按需为每个租户启用或禁用服务或服务组。
## 架构

Okapi中的web服务端点大致可以分为两部分：(1)通用模块和租户管理API，有是有也被称为"core"——最初是Okapi的一部分，但是也可能拆分成自己的服务。(2)用于访问模块提供的或者业务逻辑特定接口的端点如读者管理、流通等。本文将详细讨论前者，并大致介绍后者允许的格式和样式。

目前Okapi的核心 Web服务的规范由[RAML](http://raml.org/)（RESTful API建模语言）定义，详见[参考](#web服务)。然而，该规范的目标是对特定模块公开的实际API端点做出少量假设，这些端点基本上没有定义。其目标在于兼容不同样式和格式的API接口（如RESTfult与RPC和Json与Xml等），只要求基本的HTTP协议。假设在某些特殊情况下(例如集成非http、二进制协议，类似于消息队列的操作的真正异步协议)，传输协议的可能会被解除或处理。

### Okapi自身的Web服务

如上所述，Okapi自身的Web服务提供了设置、配置和启用模块以及管理租户的基本功能，其核心节点如下：

 * `/_/proxy`
 * `/_/discovery`
 * `/_/deployment`
 * `/_/env`
 
  
特殊前缀`/_`被用于区分Okapi内部Web服务和模块提供的扩展点的路由。

* `/_/proxy` 端点用于配置代理服务：指定我们知道哪些模块，他们的请求路由如何，我们了解哪些租户，以及为哪些租户启用哪些模块。

* `/_/discovery` 端点用于管理从服务id到集群上的网络地址的映射。信息被发布至此，代理服务将查询它以找到所需模块的实际可用地址。它还为一次性部署和注册模块提供了快捷方式。一个集群中只需一个发现端点便可覆盖所有的节点。对发现服务的请求也可以在特定节点上部署模块，因此很少需要直接调用部署。

* `/_/deployment` 端点用于负责模块部署。在集群环境中，每一个节点应只一个部署实例。它将负责启动该节点上的进程，并为各个服务模块分配网络地址。它主要由发现服务(discovery service)在内部使用，但它是开放的，在某些集群管理系统中可以使用它。

* `/_/env` 端点用于管理环境变量，在部署期间将系统范围内的参数传递给模块。

这四部分被编码为单独的服务，因此如果所选择的集群系统提供这样的服务，则可以使用替代部署和发现方法。

![Module Management Diagram](https://raw.githubusercontent.com/folio-org/okapi/master/doc/module_management.png "Module Management Diagram")

#### 什么是"modules"?

Okapi生态系统中的模块（modules）是根据它们的 _行为_ (或者说是 _接口契约_ )而不是它的 _内容_ 来定义的,这意味着模块没有作为包或存档的确切定义，例如标准化的底层文件结构。这些细节留给特定的模块实现(如前文所述，Okapi服务器端模块可以使用任何技术堆栈)。

因此，任何具有以下特征的软件都可以成为Okapi模块:

* 它是一个HTTP网络服务器，使用REST风格的web服务协议进行通信——通常使用json格式。

* 它附带一个描述符文件，[`ModuleDescriptor.json`](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml/ModuleDescriptor.json)，它声明基本模块元数据(id、名称等)，指定模块对其他模块的依赖关系(确切地说是接口标识符)，并上报所有提供的接口。
 
* `ModuleDescriptor.json` 有一个给定模块处理的所有'路由'列表 (HTTP路径和方法)，为Okapi的流量代理提供了必要的信息(这类似于一个简化的RAML规范)

* 遵循[版本控制和依赖关系](#版本控制和依赖关系)一章中定义的版本控制规则

* 提供监视和检测所需的接口

如你所见，这些需求都没有明确规定部署规则，因此完全可以将第三方Web服务（如，公开访问的网络服务API）集成为Okapi的模块。也就是说，端点样式和版本控制语义与Okapi中需要的相匹配，就可以编写合适的模块描述符来描述它。

Okapi还包含额外的服务（用以服务部署和发现），它允许在自己管理的集群上本地执行、运行和监视服务。这些 _本地模块_ 需要一个额外的描述文件[`DeploymentDescriptor.json`](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml/DeploymentDescriptor.json)用以指定关于如何运行模块的底层信息。此外，本机模块必须根据Okapi的部署服务所支持的打包选项进行打包。这意味着在每个节点上提供可执行文件(和所有依赖项)，或者使用自包含的Docker镜像集中地分发可执行文件。

#### API 指南

Okapi自己的Web服务必须遵守以及其他模块应该尽可能地遵循原则。

* 路径最后没有斜杠

* 始终期望并返回正确的JSON

* 主键名应始终使用“id”


我们试图规范Okapi的代码，这样它就可以很好地作为其他模块开发人员参照的示例



#### Okapi Web服务的核心：身份验证和授权

对核心服务（/_/ path下的所有资源）的访问权授予服务提供方(Service Provider ，SP)的管理员，因为这些服务提供的功能跨越多个租户。<font color=red> SP管理员的身份验证和授权的详细信息将在稍后的阶段定义，很可能由一个可以连接到特定服务提供者身份验证系统的外部模块提供</font>


### 部署和发现

使一个模块对租户可用是一个多步骤的过程，有几种不同的方法可以做到，但最常见的过程是：

* 向`/_/proxy`发布一个ModuleDescriptor，告诉Okapi我们知道这样的模块，它提供什么服务，依赖什么。
* 向`/_/discovery`发送消息说，我们希望在给定节点上运行这个模块，它将告诉该节点上的部署服务模块启动必要的流程
* 为给定的租户启用模块

我们假设一些外部管理程序将发出这些请求，因为它需要在部署任何模块之前运行，所以其本身不是一个合适的Okapi模块。如果要进行测试，则请参阅稍后的curl命令行[示例](#使用-okapi)

另一种方法是不将模块ID传递给发现服务，而是传递完整的LaunchDescriptor。在这种情况下，ModuleDescriptor甚至可能没有LaunchDescriptor。如果运行在一个节点非常不同的集群上，并且您希望精确地指定要找到文件的位置，那么这将非常有用。<font color=red>这不是我们想要的Okapi集群运行的方式，但是我们希望保留这个选项</font>

另一种则是使用更低的级别，并将LaunchDescriptor直接发布到任何给定节点上的`/_/deployment`。这意味着管理软件必须直接与各个节点通信，这就提出了有关防火墙等的各种问题。但它允许完全控制，这在一些特殊的集群设置中非常有用。注意，你仍然需要向`/_/proxy`发布一个ModuleDescriptor来让Okapi知道这个模块，但是`/_/deployment`将通知`/_/discovery`存在已经部署的模块。

当然，如果你根本不需要使用Okapi来管理部署，则可以向`/_/discovery`发布DeploymentDescriptor，并提供URL而不是LaunchDescriptor。这将告诉Okapi服务在哪里运行。它仍然需要一个服务ID来将URL连接到之前发布的ModuleDescriptor。与前面的示例不同，你需要为`/_/discovery`提供一个惟一的实例Id来标识模块的这个实例。这一补是必要的，因为您可以在不同的url上运行相同的模块，也可能是在集群内部或外部的不同节点上。<font color=red>如果使用存在于集群之外的Okapi模块，或者使用一些容器（可能是web服务器，其中的模块作为CGI脚本驻留在不同的url中)，则此方法非常有用。<font>

<font color=red>请注意，部署和发现都是临时的，Okapi不会在其数据库中存储这些内容。如果一个节点宕机，其上的进程也将死亡。当它重新启动时，需要再次在其上部署模块，要么通过Okapi，要么通过其他方式。</font>

发现数据保存在一个共享映射中，因此只要集群上运行一个Okapi，该映射就会存在。但是如果整个集群被关闭，发现数据就会丢失。那时无论如何都是无用的。

相反，发布到`/_/proxy`的模块描述符被持久化到数据库中。

### 请求处理

模块可以声明两种处理请求的方式：处理或者过滤。每个路径应该只有一个处理程序。默认情况下是`request-response`类型(参见下面)，如果没有找到处理程序，Okapi将返回404 NOTFOUND

每个请求可以通过一个或多个过滤器传递。`phase`决定过滤器的顺序，目前我们定义了三个阶段的请求：

* `auth` 首先调用，用于检查X-Okapi-Token和权限。
* `pre` 在处理之前调用，用于日志记录和报告所有请求。
* `post` 在处理之后调用，用于日志记录和报告所有响应。

我们希望尽可能多的增加更多的`phase`

(在以前的版本中，我们将处理程序和过滤器组合在一个管道中，并使用数字级别来控制顺序。这在1.2中已经被弃用，并将在2.0中删除)

`type` 参数存在于Moduledescription的RoutingEntry中，用以控制如何将请求传递给过滤器和处理程序以及如何处理响应。目前我们支持如下类型：

* `headers`  -- 模块只对报头/参数感兴趣，它可以检查它们并根据头/参数的与否及其对应的值执行操作。模块不期望在响应中返回任何实体，而只返回一个状态代码来控制进一步的执行链，或者在出现错误时立即终止。模块可能返回某些响应头，这些响应头将根据后续的头操作规则合并到完整的响应头列表中。

* `request-only` -- 模块对完整的客户端请求感兴趣：报头/参数以及附加到请求的实体主体。返回的报头(包括响应代码)会影响进一步的处理，但是会忽略响应主体。<font color=red>注意，对于类型为`request-only`的请求，Okapi会将请求内容（假设为Post请求）缓存到内存冲。这不适用于大型导入或类似的操作。</font>如果可能忽略响应，则使用request-log。

* `request-response` -- 模块对报头/参数和请求体都感兴趣，还期望模块在响应中返回一个实体。这可能是一个修改过的请求体，在这种情况下模块可充当过滤器。返回的响应可以作为新的请求体转发给后续模块。同样，处理或终止传递是通过响应状态代码控制的，并且使用后续描述的规则将响应头合并回完整的响应中。

* `redirect` -- 模块不直接提供路径，而是将请求重定向到其他模块提供的其他路径上。这是一种用于将更复杂的模块堆积成在简单的实现上的机制。如：一个用于编辑和列出所有用户的模块可扩展于一个管理用户和密码的模块。该模块有实际的代码来处理和创建用户，可以将请求重定向到用户列表，并将获取用户转到更简单的模块上。如果处理程序（或者过滤器）被标注为重定向，则还必须包含一个"redirectPath"来告知重定向的位置。

* `request-log` -- 该模块对完整的客户端请求感兴趣：报头/参数和附加到请求的实体主体。这与`reqauest-only`相似，但Okapi将忽略整个响应，包括报头核响应代码。这种类型出现在Okapi版本2.23.0中。
 
* `request-response-1.0` -- 这与` request-response`相似, 但它使Okapi在发送到模块之前读取整个主体，以便设置"Content-Length"并禁用分块编码。 这对于那些难以处理分块编码或需要在检查前获取内容长度的模块非常有用。这种类型出现在Okapi 2.5.0中。

大多数请求的类型可能是`request-response`类型，这是最强大的类型，但也可能是最低效的类型,因为它要求内容在模块的输入或输出流中。在可以使用更有效的类型的地方，使用更有效的类型。例如，身份验证模块的权限检查只查询请求的头部，不返回任何主体，因此它属于`headers`。但是，相同模块的初始登录请求会咨询请求体以确定登录参数，并且它还返回一条消息，那么它必须是`request-response`类型。
尽量避免使用`request-only`和`request-response-1.0`，因为Okapi会将这些HTTP请求实体完整的缓存在内存中。

Okapi有一个特性，模块异常是可以返回一个X-Okapi-Stop头，此时Okapi会终止管道，并返回该模块返回的结果，这要谨慎使用。例如，登录管道中的模块可能会得出这样的结论:由于用户来自安全办公室中的IP地址，所以他已经获得了授权，并终止将导致显示登录屏幕的事件序列。


<a id="chunked"/>虽然Okapi同时接受HTTP 1.0和HTTP 1.1请求，但它使用带分组编码的HTTP 1.1来连接模块。`request-response-1.0`除外。


### 状态码

管道的继续或终止由执行模块返回的状态代码控制。Okapi接受标准[HTTP状态码](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) 范围:

 * 2xx: 返回OK。如果模块返回此范围内的代码，Okapi将继续执行管道，并根据描述的规则将信息转发给后续模块，最后调用的模块返回的状态就是返回给调用者的状态。

 * 3xx: 重定向。管道被终止，响应(包括任何`Location`报头)立即返回给调用者。

 * 4xx-5xx: 用户请求错误或系统内部错误。如果模块返回此范围内的代码，Okapi将立即终止整个请求链并将状态代码返回给调用者。

### 请求头合并规则

由于Okapi将前一个模块的响应转发到管道中的下一个模块(例如用于额外的过滤/处理)，某些初始请求头将无效。例如当模块将实体转换为不同的内容类型或更改其大小。在将请求转发到下一个模块之前，需要根据模块的响应头值更新无效的头。与此同时，Okapi还会收集一组响应头，以便在处理管道完成时生成最终响应，并将其发送回原始客户端。

这两组请求头都按照以下规则进行修改：

* 提供关于请求实体主体的元数据的任何标头(如Content-Type、Content-Length等)，从最后一个元素合并而成，并响应回请求。

* 一组特殊的调试和监视头将从最后一个响应合并到当前请求中(以便将它们转发到下一个模块)

* 提供关于响应实体主体的元数据的报头列表被合并到最终的响应报头集中。

* 一组额外的特殊标头(调试、监视)或任何其他标头(应该在最终响应中可见)将合并到最终响应报头集中。

Okapi 总是向任何模块的请求添加X-Okapi-Url头。这将告诉模块如何在需要时进一步调用 Okapi。在启动 Okapi 时，可以在命令行上指定这个Url，并且它可以很好地指向多个 Okapi 实例前的某个负载平衡器。

### 版本控制和依赖

模块可以提供一个或多个接口，也可以使用其他模块提供的接口。接口具有版本，并且依赖项可能需要接口的给定版本。Okapi将在部署模块时以及为租户启用模块时检查依赖项和版本。

注意，我们可以为多个模块提供相同的接口。可以同时在Okapi中部署这些模块，但是在给定的时间内只能为任何给定的租户启用一个这样的模块。例如，我们可以有两种方法来管理我们的客户，一种基于本地数据库，另一种与外部系统通信。安装可以两者都知道，但是每个租户必须选择其中之一。

#### 版本号

模块软件的版本号由三个部分组成，如`3.1.41` -- 非常像[Semantic Versioning](http://semver.org/).接口版本只包含前两部分，因为它们没有实现版本。

第一个数字是主要版本。每当进行非严格向后兼容的更改时，它都需要递增，例如删除功能或更改语义。

第二个数字是次要版本。无论何时进行向后兼容的更改，例如添加新功能或可选字段，都需要对其进行递增

第三个数字是软件版本。应该增加不影响接口的更改，例如修复bug或提高效率。

虽然强烈建议对所有模块使用此版本控制模式，但Okapi并没有强制要求对模块使用此模式。
其原因是Okapi不需要了解任何模块版本，它只关心接口是否兼容。

Okapi要求所有具有相同id的模块都必须是相同的软件，我们习惯即使用由一个简短的名称和一个软件版本组成的id。例如"test-basic-1.2.0"

在检查接口版本时，Okapi将要求主版本号与所需的完全匹配，并且子版本不低于所需版本。

如果模块需要接口版本为3.2，则接受:
* 3.2 -- 相同版本
* 3.4 -- 更高的小版本，接口兼容

但它会拒绝:
* 2.2 -- 低版本
* 4.7 -- 高版本
* 3.1 -- 较低的小版本

详见[Version numbers](https://dev.folio.org/guidelines/contributing#version-numbers).

### 安全

大多数关于安全的讨论已经转移至安全模块的文档中[Okapi Security Model](https://github.com/folio-org/okapi/blob/master/doc/security.md)。这章Okapi只是一个概述。

安全模型关注三件事:
* 认证 -- 知道用户是谁
* 授权 -- 此用于允许发出的请求
* 权限 -- 从用户角色映射的所有详细权限。大部分工作都委托给了模块，所以Okapi本身没有做太多工作，只需要协调整个操作。

安全模块大致是这样工作的：客户端（通常是浏览器，实际上可以试任何东西）调用`/auth/login`服务来标识自己。依据不同的租户可能有不同的授权模块服务于`/auth/login`请求，它们可能采用不同的参数（通常是用户名/密码，但是我们可以进行任何操作，从简单的IP身份验证到与LDAP、OAuth或其他系统的复杂交互）。

授权服务向客户端返回一个令牌，客户端在它向Okapi发出的所有请求中都将这个令牌传递到一个特殊的报头中。Okapi依次将其传递给授权模块，以及将调用哪些模块来满足请求的信息、这些模块需要和希望获得哪些权限，以及它们是否具有特殊的模块级权限。授权服务检查权限。如果没有所需的权限，则拒绝整个请求。如果一切正常，模块将返回有关所需权限的信息，以及可能传递给某些模块的特殊令牌。

Okapi依次将请求传递给管道中的每个模块。它们中的每一个都获得所需权限的信息，因此可以根据需要更改行为，以及可以用于进一步调用的令牌。

"okapi-test-auth-module"并没有实现此模块的大部分功能，它只是帮助我们测试Okapi需要处理的部分。

### 未解决问题

#### 缓存

Okapi可以在模块之间提供额外的缓存层，特别是在繁忙、读取量大的多模块管道中。我们计划在这方面遵循标准HTTP机制和语义，实现细节将在未来几个月内确定。

#### 工具和分析

在微服务架构中，监控是确保整个系统健壮性和健康的关键。提供有用监视的方法是在请求处理管道执行的每个步骤之前和之后包括定义良好的插桩点（“钩子”）。除了监视之外，检测对于快速诊断正在运行的系统中的问题(“热”调试)和发现性能瓶颈(性能分析)也是至关重要的。我们正在寻找这方面的既定解决方案:例如JMX，Dropwizard Metrics, Graphite等。

多模块系统可以提供各种各样的度量指标和大量的检测数据。只有一小部分数据可以在运行时进行分析，大部分数据必须被捕获，以便在稍后的阶段进行分析。捕获和存储数据的形式有利于毫不费力的事后事实分析是必不可少的分析，我们正在研究开放和流行的解决方案与Okapi之间的集成。

#### Response Aggregation

目前Okapi还没有支持Response  Aggregation，因为Okapi是顺序执行管道，并将每个响应转发到管道中的下一个模块中。在这种模式下，完全可以实现一个Aggregation模块，该模块将与多个模块通信(通过Okapi，保留提供的身份验证和服务发现)，并合并响应。在以后的版本中，将评估一种更通用的Response  Aggregation的实现方法。

#### 异步消息

目前Okapi无论在前端还是在系统内部都将HTTP作为模块之间的传输协议。HTTP是基于request-response范式，不直接包含异步消息的功能。但完全可以在HTTP上对异步操作建模，如轮训或者websocket之类的HTTP扩展。我们将在后续的版本中为其他的协议（如STOMP）提供支持。

## 实现

我们已经有了Okapi的基本实现。下面的示例应该适用于当前实现。

### 丢失的特性

暂无

## 编译和运行

Okapi最新的的代码可以在[GitHub](https://github.com/folio-org/okapi)上找到

编译环境：

 * Apache Maven 3.3.1 or later.
 * Java 8 JDK
 * [Git](https://git-scm.com) 2
 
所有的开发和运行只需要普通用户权限，无需root权限。

构建工程：

```
git clone --recursive https://github.com/folio-org/okapi.git
cd okapi
mvn install
```

安装并进行测试。测试一般不会失败，如果有问题请提交问题，并回退到之前的版本:

```
mvn install -DskipTests
```

如果运行成功，`mvn install`会输出如下结果

```
[INFO] BUILD SUCCESS
```

有些测试使用需要访问数据库，这可能需要一定的时间。可以使用以下方法跳过测试:
```
mvn install -DtestStorage=inmemory
```

Okapi目录包含如下几个子模块：

* `okapi-core` -- 网关服务器
* `okapi-common` -- 网关和模块都使用工具类
* `doc` -- 文档，包括本文
* `okapi-test-auth-module` -- 用于测试用户认证
* `okapi-test-module` -- 测试模块
* `okapi-test-header-module` -- 用于测试试headers-only模式

(注：`pom.xml`中指定的构建顺序: 由于 okapi-core测试依赖于前面的测试 所以必须放在最后）

每个模块和okapi-core都会将必要的组件打包成jar包。监听接口由`port`参数指定。

如要运行okapi-test-auth-module模块并监听8600端口，则使用：
```
cd okapi-test-auth-module
java -Dport=8600 -jar target/okapi-test-auth-module-fat.jar
```
同样，如果要运行okapi-core，除了port参数外还需提供一个额外的命令行参数以告诉告诉okapi-core以什么模式运行。在单个节点上使用Okapi时，使用`dev`模式。

```
cd okapi-core
java -Dport=8600 -jar target/okapi-core-fat.jar dev
```

可以通过`help`参数来查看命令的详细信息

当然也可以通过maven运行

```
mvn exec:exec
```

此时okapi-core将会运行并监听9130端口。

远程调试：

```
mvn exec:exec@debug
```
该命令需 Maven >= 3.3.1。客户端的调试端口为5005。

要在集群中运行，请参阅[集群](#集群模式)示例

The latest source of the software can be found at
[GitHub](https://github.com/folio-org/okapi).

## 使用 Okapi

这些事例将展示如何使用`curl`通过命令行的方式使用Okapi。你可以从该文档复制并粘贴命令到你的命令行。

These examples show how to use Okapi from the command line, using the `curl`
http client. You should be able to copy and paste the commands to your
command line from this document.

服务的确切定义在[参考](#web-service)部分中列出的RAML文件中。

The exact definition of the services is in the RAML files listed in
the [Reference](#web-service) section.

### 存储

Okapi默认为一个内存存储，可以脱离数据库层运行。这对于开发和测试来说非常方便，但是在实际当中，我们希望一些数据能够持久化。目前可以通过`-Dstorage=mongo`和`-Dstorage=postgres`选项分别开启MongoDB和PostgresSQL存储。

我们后台逐步弃用MongDB。所以如果使用MongDB存储请查看MongoHandle.java的代码。

初始化PostgreSQL数据库分为两步：

首先，需要在PostgreSQL中创建一个用户和一个数据库。在Debian系统上具体操作如下：
```
   sudo -u postgres -i
   createuser -P okapi   # When it asks for a password, enter okapi25
   createdb -O okapi okapi
```

"okapi"、"okapi25"和"okapi"是仅供开发使用的默认值。在生产系统中，一些DBA必须设置适当的数据库名及其参数，这些参数需要通过命令行传递给Okapi。

第二部是创建必要的表和索引。Okapi可以托管此部分操作：

```
java -Dport=8600 -Dstorage=postgres -jar target/okapi-core-fat.jar initdatabase
```

使用此命令时，会删除现有的表和数据并创建新的内容，并退出Okapi。如果只想删除现有的表则可以使用 `purgedatabase`参数。

如果想了解更多Okapi相关的PostgreSQL数据库，可通过如下命令查看：
```
psql -U okapi postgresql://localhost:5432/okapi
```

### Curl 示例

下面的示例可以复制粘贴到命令行控制台。

如果你有本问的mMarkDown文本时，也可以通过perl提取所有的示例记录。通过如下命令可以把它们都保存在' /tmp '文件中，如' okapi- rent .json 
```
perl -n -e 'print if /^cat /../^END/;' guide.md | sh
```

之后，也可以用稍微复杂一点的命令运行所有示例：
```
perl -n -e 'print if /^curl /../http/; ' guide.md |
  grep -v 8080 | grep -v DELETE |
  sh -x
```

（请参阅 `doc/okapi-examples.sh`脚本）

### 示例模块

Okapi是通过模块调用来实现的，所以我们提供一些模块来运行.提供三个虚拟模块来演示不同的内容。

注：这些仅用于演示和测试，不能以此为准

可以在[folio-sample-modules](https://github.com/folio-org/folio-sample-modules)中查看

#### Okapi-test-module

这是一个非常简单的模块。如果你发送GET请求，则它会返回"It works"。如果你发送POST请求则会返回"Hello"，后面会附上你说发送的内容。它还可以回显HTTP头，这功能被用于okapi-core的测试中。

通常Okapi会为自动启动和停止这些模块，但是为了了解如何使用curl命令，我们现在会直接运行这个模块
```
java -jar okapi-test-module/target/okapi-test-module-fat.jar
```

此时会运行`okapi-test-modual`并监听8080端口。

我们开启另外一个控制台并输入如下命令：

```
curl -w '\n' http://localhost:8080/testb
```
此时会返回"It works"。

参数`-w '\n'`只是为了换行，并不影响结果。

现在我们将POST一串数据给测试模块。通常再生产环境中内容会以JSON格式的形式存在，但是现在我们只输入一些文本。
```
echo "Testing Okapi" > okapi.txt
curl -w '\n' -X POST -d @okapi.txt http://localhost:8080/testb
```
同样 `-w`参数是为了换行，我们添加`-X POST`使其为POST请求，并添加`-d @okapi.txt`来指定我们想要的数据文件。

此时测试模块会返回：
```
Hello Testing Okapi
```

“Hello”后面便是我们提交的数据。

关于`okapi-test`模块的介绍就到这里，通过`Ctrl-C`来终止程序。

#### Okapi-test-header-module

`test-header` 模块是检查HTTP报头并生成一组新的HTTP报头。模块响应主体将被忽略，并且应该是空的。

开始测试:

```
java -jar okapi-test-header-module/target/okapi-test-header-module-fat.jar
```

该模块会从指向`/testb`的请求中读取`X-my-header`报头。如果报头存在将其获取并在其取值后添加`,foo`；反之则直接返回`foo`

用一下两种方式来说明:

```
curl -w '\n' -D- http://localhost:8080/testb
```
and
```
curl -w '\n' -H "X-my-header:hey" -D- http://localhost:8080/testb
```

综上所述。

#### Okapi-test-auth-module

Okapi本身不做身份验证，其将身份验证功能委托给相应的验证模块。在生产环境中，认证内容被细分为多个不同的模块，如`mod-authtoken`,`mod-login`,`mod-permissions`，但为了测试我们使用此模块来演示其如何运作的。

该模块支持两种功能：`/auth/login`是一个接收用户名、密码登陆的功能，如果认证成果则在HTTP头中返回令牌。其他的都要经过`check`功能，用以验证HTTP头中令牌的有效性。

在学习Okapi时，我们会接触到这样的例子。你可以像验证`okapi-test-module`一样直接验证此模块。

### 运行 Okapi

现在我们准备启动Okapi。对于这个例子来说，最重要的是Okapi的当前目录是顶级目录`…/ okapi`。
```
java -jar okapi-core/target/okapi-core-fat.jar dev
```

`dev`命令表示开发模式，这使得Okapi从一个完全干净的环境运行，没有任何的模块或租户。

Okapi会列出其相应的PID(进程ID)，并打印`Api Gatway started`。这意味着Okapi已启动并且监听默认9130这个默认端口，使用你内存存储（如果想要使用数据库存储，则添加  `-Dstorage=postgres` 参数,详见[命令行](#java--d-options))

当Okapi第一次启动时，它会检查它是否为本例中使用的所有节点的内部模块提供了一个"ModuleDescriptor"。如果没有，则会自动创建。我们可以查看Okapi下所有已知的模块：
```
curl -w '\n' -D -  http://localhost:9130/_/proxy/modules

HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-2.15.1-SNAPSHOT /_/proxy/modules : 200 8081us
Content-Length: 74

[ {
  "id" : "okapi-2.15.1-SNAPSHOT",
  "name" : "Okapi"
} ]
```

版本号会实时更新。因为本例是运行在一个开发版的分支上，所以有一个`-SNAPSHOT`后缀。

由于所有Okapi操作都是代表租户完成的，所以确保Okapi将启动时至少定义了一个操作。同样，你会看到：

```
curl -w '\n' -D - http://localhost:9130/_/proxy/tenants

HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-2.0.1-SNAPSHOT /_/proxy/tenants : 200 450us
Content-Length: 117

[ {
  "id" : "okapi.supertenant",
  "name" : "okapi.supertenant",
  "description" : "Okapi built-in super tenant"
} ]
```

### 示例 1: 发布和使用一个简单模块
我们需要告诉Okapi我们想要处理一些模块。在生产情况下，这些操作将由适当授权的管理员执行。

如上所述，这个过程分为部署、发现和配置代理三个步骤。

#### 部署 test-basic module

为了告诉Okapi我们想要使用`okapi-test-module`，我们需要创建一个JSON结构的moduleDescriptor文件并将其发送给Okapi：
```
cat > /tmp/okapi-proxy-test-basic.1.json <<END
{
  "id": "test-basic-1.0.0",
  "name": "Okapi test module",
  "provides": [
    {
      "id": "test-basic",
      "version": "2.2",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    }
  ],
  "requires": [],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
END
```

其中，id是我们之后要使用的模块的标识符。由于版本号包含在id当中，所以id可以唯一标识我们正在使用的模块。（Okapi不强制这样做，你也可以使用其他诸如UUID或别的东西，只要保证其唯一性即可。但是我们还是决定对所有模块使用这种命名方案。）

此模块仅提供一个叫做`test-basic`的接口，它只处理针对/testb路径下的GET和POST请求

launchDescriptor用以告诉Okapi如何启动和停止这个模块。在这个版本中，我们使用`exec`命令。Okapi会在其启动时记住PID，并在停止时杀掉此进程。

moduleDescriptor可以包含更多的内容，在后面的示例中会详细介绍。

我们现在发送如下内容:
```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-test-basic.1.json \
  http://localhost:9130/_/proxy/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/modules/test-basic-1.0.0
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/proxy/modules : 201 9786us
Content-Length: 370

{
  "id" : "test-basic-1.0.0",
  "name" : "Okapi test module",
  "requires" : [ ],
  "provides" : [ {
    "id" : "test-basic",
    "version" : "2.2",
    "handlers" : [ {
      "methods" : [ "GET", "POST" ],
      "pathPattern" : "/testb"
    } ]
  } ],
  "launchDescriptor" : {
    "exec" : "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
```

Okapi会做出"201 Created"的响应，并反应想用的JSON内容。这里有一个Loaction header用以显示模块的位置。如果我们想修改或者删除亦或仅仅查看下，可以这样：
```
curl -w '\n' -D - http://localhost:9130/_/proxy/modules/test-basic-1.0.0
```

我们也可以让Okapi列出所有已知的模块，就像我们一开始做的那样：
```
curl -w '\n' http://localhost:9130/_/proxy/modules
```

这显示了一个由两个模块组成的简短列表，一个是内部模块，另一个是我们刚刚部署的模块。

注意：真实情况下，Okapi关注模块的细节内容会更少。

#### 部署模块

让Okapi只到存在这样一个模块是不够的。我们必须部署模块。在这里，我们必须注意Okapi是一个运行在有许多节点的集群上，所以我们首先要决定将模块部署到哪一个节点上。首先我们必须检查正在使用那些集群：

```
curl -w '\n' http://localhost:9130/_/discovery/nodes
```

Okapi会返回一个只有一个节点的简短列表：
```
[ {
  "nodeId" : "localhost",
  "url" : "http://localhost:9130"
} ]
```

这是因为我们以`dev`模式运行，所以集群中只有一个节点，默认情况下被这种情况被称为"localhost"。如果是是一个真正的集群，每个节点都有自己的ID，要么在Okapi命令上指定，要么由系统分配一个UUDI。现在我们将其部署到那里：
```
cat > /tmp/okapi-deploy-test-basic.1.json <<END
{
  "srvcId": "test-basic-1.0.0",
  "nodeId": "localhost"
}
END
```

之后我们发送POST请求到`/_/discovery`。注意，我们并没有POST到`/_/deployment`，虽然我们可以这么做。其不同之处在于对于`deployment`我们只需要将其部署到实际的节点上，而`discovery`则负责知道在哪个节点上运行什么，并且可以给节点上的任何一个Okapi使用。在生产系统中，可能会有防火墙阻止对节点的任何直接访问。

```
curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-test-basic.1.json \
  http://localhost:9130/_/discovery/modules
```

Okapi的响应内容：
```
HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/discovery/modules/test-basic-1.0.0/localhost-9131
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/discovery/modules : 201
Content-Length: 237

{
  "instId" : "localhost-9131",
  "srvcId" : "test-basic-1.0.0",
  "nodeId" : "localhost",
  "url" : "http://localhost:9131",
  "descriptor" : {
    "exec" : "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
```

这里有比我们发送的更多的细节。我们只提供了服务ID"test-basic-1.0.0"，它使用这个ID从我们之前发布的ModuleDescriptor中查找LaunchDescriptor。

同时Okapi还为该模块分配了9131端口，并且给了一个实例ID"localhost-9131"。这是必要的，因为我们可以在不同的节点上运行相同模块的多个实例。

最后，Okapi还会返回模块正在监听的URL。在集群情况下，会有防火墙来阻止对模块的直接访问，所以所有的流量都必须通过Okapi进行授权和检查、日志记录等。但是在此例中，我们验证模块实际是在URL上面进行的。不管是哪个URL，当我们把处理程序和URL路径组合在一起时会得到如下内容：
```
curl -w '\n' http://localhost:9131/testb

It works
```

#### 创建一个租户

如上所述，所有模块的流量都应该通过Okapi，而不是直接到模块，但如果我们尝试Okapi自己URL，我们得到:
```
curl -D - -w '\n' http://localhost:9130/testb

HTTP/1.1 403 Forbidden
Content-Type: text/plain
Content-Length: 14

Missing Tenant
```

由于Okapi是一个多租户系统，因此每个请求都必须代表某个租户。我们可以使用超级租户，但这不是一个好的做法。让我们为这个示例创建一个测试租户。这并不难:
```
cat > /tmp/okapi-tenant.json <<END
{
  "id": "testlib",
  "name": "Test Library",
  "description": "Our Own Test Library"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-tenant.json \
  http://localhost:9130/_/proxy/tenants

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/proxy/tenants : 201 1065us
Content-Length: 91

{
  "id" : "testlib",
  "name" : "Test Library",
  "description" : "Our Own Test Library"
}
```

#### 为租户启用模块

接下来，我们需要为租户启用模块。这是一个更简单的操作:

```
cat > /tmp/okapi-enable-basic-1.json <<END
{
  "id": "test-basic-1.0.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-basic-1.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib/modules/test-basic-1.0.0
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/proxy/tenants/testlib/modules : 201 11566us
Content-Length: 31

{
  "id" : "test-basic-1.0.0"
}
```

#### 调用模块

现在我们有了一个租户，它启用了一个模块。上次我们尝试调用模块时，Okapi的响应是“Missing tenant”。我们需要在我们的调用中添加租户，作为一个额外的报头:
```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  http://localhost:9130/testb

HTTP/1.1 200 OK
Content-Type: text/plain
X-Okapi-Trace: GET test-basic-1.0.0 http://localhost:9131/testb : 200 5632us
Transfer-Encoding: chunked

It works
```

#### 另一种方式

对于给定的租户，还有另一种调用模块的方法，如下所示:
```
curl -w '\n' -D - \
  http://localhost:9130/_/invoke/tenant/testlib/testb

It works!
```

在某些情况下我们无法控制请求消息头，如SSO的回调服务，这时候由于需要令牌所以会调用失败。我们可以添加`/_/invoke/token/xxxxxxx/....`到路径上来处理这种情况。

该方法在Okapi 1.7.0中添加

### 示例 2: 添加认证模块

之前的例子,任何人都可以猜测租户的ID。对于这样的小测试来说是没什么问题的，但是实际生产情况下，模块只能提供给授权的用户。在实际情况下我们我们有一系列复杂的模块来管理所以类型的认证和授权。但在本例中，我们只有Okapi自己的`test-auth`模块可以使用。它不会进行严格的身份验证，但足以演示如何使用身份验证。

和之前一样，我们首先创建一个ModuleDescriptor:
```
cat > /tmp/okapi-module-auth.json <<END
{
  "id": "test-auth-3.4.1",
  "name": "Okapi test auth module",
  "provides": [
    {
      "id": "test-auth",
      "version": "3.4",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/authn/login"
        }
      ]
    }
  ],
  "requires": [],
  "filters": [
    {
      "methods": [ "*" ],
      "pathPattern": "/*",
      "phase": "auth",
      "type": "headers"
    }
  ]
}
END
```

此模块对于`/authn/login`路径有一个处理程序。并且还有一个过滤器可以连接到每个传入的请求。在这里用以决定是否允许客户端发出的请求。这里有个"headers"的类型，这意味着Okaip不会将整个的请求发送给它，而只是传递消息头。在真实情况下，这两个服务可能来自与不同的模块，如`mod-authtoken`用以过滤而`mod-login`用于某种用户身份验证。

过滤器的路径可以使用通配符(`*`)来匹配所有路径。还可以用`{}`匹配。如`/users/{id}`将匹配`/users/abc`而不是`/users/abc/d`。

此时为指定应用过滤器阶段。我们只有一个常用的`auth`阶段。它在程序处理之前被调用。另外还有`pre`和`post`分别用于程序处理之前和处理之后。我们可以根据具体的需要指定不同的阶段的过滤器。

我们可以像以前一样包含一个launchDescriptor，但为了演示另一种方法，在这里将其省略。在集群环境下这样做更为有意义，因为每一个模块可能需要额外的命令参数或环境变量。

我们发送给Okapi的是：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-module-auth.json \
  http://localhost:9130/_/proxy/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/modules/test-auth-3.4.1
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/proxy/modules : 201 1614us
Content-Length: 377

{
  "id" : "test-auth-3.4.1",
  "name" : "Okapi test auth module",
  "requires" : [ ],
  "provides" : [ {
    "id" : "test-auth",
    "version" : "3.4",
    "handlers" : [ {
      "methods" : [ "POST" ],
      "pathPattern" : "/authn/login"
    } ]
  } ],
  "filters" : [ {
    "methods" : [ "*" ],
    "pathPattern" : "/*",
    "phase" : "auth",
    "type" : "headers"
  } ]
}
```

接下来我们就需要部署模块了。因为我们并没有在moduleDescriptor中放入launchDescriptor，所以在这里还需要提供一个launchDescriptor

```
cat > /tmp/okapi-deploy-test-auth.json <<END
{
  "srvcId": "test-auth-3.4.1",
  "nodeId": "localhost",
  "descriptor": {
    "exec": "java -Dport=%p -jar okapi-test-auth-module/target/okapi-test-auth-module-fat.jar"
  }
}
END

curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-test-auth.json \
  http://localhost:9130/_/discovery/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/discovery/modules/test-auth-3.4.1/localhost-9132
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/discovery/modules : 201
Content-Length: 246

{
  "instId" : "localhost-9132",
  "srvcId" : "test-auth-3.4.1",
  "nodeId" : "localhost",
  "url" : "http://localhost:9132",
  "descriptor" : {
    "exec" : "java -Dport=%p -jar okapi-test-auth-module/target/okapi-test-auth-module-fat.jar"
  }
}
```


我们为租户启用模块：

```
cat > /tmp/okapi-enable-auth.json <<END
{
  "id": "test-auth-3.4.1"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-auth.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib/modules/test-auth-3.4.1
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/proxy/tenants/testlib/modules : 201 1693us
Content-Length: 30

{
  "id" : "test-auth-3.4.1"
}
```

这样`test-auth`模块应该会拦截我们所有对Okapi的调用请求，并检查是否授权使用它。我们尝试像之前调用`basic-module`模块一样：

```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  http://localhost:9130/testb

HTTP/1.1 401 Unauthorized
Content-Type: text/plain
X-Okapi-Trace: GET test-auth-3.4.1 http://localhost:9132/testb : 401 64187us
Transfer-Encoding: chunked

test-auth: check called without X-Okapi-Token
```

实际上，我们不再被允许调用测试模块。那么我们如何获得许可呢？依照错误消息可以发现我们需要一个`X-Okapi-Token`的消息头。我们可以从登陆服务中获得。测试模块使用用户名"peter"、密码"peter-password"。虽然这样并不是很安全，但是演示足够了。

```
cat > /tmp/okapi-login.json <<END
{
  "tenant": "testlib",
  "username": "peter",
  "password": "peter-password"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -H "X-Okapi-Tenant: testlib" \
  -d @/tmp/okapi-login.json \
  http://localhost:9130/authn/login

HTTP/1.1 200 OK
X-Okapi-Trace: POST test-auth-3.4.1 http://localhost:9132/authn/login : 202 4539us
Content-Type: application/json
X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig
X-Okapi-Trace: POST test-auth-3.4.1 http://localhost:9132/authn/login : 200 159934us
Transfer-Encoding: chunked

{  "tenant": "testlib",  "username": "peter",  "password": "peter-password"}
```

响应只是回显它的参数，但是请注意，我们返回了一个HTTP头`X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig`。我们不 用担心这个头包含什么，但是我们可以看到它的格式几乎与您从JWT中所期望的一样：三个由`".""`分隔的部分，首先是一个头，其次为base-64编码的有效值,最后为一个签名。头和前面通常也使用base-64进行编码。但是`test-auth`跳过了这一部分，并且生成了一个不会被误认为是JWT的独特标记。有效值实际上也是base-64编码，如果对其解码会得到一个包含用户ID和租户ID的JSON数据。在一个真实的auth模块中会在JWT中放入更多的东西，并对签名进行加密。这对于Okapi的用户而言并没有什么影响，因为它只需要知道如何获得令牌，以及在每一个请求中加入令牌。
```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  http://localhost:9130/testb

HTTP/1.1 200 OK
X-Okapi-Trace: GET test-auth-3.4.1 http://localhost:9132/testb : 202 15614us
Content-Type: text/plain
X-Okapi-Trace: GET test-basic-1.0.0 http://localhost:9131/testb : 200 1826us
Transfer-Encoding: chunked

It works
```

我们也可以发送POST请求给测试模块：
```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  -d '{ "foo":"bar"}' \
  http://localhost:9130/testb
```

模块以相同的JSON进行响应，但是在其之前加上了"Hello"字样。

### 示例 3: 升级、版本、环境以及`_tenant`接口

升级是经常遇到的问题,Okapi亦是如此。由于我们需要为很多租户提供服务，他们对于何时升级以及升级的内容有不同的需求。在这个示例中，我们将演示升级过程、版本讨论、环境变量以及查看特殊的`_tenant`系统接口。

这里我们有一个新的和改进的示例模块:
```
cat > /tmp/okapi-proxy-test-basic.2.json <<END
{
  "id": "test-basic-1.2.0",
  "name": "Okapi test module, improved",
  "provides": [
    {
      "id": "test-basic",
      "version": "2.4",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    },
    {
      "id": "_tenant",
      "version": "1.0",
      "interfaceType": "system",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/_/tenant"
        }
      ]
    }
  ],
  "requires": [
    {
      "id": "test-auth",
      "version": "3.1"
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar",
    "env": [
      {
        "name": "helloGreeting",
        "value": "Hi there"
      }
    ]
  }
}
END
```

注意，我们给了模块一个不同的ID，虽然模块名一样，但是有着更高的版本号。对于这个示例由于我们没有太多的东西可以使用，所以我们依旧使用相同的模块`okapi-test-module`。
在真实情况下我们只修改模块的描述，那么就和这里一样。

我们添加了一个新的支持接口`_tenant`。当租户启用时，Okapi将自动调用此接口。其目的在于执行模块所需的所有初始化操作，如创建数据库表。

我们还指定模块至少在3.1版本中会用到`test-auth`接口。我们在前面的示例中安装的auth模块提供了3.4。（若需要3.5或4.0，甚至2.0都不行，请参阅[版本控制和依赖关系](#版本控制和依赖关系)或编辑描述符并尝试发布它)

最后我们在 launchDescriptor 中添加了一个环境变量。当服务处理POST请求时，模块应该返回它。

任何模块一样，升级过程首先发布新的ModuleDescriptor。我们不能修改旧的版本，因为可能其他的租户还在使用它。
```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-test-basic.2.json \
  http://localhost:9130/_/proxy/modules

HTTP/1.1 201 Created   ...
```

接下来，我们像以前一样部署模块。

```
cat > /tmp/okapi-deploy-test-basic.2.json <<END
{
  "srvcId": "test-basic-1.2.0",
  "nodeId": "localhost"
}
END

curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-test-basic.2.json  \
  http://localhost:9130/_/discovery/modules
```

现在我们已经安装并运行了两个模块。我们的租户依旧再是用旧的那个。我们通过启用新的来做切换。这步骤是通过发送POST请求来完成的。

```
cat > /tmp/okapi-enable-basic-2.json <<END
{
  "id": "test-basic-1.2.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-basic-2.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules/test-basic-1.0.0

HTTP/1.1 201 Created
Content-Type: application/json
Location: /_/proxy/tenants/testlib/modules/test-basic-1.2.0
X-Okapi-Trace: POST okapi-2.0.1-SNAPSHOT /_/proxy/tenants/testlib/modules/test-basic-1.0.0 : 201
Content-Length: 31

{
  "id" : "test-basic-1.2.0"
}
```

现在在我们为租户启用了新的模块，而不是旧的，如下所示:
```
curl -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules
```

如果你查看Okapi的日志，你会看到这样一行:
```
15:32:40 INFO  MainVerticle         POST request to okapi-test-module tenant service for tenant testlib
```

它表明我们的测试模块确实收到了对租户接口的请求。

为了验证我们真的在使用新的模块，让我们发布一个东西:
```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  -d '{ "foo":"bar"}' \
  http://localhost:9130/testb

HTTP/1.1 200 OK
X-Okapi-Trace: POST test-auth-3.4.1 http://localhost:9132/testb : 202 2784us
Content-Type: text/plain
X-Okapi-Trace: POST test-basic-1.2.0 http://localhost:9133/testb : 200 3239us
Transfer-Encoding: chunked

Hi there { "foo":"bar"}
```

事实上，我们看到的是“Hi there”而不是“Hello”，`X-Okapi-Trace`显示请求被发送到模块的改进版本。

### 示例 4: 完整的 ModuleDescriptor

在本例中，我们会展示一个完整的ModuleDescriptor，其中包含了所有的附加功能。到目前为止，你应该知道如何使用，所以没有重复的`curl`命令

```javascript
{
  "id": "test-basic-1.3.0",
  "name": "Bells and Whistles",
  "provides": [
    {
      "id": "test-basic",
      "version": "2.4",
      "handlers": [
        {
          "methods": [ "GET" ],
          "pathPattern": "/testb",
          "permissionsRequired": [ "test-basic.get.list" ]
        },
        {
          "methods": [ "GET" ],
          "pathPattern": "/testb/{id}",
          "permissionsRequired": [ "test-basic.get.details" ],
          "permissionsDesired": [ "test-basic.get.sensitive.details" ],
          "modulePermissions": [ "config.lookup" ]
        },
        {
          "methods": [ "POST", "PUT" ],
          "pathPattern": "/testb",
          "permissionsRequired": [ "test-basic.update" ],
          "modulePermissions": [ "config.lookup" ]
        }
      ]
    },
    {
      "id": "_tenant",
      "version": "1.0",
      "interfaceType": "system",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/_/tenant"
        }
      ]
    },
    {
      "id": "_tenantPermissions",
      "version": "1.0",
      "interfaceType": "system",
      "handlers": [
        {
          "methods": [ "POST" ],
          "pathPattern": "/_/tenantpermissions"
        }
      ]
    }
  ],
  "requires": [
    {
      "id": "test-auth",
      "version": "3.1"
    }
  ],
  "permissionSets": [
    {
      "permissionName": "test-basic.get.list",
      "displayName": "test-basic list records",
      "description": "Get a list of records"
    },
    {
      "permissionName": "test-basic.get.details",
      "displayName": "test-basic get record",
      "description": "Get a record, except sensitive stuff"
    },
    {
      "permissionName": "test-basic.get.sensitive.details",
      "displayName": "test-basic get whole record",
      "description": "Get a record, including all sensitive stuff"
    },
    {
      "permissionName": "test-basic.update",
      "displayName": "test-basic update record",
      "description": "Update or create a record, including all sensitive stuff"
    },
    {
      "permissionName": "test-basic.view",
      "displayName": "test-basic list and view records",
      "description": "See everything, except the sensitive stuff",
      "subPermissions": [
        "test-basic.get.list",
        "test-basic.get.details"
      ]
    },
    {
      "permissionName": "test-basic.modify",
      "displayName": "test-basic modify data",
      "description": "See, Update or create a record, including sensitive stuff",
      "subPermissions": [
        "test-basic.view",
        "test-basic.update",
        " test-basic.get.sensitive.details"
      ]
    }
  ],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar",
    "env": [
      {
        "name": "helloGreeting",
        "value": "Hi there"
      }
    ]
  }
}
```

此时大多数的描述内容都很熟悉。最大的问题是权限。关于权限的 完整介绍请查看单独的文档[权限系统](#https://github.com/folio-org/okapi/blob/master/doc/security.md)，它有`auth-moudle`统一管理。以上这些都超出了Okapi自身和本指南的范围。

在描述文件中最显眼的是permissionSets部分。这里定义了这个模块所关心的所有权限。
要指出的是"Permissions"或者 "Permission Bits" 就像是"test-basic.get.list"这样子的简单字符串。Okapi和auth模块在这个粒度级别上运行。"Permission Sets"是命名集，可以包含多个权限，甚至还可以包含其他权限集的权限。
这些将被auth模块简化为实际权限。这些是管理员用户通常看到的最低级别的权限，并由此构成构建更复杂角色的构建块。

权限被用户处理程序的入口。第一个有"permissionsRequired"字段，其中包含权限“test-basic.get.list”。这意味着如果用户没有这样的权限，auth模块将告诉Okapi,Okapi将拒绝请求。

下一个依旧有"permissionsRequired"字段，同时也有"permissionsDesired"字段，其中包含"test-basic.get.sensitive.details"。这表明模块希望用户拥有这样的权限，这不是一个硬性要求，在任何情况下请求都会被传递给该模块，但是有一个`X-Okapi-Permissions`消息头来表示否有该权限。有模块自行决定如何处理。这种情况下，用于决定显示/隐藏某些敏感字段。

在第三个可以看到"modulePermissions"字段，取值为"config.lookup"。这说明模块已经被授予了这个权限。即使用户没有这样的权限，模块也拥有这样的权限。

可见，Okapi采用细粒度权限控制。它将所需的和期望的权限传递给auth模块，auth模块会以某种方式判定用户是否拥有这些权限。在此不会讨论具体细节，但可以确定的是权限集合和处理有关系。至于auth模块如何访问moduleDescription中的权限集？这不是凭空的。当为租户启用模块时，Okapi不仅调用模块本身的"_tenant"接口，而且查看是否有模块提供tenantPermissions接口，并将permissionset传递给那里。权限模块应该这样做，并以这种方式接收权限集。

这个例子并没有涵盖ModuleDescriptor中的所涉及的所有情况。仍然支持老版本(比Okapi 1.2.0更老的版本)遗留的一些特性，但是在将来的版本中会被弃用和删除（例如，ModulePermissions和RoutingEntries）。对于完全最新的定义，请[参考](#web-service)部分中的RAML和JSON模式。

### 多接口

通常，Okapi只允许一次代理一个模块提供的接口。通过在`provides`中使用`interfaceType`和`multiple`字段，Okapi允许任意数量的模块实现相同的接口。导致的结果就是用户必须通过指定HTTP头`X-Okapi-Module-Id`来选择调用哪个模块。Okapi提供了一个工具
为租户提供给定接口的模块列表(`_/proxy/tenants/{tenant}/modules?provide={interface}`)。通常，租户将与“当前”租户相同(`X-Okapi-Tenant`)。

让我们通过一个例子来说明这个问题。我们将定义实现相同接口的两个模块，并调用其中一个模块。假设前面示例中的租户testlib和auth模块仍然存在。让我们为前面使用的测试模块定义一个ModuleDescriptor。在ModuleDescriptor中将`interfaceType`设置为`multiple`。因此Okapi允许多个接口`test-multi`模块共存。

```
cat > /tmp/okapi-proxy-foo.json <<END
{
  "id": "test-foo-1.0.0",
  "name": "Okapi module foo",
  "provides": [
    {
      "id": "test-multi",
      "interfaceType": "multiple",
      "version": "2.2",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    }
  ],
  "requires": [],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
END
```

注册和部署`foo`模块:

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-foo.json \
  http://localhost:9130/_/proxy/modules
```

```
cat > /tmp/okapi-deploy-foo.json <<END
{
  "srvcId": "test-foo-1.0.0",
  "nodeId": "localhost"
}
END
```

```
curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-foo.json \
  http://localhost:9130/_/discovery/modules
```

现在我们定义另外一个模块 `bar`：

```
cat > /tmp/okapi-proxy-bar.json <<END
{
  "id": "test-bar-1.0.0",
  "name": "Okapi module bar",
  "provides": [
    {
      "id": "test-multi",
      "interfaceType": "multiple",
      "version": "2.2",
      "handlers": [
        {
          "methods": [ "GET", "POST" ],
          "pathPattern": "/testb"
        }
      ]
    }
  ],
  "requires": [],
  "launchDescriptor": {
    "exec": "java -Dport=%p -jar okapi-test-module/target/okapi-test-module-fat.jar"
  }
}
END
```

注册和部署`bar`模块：

```
curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-proxy-bar.json \
  http://localhost:9130/_/proxy/modules

```

```
cat > /tmp/okapi-deploy-bar.json <<END
{
  "srvcId": "test-bar-1.0.0",
  "nodeId": "localhost"
}
END
```
```
curl -w '\n' -D - -s \
  -X POST \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-deploy-bar.json \
  http://localhost:9130/_/discovery/modules
```

现在为租户`testlib`启用`foo`和`bar`模块：

```
cat > /tmp/okapi-enable-foo.json <<END
{
  "id": "test-foo-1.0.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-foo.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules
```

```
cat > /tmp/okapi-enable-bar.json <<END
{
  "id": "test-bar-1.0.0"
}
END

curl -w '\n' -X POST -D - \
  -H "Content-type: application/json" \
  -d @/tmp/okapi-enable-bar.json \
  http://localhost:9130/_/proxy/tenants/testlib/modules
```

我们可以向Okapi查询哪个模块实现了`test-multi`接口：

```
curl -w '\n' \
  http://localhost:9130/_/proxy/tenants/testlib/modules?provide=test-multi

[ {
  "id" : "test-bar-1.0.0"
}, {
  "id" : "test-foo-1.0.0"
} ]
```

让我们调用`bar`模块：

```
curl -D - -w '\n' \
  -H "X-Okapi-Tenant: testlib" \
  -H "X-Okapi-Token: dummyJwt.eyJzdWIiOiJwZXRlciIsInRlbmFudCI6InRlc3RsaWIifQ==.sig" \
  -H "X-Okapi-Module-Id: test-bar-1.0.0" \
  http://localhost:9130/testb

It works
```

在Okapi版本2.8.0和更高版本提供了一种方法来查询租户的所有接口`_/proxy/tenants/{tenant}/interfaces`。可以通过`full`和`type`进行调优。`full`参数是一个boolean类型。如果为`true`就返回所有接口。在`full`模式下，如interfaceType=multiple或system之类的接口可能会重复出现。如果`full`的取值为`false`每个接口只返回一次。如果给定`tpye`参数，则会返回限`type`指定类型的接口。如果`type`没有指定则返回有所类型的接口。

### 移除

例子讲完了。友好起见，我们删除了所有已安装的东西:
```
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-basic-1.2.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-auth-3.4.1
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-foo-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib/modules/test-bar-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/tenants/testlib
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-auth-3.4.1/localhost-9132
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-basic-1.0.0/localhost-9131
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-basic-1.2.0/localhost-9133
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-foo-1.0.0/localhost-9134
curl -X DELETE -D - -w '\n' http://localhost:9130/_/discovery/modules/test-bar-1.0.0/localhost-9135
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-basic-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-basic-1.2.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-foo-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-bar-1.0.0
curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/test-auth-3.4.1
```
Okapi的响应很简单：
```
HTTP/1.1 204 No Content
Content-Type: application/json
X-Okapi-Trace: DELETE ...
Content-Length: 0
```
<!-- STOP-EXAMPLE-RUN -->
最后我们通过`Ctrl-C`来终止Okapi。

Finally we can stop the Okapi instance we had running, with a simple `Ctrl-C`
command.


### 集群模式

到目前为止，所有示例都在一台机器上以`dev`模式运行。这对于演示、开发等等非常有用，但是在实际的生产环境中，我们需要在集群上运行。

#### 单机部署

最简单的集群是在一台机器上部署多个Okapi实例。虽然不推荐这么做，但是对于演示却非常有用。

打开控制台并启动你的第一个Okapi
```
java -jar okapi-core/target/okapi-core-fat.jar cluster
```

Okapi会在`dev`模式下打印跟多的信息。值得关注的一个信息是：
```
Hazelcast 3.6.3 (20160527 - 08b28c3) starting at Address[172.17.42.1]:5701
```

它意味着我们正在使用Hazelcast——Hazelcast是vert.x用于部署集群的工具,并监控IP地址为172.17.42.1的5701端口。虽然端口是默认的，但Hazelast会找到一个空闲端口，所以你可能会看到其他的端口。地址为你本机的IP地址。至于为什么会这样，稍后会做具体解释。

打开另一个控制台并启动另一个Okapi实例。由于是在同一台机器上，两个实例会监听不同的端口。默认情况下Okapi会为模块分配20个端口，所以我们在9150上运行下一个实例Okapi：
```
java -Dport=9150 -jar okapi-core/target/okapi-core-fat.jar cluster
```

Okapi会再次打印一些运行信息，但是请注意，第一个Okapi也会打印一些信息。这表明两个Okapi在相互通信。

现在你可以在第三个控制台窗口试着让Okapi列出已知的节点：
```curl -w '\n' -D - http://localhost:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-2.0.1-SNAPSHOT /_/discovery/nodes : 200
Content-Length: 186

[ {
  "nodeId" : "0d3f8e19-84e3-43e7-8552-fc151cf5abfc",
  "url" : "http://localhost:9150"
}, {
  "nodeId" : "6f8053e1-bc55-48b4-87ef-932ad370081b",
  "url" : "http://localhost:9130"
} ]
```

实际上，它列出了两个节点。每一个节点都有一个URL和一个nodeId(随机的UUID字符串)

您可以通过将URL中的端口更改为9150来获取其他节点列出的信息，并且应该得到相同的列表，可能顺序不同。

#### 部署在独立的机器上

当然，您也可以在多台机器上部署集群。

*注意* Okapi使用Hazelcast库来管理集群设置，并使用广播的方式来发现集群。广播方式在有线网上运行良好，但在无线网络环境下有些欠缺。当然任何人都都不应该将集群部署在无线网环境下，但开发人员可能希望使用无线网在笔记本上做试验。这样是行不通的！这里有一个涉及hazelcast-config-file的解决方案，用以列出集群中所有的IP地址，但是文件内容有点乱，我们在这里不做详细讨论。

部署的流程出来两个小细节外，和之前几乎是一样的。首先，因为Okapi位于不同的机器上，不会发生端口冲突，所以不需要指定不同的端口。相反，你需要确保两台机器在同一个网络中。由于Linux系统支持多个网络接口，通常有有线网接口和回环接口，也有Wifi端口。如果你使用Docker，它还会建立内部网络。可以通过`sudo  ifconfig`命令来列出你所看到的所有接口。Okapi并不能够很智能的猜测出需要使用那个接口，所以你需要如下命令来指定：
```
java -jar okapi-core/target/okapi-core-fat.jar cluster -cluster-host 10.0.0.2
```

注意，集群主机地址必须位于命令行末端，参数名` -cluster-host`可能会误导，使用的的主机地址而不是主机名。


在相同网络中的另一个机器上启动Okapi。注意，要在命令行中使用正确的IP地址。如果一切顺利，两台机器能够相互通信。你可以在两台机器的日志中看到日志信息。现在你可以在集群中的任何一个Okapi中列出所有的节点：
```curl -w '\n' -D - http://localhost:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-2.0.1-SNAPSHOT /_/discovery/nodes : 200
Content-Length: 186

[ {
  "nodeId" : "81d1d7ca-8ff1-47a0-84af-78cfe1d05ec2",
  "url" : "http://localhost:9130"
}, {
  "nodeId" : "ec08b65d-f7b1-4e78-925b-0da18af49029",
  "url" : "http://localhost:9130"
} ]
```

注意，不同节点有不同的UUID，但有相同的URL。它们都可以通过`http://localhost:9130`来访问。理论上讲，如果在节点本身上使用curl，则localhost:9130会指向Okapi。但是如果你希望在网络上的其它地方与节点通信，那这个方法就行不通了。你需要在命令行中添加另一个参数以告诉主机返回具体哪个Okapi。

关闭所有的Okapi，并并再一次用如下命令启动：
```
java -Dhost=tapas -jar okapi-core/target/okapi-core-fat.jar cluster -cluster-host 10.0.0.2
```

用正确的机器名或者IP地址来替代"tapas"，再一次列出所有节点：

```curl -w '\n' -D - http://localhost:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 178

[ {
  "nodeId" : "40be7787-657a-47e4-bdbf-2582a83b172a",
  "url" : "http://jamon:9130"
}, {
  "nodeId" : "953b7a2a-94e9-4770-bdc0-d0b163861e6a",
  "url" : "http://tapas:9130"
} ]
```

你可以通过URL来验证：

```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 178

[ {
  "nodeId" : "40be7787-657a-47e4-bdbf-2582a83b172a",
  "url" : "http://jamon:9130"
}, {
  "nodeId" : "953b7a2a-94e9-4770-bdc0-d0b163861e6a",
  "url" : "http://tapas:9130"
} ]
```

#### 给节点命名

如上所述，Hazelcast会默认为系统中的节点分配UUID作为唯一标识。这没什么问题，但是不够人性化，而且每次运行的时候UUID都在变化，这使得在脚本中引用节点有点困难。所以我们添加了一个功能，让节点在命令行上运行的时候可以指定一个节点名。如下所示：
```
java -Dhost=tapas -Dnodename=MyFirstNode \
  -jar okapi-core/target/okapi-core-fat.jar cluster -cluster-host 10.0.0.2
```

这时如果你查看所有节点你会看到：
```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes```

```
HTTP/1.1 200 OK
Content-Type: application/json
X-Okapi-Trace: GET okapi-2.0.1-SNAPSHOT /_/discovery/nodes : 200
Content-Length: 120

[ {
  "nodeId" : "7d6dc0e7-c163-4bbd-ab48-a5d7fa6c4ce4",
  "url" : "http://tapas:9130",
  "nodeName" : "MyFirstNode"
} ]
```

这样你就可以在许多地方使用名称而不是节点标识，例如：
```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes/myFirstNode```


You can use the name instead of the nodeId in many places, for example
```curl -w '\n' -D -    http://tapas:9130/_/discovery/nodes/myFirstNode```

#### 这样你已经拥有一个集群了~

Okapi集群的工作模式你与之前看到的单机工作模式非常相似。在大多数情况下，与哪个节点通信并不重要，它们共享所有信息。你可以通过一个节点新建模块，通过另一个节点创建租户，并再次通过第一个节点为该租户启用模块。在部署模块时你必须更加小心，因为你需要控制模块在哪个节点上运行。以上的节点示例仅使用"localhost"作为部署节点的id，在集群上你则需要用你见过多次的UUID来指定节点。在部署完模块后你可以使用任何一个你希望的Okapi来代理模块的流量。

这里有一些差异你应该要知道：

* 集群中的所有节点共享内存，这意味着如果你关闭再启动一个节点，它会自动与另外一个节点同步，并且仍然能够感知共享数据。知道当所有的Okapi都关闭了，数据才会从内存中清楚。当然，你也可以用Postgres来做持久化。

* 你可以通过`/_/deployment`把模块部署到节点上。这必须在你期望的节点上执行，Okapi会将其信息通知到其他的节点。所以通常应该通过`/_/discovery`来进行部署，并指定相应的节点ID。

* 运行Okapi可能会需要稍长一点的时间。

你还可以使用另外两种集群模式。`deployment`以集群模式运行Okapi，但是不进行代理。这在只有一个节点且外部可见的集群中非常有用，其他的节点可以在`deployment`模式下运行。

另外一种是`proxy`模式，恰好相反，它用于接收Okapi所有的请求，并将其转发给"deployment"节点，这通常是一个外部可见的节点。当大流量时，仅通过代理就可以完全占用一个节点。

### Securing Okapi

在上述例子中，我们只是向Okapi发送命令让Okapi为我们部署和启用模块，但并没有做任何检查。在生产系统中，这是不可接受的。Okapi被设计成抑郁保护的。实际上，有几个小技巧可以在不进行检查的情况下使用Okapi。例如，如果没有指定超级租户并启用了内部模块，Okapi默认为提供`okapi.supertenant`.

原则上，保护Okapi自身安全的方法与对任何模块访问的方法相同：为`okapi.supertenant`安装一个auth校验过滤器，它会阻止非授权用户。auth示例模块在这方面过于简单——在实际情况下会希望系统为不同的用户处理不同的权限等。

保障Okapi的具体细节取决于使用auth模块的性质。在任何情况下，我们必须小心，不要把自己锁在外面，当设置好一切便可以再次进入。大致如下：
* 当Okapi启动时，它创建内部模块supertenant，并为承租者启用该模块。所有操作都不需要任何检查。
* 管理员安装和部署必要的auth模块。
* 管理员为auth模块启用存储端。
* 管理员将适当的凭证和权限发布到auth模块。
* 管理员启用了auth-check过滤器，现在什么都被拒绝。
* 管理员使用之前加载的凭据登录，并获得一个令牌。
* 令牌赋予管理员在Okapi中执行进一步操作的权利。

ModuleDescriptor提供为细粒度访问控制定义合适权限的功能。目前，大多数只读操作对任何人都是开放的，但是修改操作是需要获得许可。

如果普通的客户端想要使用Okapi的管理功能，如列出他们有哪些模块可以使用，则需要为他们提供内部模块，如果需要，还需要为其分配权限。

在[securing an Okapi installation](securing.md)中有更详细的介绍。

### 共享模块描述
Okapi的安装是通过共享模块描述来完成的。Okapi可以通过"pull"的方式从另一个Okapi代理实力中获取模块的信息。这个"pull"操作类似于Git SCM的pull操作。模块所在的Okapi的代理能够以"pull"的方式从另外一个Okapi代理示例中获得。Okapi从远程代理实力（或对等的节点）获取模块信息而不需要安装模块。所需要的只是`/_/proxy/modules`操作。"pull"从远程安装所有已不可用的模块描述。它基于表示模块唯一ID的模块描述。

对于"pull"操作，Okapi提供一个Pull Descriptor。在这阶段，它包括远程实例的URL。在Okapi未来的版本中可能会有用于身份验证和其他目的的Pull描述信息。为本地Okapi提供调用的路劲为`/_/proxy/pull/modules`。

"pull"操作返回在本地获取和安装的模块。

#### Pull操作示例

在这个例子中，我们有两次Pull操作。第二次Pull操作要比第一次Pull操作快，因为所有或者大多数模块已经被获取了。
```
cat > /tmp/pull.json <<END
{"urls" : [ "http://folio-registry.aws.indexdata.com:80" ]}
END

curl -w '\n' -X POST -d@/tmp/pull.json http://localhost:9130/_/proxy/pull/modules
curl -w '\n' -X POST -d@/tmp/pull.json http://localhost:9130/_/proxy/pull/modules
```

### 为每个租户安装模块

到目前为止，在本指南中，我们一次只安装了几个模块，并且我们能够跟踪依赖关系并确保它们是有序的。例如`test-basic`所需的`test-auth`接口由`test-auth`模块提供。这两个名字很巧合，更不用说一个模块可能需要许多接口了。

Okapi 1.10及之后的版本提供`/_/proxy/tenants/id/install`来纠正这种情况。此调用接受要启用/更新/禁用的一个或多个模块，并返回一个类似的列表，该列表依照依赖关系。有关详细的信息，请参数TenantModuleDescriptorList和RAML的定义。

#### 安装操作示例

假设我们已经从远程仓库获取了模块的描述信息(如之前的[Pull操作示例](#pull操作示例))
并且希望为我们的租户启用`mod-users-bl-2.0.1`。

```
cat > /tmp/okapi-tenant.json <<END
{
  "id": "testlib",
  "name": "Test Library",
  "description": "Our Own Test Library"
}
END
curl -w '\n' -X POST -D - \
  -d @/tmp/okapi-tenant.json \
  http://localhost:9130/_/proxy/tenants

cat >/tmp/tmdl.json <<END
[ { "id" : "mod-users-bl-2.0.1" , "action" : "enable" } ]
END
curl -w '\n' -X POST -d@/tmp/tmdl.json \
 http://localhost:9130/_/proxy/tenants/testlib/install?simulate=true

[ {
  "id" : "mod-users-14.2.1-SNAPSHOT.299",
  "action" : "enable"
}, {
  "id" : "permissions-module-4.0.4",
  "action" : "enable"
}, {
  "id" : "mod-login-3.1.1-SNAPSHOT.42",
  "action" : "enable"
}, {
  "id" : "mod-users-bl-2.0.1",
  "action" : "enable"
} ]
```

共需要一组4个模块。当然，这个列表可能会根据远程存储库中的当前模块集有所变更。

Okapi 1.11.0及以后的版本可以在没有版本号的情况下启用。在上面的例子中，我们可以使用 `mod-users-bl`。在这种情况下，最新的可用模块的action=enable，已安装的模块的anction=disable。Okapi总是使用完整的结果模块id进行响应

默认情况下，无论是否预发布，所有的模块都会被安装。对于Okapi 1.11.0可以添加`preRelease`过滤器来接受一个boolean值，如果是false，将只安装没有预发布的模块。

如果模块提供的`_tenant`接口版本为1.2或更高版本，则Okapi 2.20.0及更高版本允许在为租户启用或升级时将参数`tenantParameters`传递给模块。
`tenantParameters`是一个字符串，由键值对(用逗号分隔)和值(用等号(=)分隔)组成。对于URI来说，这是一个单独的参数，所以一定要将逗号编码为%2C、等号编码为%3D。有关更多信息，请参见[租户接口](#租户接口)。

### 为每个租户升级模块

升级工具由一个无请求实体的POST请求和一个与安装工具类似的响应组成。调用的路径是`/_/proxy/tenants/id/upgrade`。与安装工具一样，有一个可选的模拟参数(boolean类型)，如果为true，则模拟升级。此外`preRelease`参数也会被识别出来用以控制是否应该包含预发布新的的模块ID。

升级工具在Okapi 1.11.0及以后版本中中包含

### 自动部署

从Okapi 2.3.0开始，安装和升级操作有一个可选参数`deploy`，为boolean类型。如果为true，安装操作将根据需要部署和取消部署。这仅当ModuleDescriptor包含launchDescriptor时生效。

### 清除持久化数据

默认情况下，当模块被禁用时，持久化数据将被保留。可以通过可选参数`purge`来删除，将其设置为`true`,表示清楚所有持久化数据。这同样只对禁用的模块生效，对已启用的模块没有任何影响。该参数是在Okapi 2.16.0中添加的。如果模块提供了DELETEF方法，则清除时将会调用带有DELETE方法的`_tenant`接口。

## 参考

### Okapi

Okapi是作为jar文件(okapi-core-fat.jar)提供的。通常的调用方法如下：
  `java` [*java-options*] `-jar path/okapi-core-fat.jar` *command* [*options*]
  
这是一个标准的java命令行。值得注意的是，java-option `-D`参数可以为程序提参数(详见下一节)。Okapi自身会解析*command*和*option*后的选项参数。

#### Java -D options

`-D`参数可用于指定Okapi中的运行参数。这些必须在命令行的开头，在`-jar`之前。

* `port`：Okapi的监听端口，默认为9130
* `port_start`和`port_end`：模块的端口范围。默认为从`port`+1到`port`+10，通常是9131到9141。
* `host`：在部署服务返回的URL中使用的主机名。默认为`localhost`。
* `nodename`：节点实例的名称。用以替代在集群模式下系统生成的UUID或者开发模式下的·`localhost`。
* `storage`：定义后端存储方式，取值为`postgres`,`inmemory`。默认为`inmemory`。
* `lang`：Okapi返回消息的默认语种。
* `loglevel`：日志级别。，默认为`INFO`。还可以有`DEBUG`,`TRACE`,`WARN`和`ERROR` 。
* `okapiurl`：告诉Okapi其自身的URL。它作为消息头X-Okapi-Url传递给模块，模块可以使用它进一步Okapi发送消息。默认为`http://localhost:9130`或其他指定的端口。后面不应该带有"/"，如果有的话Okapi会将其删除。注意，它有可能类似于`https://folio.example.com/okapi`。
* `dockerUrl`：告诉Okapi部署Docker Daemon的位置。默认是`http://localhost:4243`。
* `postgres_host`：PostgresSQl地址，默认为`localhost`。
* `postgres_port`：PostgresSQl端口。默认为`5432`
* `postgres_username` : PostgreSQL 用户名. 默认为`okapi`.
* `postgres_password`: PostgreSQL 密码. 默认为 `okapi25`.
* `postgres_database`: PostgreSQL 数据库. 默认为 `okapi`.
* `postgres_db_init`：初始化数据库。如果值为`1`，Okapi将删除现有的的数据库并创建一个新的。如果为`0`(默认值)将不做初始化。

#### 相关命令

Okapi在运行时需要给出确定的命令。具体如下：
* `cluster` 以集群模式运行
* `dev` 开发模式运行，单节点
* `deployment` 仅部署。集群模式有效
* `proxy` 代理和发现。集群模式有效
* `help` 列出命令行可选项和命令。
* `initdatabase` 如果存在则删除现有数据，并初始化数据库
* `purgedatabase` 删除现有表和数据。

#### 命令行选项

命令行可选下具体如下：
* `-hazelcast-config-cp` _file_ -- 从class path中读取配置
* `-hazelcast-config-file` _file_ -- 从本地文件中读取配置
* `-hazelcast-config-url` _url_ -- 从URL中读取配置
* `-enable-metrics` -- 支持各种指标发送到后端
* `-cluster-host` _ip_ -- Vert.x集群主机
* `-cluster-port` _port_ -- Vert.x集群端口

### 环境变量

Okapi提供了一个概念:环境变量。它是系统范围内的环境变量，提供了再部署期间向模块传递信息的方式。如，访问数据库模块时需要知道连接的具体细节。

在部署时，为要部署的流程定义环境变量。注意，这些只能传递给Okapi管理的模块，如Docker实例和进程。但不是由URL定义的远程服务(URL无论如何都不会部署)。

除了部署模式，Okapi在`/_/env`下还提供了CRU(create,read,update)服务，服务的标识符是变量的名称。不能包含"/"，也不能以"_"开头。环境变量实体由[`EnvEntry.json`](../okapi-core/src/main/raml/EnvEntry.json)提供。

### Web Service

Okapi请求服务 (所有的请求以 `/_/`开头) 在 [RAML](http://raml.org/) 中有语法定义。
* top-level文件，[okapi.raml](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml/okapi.raml)
* [RAML目录和包含JSON的Schema文件](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml)
*相关文件由[API相关文档](https://dev.folio.org/reference/api/)生成
 
### 内部模块

当Okapi启动时，它已定义了一个内部模块。该模块提供两个接口：`okapi`和`okapi-proxy`。"okapi"接口涵盖所有RAML中定义的所有管理功能(参见上面)。`okapi-proxy`接口是负责代理相关功能。由于该功能取决于模块提供了什么，所以不能再RAML中定义。它的主要用途是模块可以依赖于它，特别是需要一些新的代理功能时。预计随着时间的推移，这个接口将保持稳定。

内部模块在Okapi 1.9.0中提空，并且完整的ModuleDescriptor在1.10.0版本中。

### 部署

部署是由[DeploymentDescriptor.json](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml/DeploymentDescriptor.json)
and [LaunchDescriptor.json](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml/LaunchDescriptor.json)j决定的。LaunchDescriptor 可以是ModuleDescriptor的一部分，也可以在DeploymentDescriptor中指定。

以下方法用于启动模块：

* Process:`exec`用于制定一个进程。该进程保持激活状态并且能够被Okapi提供过信号杀死。

* Commands：如果`cmdlineStart`和`cmdlineStop`存在则触发。`cmdlineStart`是一个生成后台服务的shell脚本。`cmdlineStop`是一个终止相应服务的shell脚本。

* Docker：`dockerImage`用于制定一个已存在的Docer镜像。Okapi管理一个基于镜像的容器。这个参数需要一个`dockerUrl`参数用以指向通HTTP访问的Docker Daemon。默认情况下，Okapi会在启动镜像前尝试获取镜像。这可以通过一个boolean类型的参数`dockerPull`来控制，可以将其设置为false用以防止pull操作。Dockerfile的`CMD`命令可以通过`dockerCMD`来控制。假设，`ENTRYPOINT`是模块的完整调用并且`CMD`是默认值，或者最好是空的。最后，`dockerArgs`可以用于传递给Docker SDK用以创建容器的相关参数。这是一组键值对，如`Hostname`,`DomainName`,`User`,`AttachStdin`等等。具体详见Docker文档[v1.26 API](https://docs.docker.com/engine/api/v1.26).

对于所有的部署类型，环境变量可以通过`env`来传递。这需要指定每个环境变量的对象数组。每一个对象有name和value两个值，分别对应`name`和`value`两个属性。

当启动一个模块时，会分配一个TCP监听端口。模块应该在成功部署（服务于HTTP请求）之后监听该端口。改端口可以在`exec`和`cmdlineStart`中通过`%p`来指定。对于Docker部署。Okapi可以把公开的端口映射到动态分配的端口上。

同时，还可以引用已经启动的流程（可能在你开发的IDE中运行），该方法是将DeploymentDescriptor发布到`/_/discovery`，其中没有nodeId和LaunchDescriptor，但有运行模块的URL。

### Docker

Okapi使用Docker([Docker Engine API](https://docs.docker.com/engine/api/) )来启动镜像。Docker Deamon为了不处理Unix本地socket必须监听一个TCP端口。启用Docker Deamon依赖于系统主机。对于基于systemd的系统,`/lib/systemd/system/docker.service`服务必须进行调整,并且`ExecStart`命令应该包含`-H`用于监听主机和端口。例如``-H tcp://127.0.0.1:4243`。

```
vi /lib/systemd/system/docker.service
systemctl daemon-reload
systemctl restart docker
```

### 系统接口

模块可以提供系统接口，Okapi可以在某些定义良好的情况下向这些接口发送请求。按照惯例，这些接口的名称以`_`开头。

目前，我们已经定义两个系统接口。但随着时间推移，我们可能会提供更多。

#### tenant 接口

如果模块提供了一个名为`_tenant`的系统接口，那么每次启用租户时，Okapi都会调用该接口。请求包含关于新启用的模块的信息，以及同时禁用的一些模块的信息。例如在模块升级时，模块可以使用这些信息来升级或初始化它的数据库，并执行它需要的任何类型的内务管理。

具体[细节](#web-service)请参阅 `.../okapi/okapi-core/src/main/raml/raml-util`文件 `ramls/tenant.raml` and `schemas/moduleInfo.schema`。

okapi-test-module有一个非常简单的实现，而moduleTest显示了一个定义这个接口的模块描述。

租户接口在Okapi 1.0中启用。

#### TenantPermissions 接口

当为租户启用模块时，Okapi还尝试定位提供`_tenantPermissions`接口的模块，并调用该模块。通常，这将由权限模块提供。它获取一个结构，其中包含要启用的模块的ModuleDescriptor和permissionset。这样做的目的是将权限和权限集加载到权限模块中，以便将它们分配给用户，或用于其他权限集。

除非你重写权限模块，否则永远不需要提供此接口。

服务应该是幂等的，因为如果模块启用出现问题，可能会再次调用它。它应该首先删除命名模块的所有权限和权限集，然后在请求中插入它接收到的权限集。这样，它将清理模块的一些旧版本中引入的、不再使用的权限。

更多[细节](#web-service)请参阅 `.../okapi/okapi-core/src/main/raml/raml-util`下的`ramls/tenant.raml`和`schemas/moduleInfo.schema`文件。 okapi-test-header-module由一个非常简单的实现，moduleTest则显示了一个定义这个接口的模块描述。

tenantPermissions接口在Okapi 1.1中启用。

### 数据监控

Okapi会把可视化数据发送到Carbon/Graphite的后台，通过Grafana之类的工具来显示。Vert.x会自动推送一些数据，但是Okapi的各个部分会推送他们自己的数据。所以我们可以根据租户或模块来进行分类。单个模块也可以推送自己的数据。希望他们能使用一个与我们在Okapi中所做的类似的密钥命名方案。

通过`-enable-metrics`启用度量将开始向`localhost:2003`发送数据。

如果您将`graphiteHost`作为参数添加到java命令中，如：
`java -DgraphiteHost=graphite.yourdomain.io -jar okapi-core/target/okapi-core-fat.jar dev -enable-metrics`
数据指标将会发送到`graphite.yourdomain.io`

  * `folio.okapi.`_\$HOST_`.proxy.`_\$TENANT_`.`_\$HTTPMETHOD_`.`_\$PATH`_ -- 整个请求的时间，包括它最终调用的所有模块。
  * `folio.okapi.`_\$HOST_`.proxy.`_\$TENANT_`.module.`_\$SRVCID`_ -- 调用一个模块的时间。
  * `folio.okapi.`_\$HOST_`.tenants.count` -- 系统已知租户的数量
  * `folio.okapi.`_\$HOST_`.tenants.`_\$TENANT_`.create` -- 创建租户的时间
  * `folio.okapi.`_\$HOST_`.tenants.`_\$TENANT_`.update` -- 更新租户的时间
  * `folio.okapi.`_\$HOST_`.tenants.`_\$TENANT_`.delete` -- 删除租户的时间
  * `folio.okapi.`_\$HOST_`.modules.count` -- 系统已知模块的数量
  * `folio.okapi.`_\$HOST_`.deploy.`_\$SRVCID_`.deploy` -- 部署模块的时间
  * `folio.okapi.`_\$HOST_`.deploy.`_\$SRVCID_`.undeploy` -- 卸载模块的时间
  * `folio.okapi.`_\$HOST_`.deploy.`_\$SRVCID_`.update` -- 更新模块的时间
  `$` _NAME_变量当然会得到实际值。


在`doc`目录中有一些Grafana仪表板定义的例子:
* [`grafana-main-dashboard.json`](grafana-main-dashboard.json)
* [`grafana-module-dashboard.json`](grafana-module-dashboard.json)
* [`grafana-node-dashboard.json`](grafana-node-dashboard.json)
* [`grafana-tenant-dashboard.json`](grafana-tenant-dashboard.json)


这里有一些在Grafana中有用的图的例子。当你将编辑模式(行尾的工具菜单)更改为文本模式时，可以将这些内容直接粘贴到metric之下。

  * 租户的活动:

      `aliasByNode(sumSeriesWithWildcards(stacked(folio.okapi.localhost.proxy.*.*.*.m1_rate, 'stacked'), 5, 6), 4)`
  * HTTP每分钟的请求 (如 PUT, POST, DELETE等)：

      `alias(folio.okapi.*.vertx.http.servers.*.*.*.*.get-requests.m1_rate, 'GET')`
  * HTTP返回码 (如 4XX、5XX )：

      `alias(folio.okapi.*.vertx.http.servers.*.*.*.*.responses-2xx.m1_rate, '2XX OK')`
  * 租户使用的模块：

      `aliasByNode(sumSeriesWithWildcards(folio.okapi.localhost.SOMETENANT.other.*.*.m1_rate, 5),5)`

## 模块相关

本节视图总结一个模块的作者在创建一个模块时应该知道的所有内容。和其他项目一样，它仍在建设中。我们希望这里将有更多的资料，所以我们有必要将这部分独立为一个章节。

本节集中讨论提供常规web服务的常规模块。特殊模块、过滤器和身份验证大部分被省略。

在Okapi指南中有很多有用的信息。看到例如
* [版本控制和依赖](#版本控制和依赖)
* [状态码](#状态码)

### 一个模块的生命周期

一个模块在其生命中会有许多不同的阶段。

#### 部署

这意味着以某种方式启动在给定端口上侦听的进程。部署模块的方法有很多，但最终的结果是开始时运行。大多数情况下，部署由Okapi管理，但是也可以由一些不在本指南范围内的外部实体管理。

这里有几个值得注意的阶段：
* 模块应该在集群节点上运行，因此不应该使用任何本地存储。
* 可以有多个模块实例运行在不同的节点上，也可以在至同一个节点上。
* Okapi可以部署相同模块的不同版本，即使是在相同的节点上。
* 在启动时，模块应该只设置它的HTTP监听之类的东西。它不应该初始化任何数据库，请参见下面的"启用"。

#### 启用一个租户

当为租户启用模块时，Okapi会检查是否为模块提供了`_tenant`接口。如果有此接口。Okapi会发送HTTP POST给`/_/tenant`用于`_tenant`接口1.0及以后的版本。如果需要模块可以在这里初始化它的数据库等等。通过POST发送一个JSON对象：`module_to`成员是启用的模块ID。

详细定义请参考，请参考 https://github.com/folio.org/raml/blob/raml1.0/ramls/tenant.raml 

#### 更新

当一个模块升级到一个新版本时，它会为每个租户单独执行。一些租户可能不希望在旺季升级，而另一些租户可能希望所有东西都是最新版本的。这个过程由部署模块新版本的Okapi启动，同时旧版本也在运行。然后，不同的租户可以一次升级到一个新版本。

升级的是指是通过Okapi禁用旧版本并启用新版本的过程。如果提供`_tenant`1.0或更高的版本，Okapi将使用`/_/tenant`路劲发送POST请求。通过POST请求将发送一个JSON对象：该对象包含`module_from`属性，用以表示我们要升级模块的ID,`module_to`用以表示我们要升级到的模块的ID。注意，此时目标模块(`module_to`)的ModuleDescriptor已经被调用。

将大量数据升级到较新的模式可能很慢。我们正在考虑一种使其异步发生的方法，但还没有设计出来。 = =b

我们使用语义版本控，详见[版本控制和依赖](#版本控制和依赖)

#### 禁用

Okapi 1.1及以后的版本为禁用租户提供了一个`_tenant`接口，其路劲为`/_/tenant/disable`。通过POST请求传递一个JSON对象:`module_from`表示时被禁用的模块ID。

#### 清除数据

当为租户清除模块时，它会禁用模块的租户，但也会删除持久内容。在Okapi 1.0及以后的版本，可以通过提供`_tenant`接口的DELETE方法来实现这一点。

#### 租户参数

模块除了进行基本的存储初始化等操作外，还可以加载一系列的相关数据。这可以通过提供租户参数来控制。这些属性以键值对(key-value)的形式在启用或升级时传递给模块。只有在安装指定的tenantParameters以及租户接口版本为1.2是才生效。

### 租户接口

完整的`_tenant`接口 1.1/1.2版本如下：

```
   "id" : "_tenant",
   "version" : "1.2",
   "interfaceType" : "system",
   "handlers" : [ {
     "methods" : [ "POST", "DELETE" ],
     "pathPattern" : "/_/tenant"
    }, {
     "methods" : [ "POST" ],
     "pathPattern" : "/_/tenant/disable"
    } ]
```

#### 关闭Okapi

当Okapi关闭时，它也会关闭所有的模块。当启动时，这些模块也会被再次启动。作为一个模块的作者，你不必太担心这些。

### HTTP
FOLIO的主要设计标准之一是基于RESTful的HTTP服务，使用JSON进行数据传输。节描述了关于处理HTTP的一些细节。我们必须处理HTTP。。。。。。。。 = =b

#### HTTP 状态码
详见[状态码](#状态码).


#### X-Okapi 消息头

Okapi使用一些X-Okapi的消息头用以在客户端发送请求时传输额外的信息，这些消息头被用于Okapi本身和模块处理请求，以及模块箱进一步通过Okapi请求其他模块时，当模块返回消息给Okapi，或者返回给客户端时。同样这些特殊的消息头也被用于auth模块和Okapi之间的通信。但是我们可以忽略他们。

这里罗列了一些相关的消息头：
* `X-Okapi-Token` 身份验证令牌。携带租户和用户id以及一些权限
* `X-Okapi-Tenant` 我们操作的租户ID (UUID)
* `X-Okapi-User-Id` 已登录用户的UUID
* `X-Okapi-Url` Okapi安装的基本URL。例如 http://localhost:9130 。这也可以指向Okapi前面的负载均衡，你所需要知道的就是在向其他模块发出进一步请求时使用它。其基本的URL可以以`https://folio.example.com/okapi`这样的路径结束，以确保主机名和端口与前端URL`https://folio.example.com`匹配，从而避免CORS HTTP OPTIONS 请求(跨域请求)。
* `X-Okapi-Request-Id` 当前请求的ID, 例如"821257/user;744931/perms"，这表明这是请求821257的`/users/…`给744931的`/perms/...`。这些数字式随机的。是Okapi看到请求后选择的。
* `X-Okapi-Trace` 模块可以返回此信息将跟踪和计时信息添加到响应头中。这样客户端就可以看到请求的结束位置以及各部分花费了多少时间。例如`GET sample-module-1.0.0 http://localhost:9231/testb : 204 3748us`
* `X-Okapi-Permissions`模块所希望的和已赋予的权限。(注意，如果一个模块严格要求一个权限，而这个权限没有被授予，那么这个请求将永远不会到达这个模块。这些仅适用于特殊情况，比如包含模块可以处理的其自己的关于用户的敏感数据)。

完整的信息详见[X-Okapi-Headers.java](#https://github.com/folio-org/okapi/blob/master/okapi-common/src/main/java/org/folio/okapi/common/XOkapiHeaders.java)

如果使用Java编写模块，建议您参考这个文件，而不是定义自己的模块。这样，您还可以为它们获得Javadoc。

The full list is in
[X-Okapi-Headers.java](../okapi-common/src/main/java/org/folio/okapi/common/XOkapiHeaders.java).
If writing your module in Java, it is recommended you refer to this file, instead
of defining your own. That way, you also get the Javadoc for them.

当UI或其他客户机程序向Okapi发出请求时。它需要传递`X-Okapi-Token`报头。当它调用`authn/login`时，应该已经收到了一个。（注意：在登录请求和其他一些特殊情况下，比如安装的早期阶段，客户端还没有`X-Okapi-Token`。在这种情况下，它应该传递一个`X-Okapi-Tenant`报头，以告诉它充当的是哪个承租者。）客户端可以选择在更标准的`授权`头中传递令牌。

客户端还可以传递一个`X-Okapi-Request-Id`报头。通过将Okapi的日志条目绑定到各种请求，这将有助于调试。如果UI中的一个操作需要对后端模块发出多个请求，则特别有用。所有请求都应该传递相同的Id, Okapi将通过向每个请求添加自己的Id来区分它们。例如` 123456/disable-user`，其中前缀是一个随机数，字符串是对操作的非常简短的描述。

在Okapi将请求传递给实际模块之前，它会执行各种操作。它要求auth过滤器验证`X-Okapi-Token`报头，并从中提取各种信息。Okapi将以下头信息传递给模块:`X-Okapi-Token`、`X-Okapi-Tenant`、`X-Okapi-User-Id`、`X-Okapi-Url`、`X-Okapi-Request-Id`、`X-Okapi-Permissions`。（注意：`X-Okapi-Request-Id`报头是一个新的报头，但是它将包含接收到的报头Okapi的值。同样，`X-Okapi-Token`头可能与传递给Okapi的头不同，它可能包含特定于模块的权限等）

如果模块希望向其他模块发出请求，它应该将其地址与需要访问的路径结合到`X-Okapi-Url`的基URL。它至少应该传递`X-Okapi-Token`，最好还传递`X-Okapi-Request-Id`。在很多情况下，传递所有的`X-Okapi`头文件更容易——Okapi将删除那些可能混淆的头文件。

当模块接收到来自另一个模块的响应时，最好能将所有`X-Okapi-Trace`头信息传递到它自己的响应中。有助于客户机调试其代码，方法是列出在此过程中发出的所有请求，以及每个请求花费的时间。但这并不是严格必要的。

当模块返回它的响应时，它不需要传递任何头部给Okapi，但是它可以传递它自己的一个或两个`X-Okapi-Trace`头部。模块将所有`X-Okapi`头从请求复制到响应，这是一个传统。这完全可以接受，但没有必要。

当Okapi将响应传递给客户机时，它将传递上面提到的所有`X-Okapi`报头。它可能会删除一些内部使用的头文件。
