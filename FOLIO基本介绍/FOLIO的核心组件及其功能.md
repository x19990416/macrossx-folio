FOLIO总体来说分为两部分：控制面和行为面，如下所述：
- <font color='red'>行为面</font>被称为"Module"，是FOLIO生态系统中的模块，是一种接口契约，没有确切的定义。所以满足如下几个特征的均可以成为Okapi的模块
  - 模块为一个网络服务，使用REST风格的WEB服务协议进行同行，通信内容通常使用JSON格式.
  - 模块带有一个描述文件([ModuleDescriptor.json](https://github.com/folio-org/okapi/blob/master/okapi-core/src/main/raml/ModuleDescriptor.json)),用以声明模块的信息，如ID，名称等；也指定模块的依赖关系，并报告其所提供的所有接口；列出模块处理所需的所有列表，这样为Okapi的流量代理提供必要信息。
  - 遵循版本控制规则
  - 提供监测所需接口
  
- 控制面被称为"Okapi"，是FOLIO的核心，管理所有FOLIO的的功能,大致如图所示：
![GitHub](https://github.com/folio-org/okapi/blob/master/doc/module_management.png "module_management.png")

  1. Proxy

  2. Discovery
  
  3. Deployment
  
  4. Env
