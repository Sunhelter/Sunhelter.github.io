### 介绍

ABP提供了构建模块的基础结构，并将它们组合在一起以创建应用程序。模块可以依赖于另一个模块。程序集通常被视为模块。如果创建具有多个程序集的应用程序，则建议为每个程序集创建一个模块定义。

模块系统目前专注于服务器端而不是客户端。

### 模块定义

模块由[ABP package](https://www.nuget.org/packages/Abp)中的AbpModule派生的类定义。假设我们正在开发一个可以在不同应用程序中使用的Blog模块。最简单的模块定义如下所示：

``` C#
    public class MyBlogApplicationModule : AbpModule
    {
        public override void Initialize()
        {
            IocManager.RegisterAssemblyByConvention(Assembly.GetExecutingAssembly());
        }
    }
```
如果需要，模块定义类负责通过[依赖注入](https://aspnetboilerplate.com/Pages/Documents/Dependency-Injection)来注册它（它可以按常规方式完成，如上所示）。它还可以配置应用程序和其他模块，为应用程序添加新功能，等等......

### 生命周期方法

ABP在应用程序启动和关闭时调用某些特定的模块方法。您可以覆盖这些方法以执行某些特定任务。

ABP**按照依赖的顺序**调用这些方法。如果模块A依赖于模块B，则先初始化模块B。

开始方法的正确顺序是：预初始化B，预初始化A，初始化B，初始化A，初始化后B和初始化后A。所有依赖顺序图都是如此。**关闭方法**类似，不过**顺序相反**。

#### 预初始化

当应用程序启动时，首先调用此方法。这是在初始化之前[配置](https://aspnetboilerplate.com/Pages/Documents/Startup-Configuration)框架和其他模块的首选方法。

你还可以在此处编写一些特定代码，以便在注册[依赖注入](https://aspnetboilerplate.com/Pages/Documents/Dependency-Injection)之前运行。例如，如果您创建[传统注册](https://aspnetboilerplate.com/Pages/Documents/Dependency-Injection)类，则应使用IocManager.AddConventionalRegisterer方法在此处注册它。

#### 初始化

这里是应该注册[依赖注入](https://aspnetboilerplate.com/Pages/Documents/Dependency-Injection)的地方。它通常使用IocManager.RegisterAssemblyByConvention方法完成。如果要自定义依赖项注册，请参阅[依赖项注入文档](https://aspnetboilerplate.com/Pages/Documents/Dependency-Injection)。

#### 初始化后

该方法在应用启动的最后调用。在这里可以安全地解析一个依赖。

#### 关闭

该方法在应用关闭的时候调用

### 模块依赖

一个模块可以独立于另一个模块。你需要使用DependOn属性显示声明依赖，如下所示：

``` C#
    [DependsOn(typeof(MyBlogCoreModule))]
    public class MyBlogApplicationModule : AbpModule
    {
        public override void Initialize()
        {
            IocManager.RegisterAssemblyByConvention(Assembly.GetExecutingAssembly());
        }
    }
```

在这里，我们向ABP声明MyBlogApplicationModule依赖于MyBlogCoreModule，因此MyBlogCoreModule应该在MyBlogApplicationModule之前初始化。

ABP可以从启动模块开始递归地解析依赖关系，并相应地初始化它们。**启动模块**初始化时最后一个模块。

### 插件模块

尽管ABP是从启动模块开始通过依赖项解析模块，ABP也可以**动态**加载模块。**AbpBootstrapper**定义了**PlugInSources**属性，可以用于在动态加载的[插件模块](https://aspnetboilerplate.com/Pages/Documents/Plugin)中添加资源。任何实现**IPlugInSource**接口的类都可以是插件资源。**PlugInFolderSource**类通过从位于文件夹中的组件获得插件模块实现它。

#### ASP.NET Core

ABP在AddAbp扩展方法中定义选项以便在**Startup**类中添加插件：
``` C#
    services.AddAbp<MyStartupModule>(options =>
    {
        options.PlugInSources.Add(new FolderPlugInSource(@"C:\MyPlugIns"));
    });
```

还可以使用写法更简单的AddFolder扩展方法：

``` C#
    services.AddAbp<MyStartupModule>(options =>
    {
        options.PlugInSources.AddFolder(@"C:\MyPlugIns");
    });
```

有关Startup类的更多信息，请参阅[ASP.NET Core文档](https://aspnetboilerplate.com/Pages/Documents/AspNet-Core)。

#### ASP.NET MVC, Web API

对于经典的ASP.NET MVC应用程序，我们可以通过重写Global.asax中的**Application_Start**来添加插件文件夹，如下所示：

``` C#
    public class MvcApplication : AbpWebApplication<MyStartupModule>
    {
        protected override void Application_Start(object sender, EventArgs e)
        {
            AbpBootstrapper.PlugInSources.AddFolder(@"C:\MyPlugIns");
            //...
            base.Application_Start(sender, e);
        }
    }
```

##### 插件中的控制器

如果您的模块包含MVC或Web API控制器，则ASP.NET无法解析您的控制器。要解决此问题，您可以更改global.asax文件，如下所示：

``` C#
    using System.Web;
    using Abp.PlugIns;
    using Abp.Web;
    using MyDemoApp.Web;

    [assembly: PreApplicationStartMethod(typeof(PreStarter), "Start")]

    namespace MyDemoApp.Web
    {
        public class MvcApplication : AbpWebApplication<MyStartupModule>
        {
        }

        public static class PreStarter
        {
            public static void Start()
            {
                //...
                MvcApplication.AbpBootstrapper.PlugInSources.AddFolder(@"C:\MyPlugIns\");
                MvcApplication.AbpBootstrapper.PlugInSources.AddToBuildManager();
            }
        }
    }
```

### 附加组件

IAssemblyFinder和ITypeFinder（ABP用于解析应用程序中的特定类）的默认实现仅在这些程序集中查找模块程序集和类型。我们可以重写模块中的GetAdditionalAssemblies方法以包含其他程序集。

### 自定义模块方法

你的模块还可以具有自定义方法，以供其他依赖于此模块的模块使用。假设 MyModule2依赖于 MyModule1，想要在预初始化方法中调用 MyModule1的方法。

``` C#
    public class MyModule1 : AbpModule
    {
        public override void Initialize()
        {
            IocManager.RegisterAssemblyByConvention(Assembly.GetExecutingAssembly());
        }

        public void MyModuleMethod1()
        {
            //this is a custom method of this module
        }
    }

    [DependsOn(typeof(MyModule1))]
    public class MyModule2 : AbpModule
    {
        private readonly MyModule1 _myModule1;

        public MyModule2(MyModule1 myModule1)
        {
            _myModule1 = myModule1;
        }

        public override void PreInitialize()
        {
            _myModule1.MyModuleMethod1(); //Call MyModule1's method
        }

        public override void Initialize()
        {
            IocManager.RegisterAssemblyByConvention(Assembly.GetExecutingAssembly());
        }
    }
```

这里我们将 MyModule1通过构造函数注入到 MyModule2，因此 MyModule2可以调用 MyModule1的自定义 方法。只有在 MyModule2依赖于 MyModule1时才可以这么做。

### 模块配置

尽管可以使用自定义模块方法配置模块，我们还是建议使用[启动配置](https://aspnetboilerplate.com/Pages/Documents/Startup-Configuration)系统来定义和设置模块配置。

### 模块生命

模块类自动注册为**单例（singleton）**。