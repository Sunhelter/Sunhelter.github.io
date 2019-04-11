### 介绍

应用程序代码库的分层是一种被广泛接受的技术，有助于降低复杂性，并提高代码的可重用性。为了实现分层体系结构，ABP遵循**领域驱动设计**原则。

### 领域驱动设计分层

在领域驱动设计（DDD）中有以下四个基础层：
* 表现层：提供用户界面。使用应用层实现用户交互。
* 应用层：表现层与领域层的媒介。负责组织业务对象，执行特定的应用任务。
* 领域层：包含业务对象和逻辑规则。应用的核心。
* 基础设施层：提供通用技术功能，主要是用第三方库支持更高层。

### ABP应用程序体系结构模型

除了DDD以外，一个现代的架构应用程序还有其他逻辑和物理层。建议使用以下模型来实现ABP应用程序。ABP不仅通过提供基类和服务更容易地实现此模型，而且还提供从此模型直接开始的[开始模板](https://aspnetboilerplate.com/Templates)。

![ABPN层架构](/img/Overall/abp-nlayer-architecture.png)

#### 客户端应用

这是通过HTTP API（API控制器，[OData](https://aspnetboilerplate.com/Pages/Documents/OData-Integration)控制器，甚至可能是GraphQL端点）将应用程序用作服务的远程客户端。远程客户端可以是SPA（单页面应用程序），移动应用程序或第三方消费者。[本地化](https://aspnetboilerplate.com/Pages/Documents/Localization)和[导航](https://aspnetboilerplate.com/Pages/Documents/Navigation)可以在此应用程序内完成。

#### 表现层

ASP.NET [Core] MVC（模型 - 视图 - 控制器）可以被认为是表示层。它可以是物理层（通过HTTP API使用应用程序）或逻辑层（直接注入和使用[应用程序服务](https://aspnetboilerplate.com/Pages/Documents/Application-Services)）。在任何一种情况下，它都可以包括[本地化](https://aspnetboilerplate.com/Pages/Documents/Localization)，[导航](https://aspnetboilerplate.com/Pages/Documents/Navigation)， [对象映射](https://aspnetboilerplate.com/Pages/Documents/Object-To-Object-Mapping)， [缓存](https://aspnetboilerplate.com/Pages/Documents/Caching)，[配置管理](https://aspnetboilerplate.com/Pages/Documents/Setting-Management)，[审计日志记录](https://aspnetboilerplate.com/Pages/Documents/Audit-Logging)等。它还涉及[授权](https://aspnetboilerplate.com/Pages/Documents/Authorization)，[会话](https://aspnetboilerplate.com/Pages/Documents/Abp-Session)， [功能](https://aspnetboilerplate.com/Pages/Documents/Feature-Management)（用于 [多租户](https://aspnetboilerplate.com/Pages/Documents/Multi-Tenancy)应用程序）和[异常处理](https://aspnetboilerplate.com/Pages/Documents/Handling-Exceptions)。



#### 分布式服务层

该层用于通过REST，OData，GraphQL等远程API提供应用程序/领域功能......它们不包含业务逻辑，只是将HTTP请求转换为领域交互，或者可以使用应用程序服务委托操作。该层通常包括[授权](https://aspnetboilerplate.com/Pages/Documents/Authorization)，[缓存](https://aspnetboilerplate.com/Pages/Documents/Caching)， [审计日志](https://aspnetboilerplate.com/Pages/Documents/Audit-Logging)，[对象映射](https://aspnetboilerplate.com/Pages/Documents/Object-To-Object-Mapping)，[异常处理](https://aspnetboilerplate.com/Pages/Documents/Handling-Exceptions)，[会话](https://aspnetboilerplate.com/Pages/Documents/Abp-Session)等……

#### 应用层

应用层主要包括[应用服务](https://aspnetboilerplate.com/Pages/Documents/Application-Services)，它使用领域层和领域对象（[领域服务](https://aspnetboilerplate.com/Pages/Documents/Domain-Services)， [实体](https://aspnetboilerplate.com/Pages/Documents/Entities)……）来执行请求的应用程序功能。它使用[数据传输对象（DTO）](https://aspnetboilerplate.com/Pages/Documents/Data-Transfer-Objects)从表现层或分布式服务层获取数据并将数据返回。它还可以处理[授权](https://aspnetboilerplate.com/Pages/Documents/Authorization)，[缓存](https://aspnetboilerplate.com/Pages/Documents/Caching)，[审计日志](https://aspnetboilerplate.com/Pages/Documents/Audit-Logging)，[对象映射](https://aspnetboilerplate.com/Pages/Documents/Object-To-Object-Mapping)，[会话](https://aspnetboilerplate.com/Pages/Documents/Abp-Session)等……

#### 领域层

这是实现我们领域逻辑的主要层。它包括[实体](https://aspnetboilerplate.com/Pages/Documents/Entities)，[值对象](https://aspnetboilerplate.com/Pages/Documents/Value-Objects)和[领域服务](https://aspnetboilerplate.com/Pages/Documents/Domain-Services)，来执行业务/领域逻辑。它还可以包括[规范](https://aspnetboilerplate.com/Pages/Documents/Specifications)和触发[领域事件](https://aspnetboilerplate.com/Pages/Documents/EventBus-Domain-Events)。它定义了存储库接口，以便从数据源（通常是DBMS）读取和保留实体。

#### 基础设施层

基础结构层使其他层工作：它实现存储库接口（例如使用[Entity Framework Core](https://aspnetboilerplate.com/Pages/Documents/Entity-Framework-Core)）以实际使用真实数据库。它还可能包括与供应商的集成以[发送电子邮件](https://aspnetboilerplate.com/Pages/Documents/Email-Sending)等。这不是所有层之下的严谨层，实际上常常通过实现其他层的抽象来支撑其他层。