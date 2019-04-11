### 什么是ABP？

ABP是一个有完善文档的开源应用框架。它不只是框架，它还提供了一个强大的基于**领域驱动设计**（DDD）和**最佳实践**的架构模型。

ABP使用最新的**ASP.NET Core**和**EF Core**，也支持ASP.NET MVC 5.x与EF 6.x。

### 示例

通过一个示例来看ABP的优点：
``` C#
    public class TaskAppService : ApplicationService, ITaskAppService
    {
        private readonly IRepository<Task> _taskRepository;

        public TaskAppService(IRepository<Task> taskRepository)
        {
            _taskRepository = taskRepository;
        }

        [AbpAuthorize(MyPermissions.UpdateTasks)]
        public async Task UpdateTask(UpdateTaskInput input)
        {
            Logger.Info("Updating a task for input: " + input);

            var task = await _taskRepository.FirstOrDefaultAsync(input.TaskId);
            if (task == null)
            {
                throw new UserFriendlyException(L("CouldNotFindTheTaskMessage"));
            }

            input.MapTo(task);
        }
    }
```

这里我们看到了一个[应用服务](/WebSite/Application.Layer/Application-Services.md)方法示例。在DDD中，应用服务被表现层直接使用，来执行应用的**用例**。还可以考虑通过AJAX使用JavaScript来调用**UpdateTask**。

这里是ABP的一些优点：

