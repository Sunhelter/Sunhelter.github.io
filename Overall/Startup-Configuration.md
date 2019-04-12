## 启动配置

ABP提供了基础结构和模型，可在启动时配置[模块](/Overall/Module-System)。

### ABP配置

在模块中的**预初始化（PreInitialize）**方法中配置ABP。举个例子：

``` C#
    public class SimpleTaskSystemModule : AbpModule
    {
        public override void PreInitialize()
        {
            //添加语言
            Configuration.Localization.Languages.Add(new LanguageInfo("en", "English", "famfamfam-flag-england", true));
            Configuration.Localization.Languages.Add(new LanguageInfo("tr", "Türkçe", "famfamfam-flag-tr"));

            //添加本地化资源
            Configuration.Localization.Sources.Add(
                new XmlLocalizationSource(
                    "SimpleTaskSystem",
                    HttpContext.Current.Server.MapPath("~/Localization/SimpleTaskSystem")
                    )
                );

            //配置导航菜单
            Configuration.Navigation.Providers.Add<SimpleTaskSystemNavigationProvider>();        
        }

        public override void Initialize()
        {
            IocManager.RegisterAssemblyByConvention(Assembly.GetExecutingAssembly());
        }
    }
```

ABP深深基于[模块化](/Overall/Module-System.md)设计。不同模块都可以配置ABP。比如，不同的模块可以通过添加导航来源来在主菜单中添加自己的菜单选项（详细配置方法请参阅[本地化](/Presentation.Layer/Localization)和[导航](/Presentation.Layer/Navigation)）。

#### 替换内置服务

**Configuration.ReplaceService**方法可以用来**重写**内置服务。比如，你可以像下面这样自定义实现IAbpSession服务：

```C#
    Configuration.ReplaceService<IAbpSession, MySession>(DependencyLifeStyle.Transient);
```

ReplaceService方法有一个重载来传递**行为（action）**，使其以自定义的方式进行替换（你可以直接使用Castle Windsor的高级注册API）。

可以在不同的模块中多次更换相同的服务，但只有最后一次替换有效。模块预初始化方法按照[依赖顺序](/Overall/Module-System)执行。

### 模块配置

除了框架自带的启动配置，还可以扩展通过IAbpModuleConfigurations接口为模块提供配置点。例如：
``` C#
    ...
    using Abp.Web.Configuration;
    ...
    public override void PreInitialize()
    {
        Configuration.Modules.AbpWebCommon().SendAllExceptionsToClients = true;
    }
    ...
```



In this example, we configured the AbpWebCommon module to send all
exceptions to clients.

Not every module should define this type of configuration. It's
generally needed when a module will be re-usable in different
applications and needs to be configured on startup.

### Creating Configuration For a Module

Assume that we have a module named MyModule and it has some
configuration properties. First, we create a class for these configurable
properties:

    public class MyModuleConfig
    {
        public bool SampleConfig1 { get; set; }

        public string SampleConfig2 { get; set; }
    }

We then register this class via [Dependency
Injection](Dependency-Injection.md) on the **PreInitialize** method of
MyModule (Thus, it will be injectable):

    IocManager.Register<MyModuleConfig>();

It should be registered as a **Singleton** like in this example. We can now
use the following code to configure MyModule in our module's
PreInitialize method:

    Configuration.Get<MyModuleConfig>().SampleConfig1 = false;

While we can use the IAbpStartupConfiguration.Get method as shown below, we
can create an extension method to the IModuleConfigurations like this:

    public static class MyModuleConfigurationExtensions
    {
        public static MyModuleConfig MyModule(this IModuleConfigurations moduleConfigurations)
        {
            return moduleConfigurations.AbpConfiguration.Get<MyModuleConfig>();
        }
    }

Now other modules can configure this module using the extension method:

    Configuration.Modules.MyModule().SampleConfig1 = false;
    Configuration.Modules.MyModule().SampleConfig2 = "test";

This makes it easy to investigate module configurations and collect them in
a single place (Configuration.Modules...). ABP itself defines extension
methods for its own module configurations.

At some point, MyModule needs this configuration. You can inject
MyModuleConfig and use the configured values. Example:

    public class MyService : ITransientDependency
    {
        private readonly MyModuleConfig _configuration;

        public MyService(MyModuleConfig configuration)
        {
            _configuration = configuration;
        }

        public void DoIt()
        {
            if (_configuration.SampleConfig2 == "test")
            {
                //...
            }
        }
    }

This way, modules can create central configuration points in the ASP.NET
Boilerplate system.