* [**依赖注入(DI)**](https://aspnetboilerplate.com/Pages/Documents/Dependency-Injection)：ABP使用并提供了一个健壮而又传统的DI基础设施。因为上面的类是在一个应用服务中定义的，所以它会按照惯例约定短暂地（每个请求创建一次）注册到DI容器中。它也简单地注入了所有依赖（本例中注入了IRepository）。
* [**仓储**](https://aspnetboilerplate.com/Pages/Documents/Repositories)：ABP可以为每一个实体创建一个默认的仓储（本例中是IRepository）。默认的仓储有许多有用的方法，如本例中的 FirstOrDefault。我们也可以根据我们的需求轻易地扩展默认仓储。仓储抽象了DBMS和ORM，并简化了数据的访问逻辑。
* [**授权**](https://aspnetboilerplate.com/Pages/Documents/Authorization)：ABP可以声明式检测权限。如果当前的用户没有“updating task”的权限或者没登录，那么ta不能访问UpdateTask方法。它不仅可以使用声明式特性，还可以有其他的授权方法。
* [**验证**](https://aspnetboilerplate.com/Pages/Documents/Validating-Data-Transfer-Objects)：ABP会自动检测输入是否为null。它也基于标准的数据注解特性和自定义的验证规则验证输入对象的所有属性。如果请求不合法，那么它会抛出一个合适的验证异常并在客户端处理。
* [**审计日志**](https://aspnetboilerplate.com/Pages/Documents/Audit-Logging)：用户，浏览器，IP地址，调用服务，方法，参数，调用时间，执行时长和其他的一些信息也会基于惯例和配置为每个请求自动地保存。
* [**工作单元**](https://aspnetboilerplate.com/Pages/Documents/Unit-Of-Work)：在ABP中，每个应用服务方法默认视为一个工作单元。它会自动创建一个连接并在方法的开始位置开启一个事务。如果方法没有异常地成功完成了，那么事务会提交并且释放连接。即使该方法使用了不同的仓储或者方法，它们全部也都是原子的（事务的）。当事务提交时，实体的所有改变都会自动保存。因此，正如这里展示的那样，我们甚至都不用调用_repository.Update(task)方法。
* [**异常处理**](https://aspnetboilerplate.com/Pages/Documents/Handling-Exceptions)：在一个使用了ABP框架的Web应用中，我们基本上不用处理异常。所有的异常都会默认自动处理。如果一个异常发生了，那么ABP会自动地记录它，然后返回给客户端一个合适的结果。比如，对于Ajax请求，它会向客户端返回一个JSON对象，表明发生了错误。它隐藏了客户端的实际异常，除非像示例中那样使用的用户友好型异常。它也理解并处理客户端的错误，最后将合适的信息呈现给用户。
* [**日志**](https://aspnetboilerplate.com/Pages/Documents/Logging)：我们可以使用在基类中定义的Logger来写日志。ABP默认使用了Log4Net，但是它是可改变的或可配置的。
* [**本地化**](https://aspnetboilerplate.com/Pages/Documents/Localization)：有没有注意到我们使用了L方法抛出异常？这样它会基于当前用户的文化自动进行本地化。在[本地化](https://aspnetboilerplate.com/Pages/Documents/Localization)文档中查看更多信息。
* [**自动映射**](https://aspnetboilerplate.com/Pages/Documents/Data-Transfer-Objects)：上面的最后一行代码，我们使用了ABP的MapTo扩展方法将输入对象的属性映射到实体属性。它使用了AutoMapper库来执行映射。我们可以基于命名习惯轻易地将属性从一个对象上映射到另一个对象上。
* [**动态Web API层**](https://aspnetboilerplate.com/Pages/Documents/Dynamic-Web-API)：TaskAppService实际上是一个简单的类。我们一般会写一个Web API Controller包装器来将方法暴露给javascript客户端，但ABP在运行时会自动完成。这样，我们可以从客户端直接使用应用服务方法。
* [**动态Ajax代理**](https://aspnetboilerplate.com/Pages/Documents/Dynamic-Web-API#dynamic-javascript-proxies)：ABP创建的代理方法使调用应用服务方法就像调用客户端的javascript方法一样简单。

我们可以在这个简单的示例中看到ABP的优点。所有这些任务通常需要很长时间，但它们可以由框架自动处理。

除了这个简单的示例，ABP还为[模块化](https://aspnetboilerplate.com/Pages/Documents/Module-System)，[多租户](https://aspnetboilerplate.com/Pages/Documents/Multi-Tenancy)，[缓存](https://aspnetboilerplate.com/Pages/Documents/Caching)，[后台作业](https://aspnetboilerplate.com/Pages/Documents/Background-Jobs-And-Workers)，[数据过滤器](https://aspnetboilerplate.com/Pages/Documents/Data-Filters)，[设置管理](https://aspnetboilerplate.com/Pages/Documents/Setting-Management)，[领域事件](https://aspnetboilerplate.com/Pages/Documents/EventBus-Domain-Events)，单元和集成测试等提供了强大的基础架构和开发模型。你可以专注于你的业务代码，不要重复自己！

### 开始使用

你可以从开始模板或介绍教程开始使用它。

#### 开始模板

从开始模板直接创建一个现代化的启动项目。
![开始模板](./img/Overall/module-zero-core-template-ui-home.png)

开始模板为应用提供了一个基础页面和一些常用功能。这里有一些开始模板以供选择。

##### ASP.NET Core

* [使用ASP.NET Core & Angular创建单页应用程序](https://aspnetboilerplate.com/Pages/Documents/Zero/Startup-Template-Angular)
* [使用ASP.NET Core & jQuery创建多页应用程序](https://aspnetboilerplate.com/Pages/Documents/Zero/Startup-Template-Core)

##### ASP.NET MVC 5.x

* [使用ASP.NET MVC 5.x与AngularJS 1.x或ASP.NET MVC 5.x与jQuery](https://aspnetboilerplate.com/Pages/Documents/Zero/Startup-Template)

有关其他组合，请查看[下载页面](https://aspnetboilerplate.com/Templates)。

#### 介绍教程

分步教程介绍了本框架，并说明了如何基于开始模板创建应用程序。

##### ASP.NET Core

* [ASP.NET Core与Entity Framework Core介绍](/WebSite/Articles/Introduction.Net.Core.EFCore.P1.md)
* [使用ASP.NET Core，Entity Framework Core与Angular开发多租户（SaaS）应用](https://aspnetboilerplate.com/Pages/Documents/Articles/Developing-a-Multi-Tenant-SaaS-Application-with-ASP.NET-MVC-EntityFramework-AngularJs/index.html)

##### ASP.NET MVC 5.x

* [介绍ASP.NET MVC 5.x, Web API 2.x, EntityFramework 6.x与AngularJS 1.x](https://aspnetboilerplate.com/Pages/Documents/Articles/Introduction-With-AspNet-MVC-Web-API-EntityFramework-and-AngularJs/index.html)
* [使用ASP.NET MVC 5.x, EntityFramework 6.x与AngularJS 1.x开发多租户（SaaS）应用](https://aspnetboilerplate.com/Pages/Documents/Articles/Developing-a-Multi-Tenant-SaaS-Application-with-ASP.NET-MVC-EntityFramework-AngularJs/index.html)

### 示例

在[示例页面](https://aspnetboilerplate.com/Samples)可以查看许多使用本框架开发的示例项目。

### 社区

这是一个开源项目，接受社区贡献。

* 使用[GitHub库](https://github.com/aspnetboilerplate/aspnetboilerplate)访问最新的源代码，创建[issues](https://github.com/aspnetboilerplate/aspnetboilerplate/issues)并发送[pull requests](https://github.com/aspnetboilerplate/aspnetboilerplate/pulls)。
* 使用[Stack Overflow的aspnetboilerplate标签](https://stackoverflow.com/questions/tagged/aspnetboilerplate)询问用法问题。
* 在Twitter上关注[aspboilerplate](https://twitter.com/aspboilerplate)以获取最新消息。