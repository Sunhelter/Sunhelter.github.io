# ASP.NET Core与Entity Framework Core介绍 - 第一部分

> 从[GitHub仓库](https://github.com/aspnetboilerplate/aspnetboilerplate-samples/tree/master/SimpleTaskSystem-Core)获取源码

## 简介

本文是“使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用”系列文章的第一部分，查看其他部分：

* 使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用 - 第一部分（本文）
* [使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用 - 第二部分](/Articles/Introduction.Net.Core.EFCore.P2.md)

在本文中，我将使用以下工具带你创建一个简单的跨平台多层网页应用：

* [.Net Core](https://www.microsoft.com/net/core#windowsvs2017)作为跨平台应用开发基础框架。
* [ASP.NET Boilerplate](https://aspnetboilerplate.com/)（ABP）作为开始模板和应用框架。
* [ASP.NET Core](https://docs.asp.net/)作为网页框架。
* [Entity Framework Core](https://docs.efproject.net/)作为ORM框架。
* [Twitter BootStrap](http://getbootstrap.com/)作为HTML&CSS框架。
* [jQuery](http://jquery.com/)作为客户端AJAX/DOM库。
* [xUnit](https://xunit.github.io/)和[Shouldly](http://shouldly.readthedocs.io/en/latest/)作为服务端单元/集成测试。

同时我也会使用ABP开始模板中默认包含的Log4Net和AutoMapper。我们将会使用以下技术：

* [分层架构](http://aspnetboilerplate.com/Pages/Documents/NLayer-Architecture)
* [领域驱动设计](https://en.wikipedia.org/wiki/Domain-driven_design)（DDD）
* [依赖注入](http://www.aspnetboilerplate.com/Pages/Documents/Dependency-Injection)（DI）
* [集成测试](https://en.wikipedia.org/wiki/Integration_testing)

此处开发的项目是一个**简单任务管理应用**，任务可以指定给某人。不同于按层级顺序开发，我会采用垂直架构并随着开发进度改变层级。伴随着应用的构建，我将根据需要介绍一些ABP和其他框架的功能。

### 准备工作

你应安装以下工具来运行/开发应用：

* Visual Studio 2019
* SQL Server (你可以修改连接字符串到本地数据库)
* VS扩展：
   * [Bundler & Minifier](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.BundlerMinifier)
   * [Web Compiler](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.WebCompiler)

## 创建应用

使用[ABP的开始模板](http://www.aspnetboilerplate.com/Templates)来新建一个名为“**Acme.SimpleTaskApp**”的网页应用。在创建模板时可以选填公司名（这里是Acme）。我还选择了**多页应用**，因为我不想在本文中使用SPA（单页应用）。另外为了使用最基础的开始模板，我还**禁用了权限验证**。

![ABP创建模板](/img/Introduction.Net.Core.EFCore.P1/Template-Creation-3.png)

它创建了一个如下图所示的多层解决方案：

![开始模板项目](/img/Introduction.Net.Core.EFCore.P1/Template-Projects.png)

它包含6个以我输入的项目名称开头的项目：

* **.Core** 项目是领域/业务层（实体，领域服务……）
* **.Application** 项目是应用层（DTOs，应用服务……）
* **.EntityFramework** 项目是EF Core集成（其它层的EF Core抽象）
* **.Web** 项目是ASP.NET MVC层
* **.Test** 项目是单元与集成测试（至应用层，不包含页面层）
* **.Web.Test** 项目是ASP.NET Core的集成测试（包含页面层的完整集成测试）

当你运行应用时，你可以看到用户界面模板：

![首页模板](/img/Introduction.Net.Core.EFCore.P1/template-default-home.png)

它包含了顶部菜单，空白主页和关于页面，以及一个切换语言的下拉框。

## 开发应用

### 创建Task实体类

我想从一个简单的**Task**实体类开始。实体类属于领域层，因此我把它加进了 **.Core** 项目：

``` C#
using Abp.Domain.Entities;
using Abp.Domain.Entities.Auditing;
using Abp.Timing;
using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace Acme.SimpleTaskApp.Tasks
{
    [Table("AppTasks")]
    public class Task : Entity, IHasCreationTime
    {
        public const int MaxTitleLength = 256;
        public const int MaxDescriptionLength = 64 * 1024;    //64KB

        [Required]
        [StringLength(MaxTitleLength)]
        public string Title { get; set; }

        [StringLength(MaxDescriptionLength)]
        public string Description { get; set; }

        public DateTime CreationTime { get; set; }

        public TaskState State { get; set; }

        public Task()
        {
            CreationTime = Clock.Now;
            State = TaskState.打开;
        }

        public Task(string title, string description = null)
            : this()
        {
            Title = title;
            Description = description;
        }
    }

    public enum TaskState : byte
    {
        打开 = 0,
        已完成 = 1
    }
}

```

* 它是由ABP的基础[实体](http://www.aspnetboilerplate.com/Pages/Documents/Entities)类派生而来，基础类包含int型的默认ID字段。我们可以使用通用的 **Entity\<TPrimaryKey\>** 来选择不同的主键类型。
* **IHasCreationTime**是一个定义创建时间字段的接口（创建时间最好使用标准命名CreationTime）。
* **Task**实体类定义了一个必填的**Title**字段以及一个选填的**Description**字段。
* **TaskState**是一个定义任务状态的枚举。
* **Clock.Now**默认返回DateTime.Now。但它提供了一个抽象，使得我们在需要时可以很轻易的转换成DateTime.UtcNow。在ABP框架中始终记得使用Clock.Now来代替DateTime.Now。
* 我想将Task实体存储在数据库的AppTasks表中。

### 添加Task到DbContext

**.EntityFrameworkCore**项目包含一个预定义的**DbContext**。我应该在DbContext里为Task实体类添加一个**DbSet**：

``` C#
using Abp.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Acme.SimpleTaskApp.Tasks;

namespace Acme.SimpleTaskApp.EntityFrameworkCore
{
    public class SimpleTaskAppDbContext : AbpDbContext
    {
        //为实体添加DbSet属性
        public DbSet<Task> Tasks { get; set; }

        public SimpleTaskAppDbContext(DbContextOptions<SimpleTaskAppDbContext> options) 
            : base(options)
        {

        }
    }
}

```

现在，EF Core知道我们有一个Task实体类了。

### 创建首次数据库迁移

我们将创建一个初始数据库迁移，来新建数据库和AppTasks表。打开**VS的程序包管理器控制台**，运行**Add-Migration**命令（默认项目必须选择.EntityFrameworkCore项目）

![EFCore添加迁移](/img/Introduction.Net.Core.EFCore.P1/EF-Core-Initial-Migration-2.png)

这个命令在.EntityFrameworkCore项目中创建了包含迁移类和数据库模型快照的**Migrations**文件夹：

![EFCore初始化迁移](/img/Introduction.Net.Core.EFCore.P1/EntityFrameworkCore-Project-Migrations-2.png)

**自动生成**的“Initial”迁移类如下所示：

``` C#
namespace Acme.SimpleTaskApp.Migrations
{
    public partial class Initial : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "AppTasks",
                columns: table => new
                {
                    Id = table.Column<int>(nullable: false)
                        .Annotation("SqlServer:ValueGenerationStrategy", SqlServerValueGenerationStrategy.IdentityColumn),
                    Title = table.Column<string>(maxLength: 256, nullable: false),
                    Description = table.Column<string>(maxLength: 65536, nullable: true),
                    CreationTime = table.Column<DateTime>(nullable: false),
                    State = table.Column<byte>(nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_AppTasks", x => x.Id);
                });
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(
                name: "AppTasks");
        }
    }
}
```

这些代码用来在执行数据库迁移时创建AppTasks表（在[EntityFrameworkCore文档](https://docs.efproject.net/)中查看更多关于迁移的信息）。

### 创建数据库

在程序包管理器控制台中运行**Update-Database**命令来创建数据库。

![EF Update-Database命令](/img/Introduction.Net.Core.EFCore.P1/EF-Core-Database-Update-2.png)

这个命令在数据库中创建了一个名为**SimpleTaskAppDb**的数据库，并执行迁移（目前只有一个“Initial”迁移）。

![创建数据库](/img/Introduction.Net.Core.EFCore.P1/Created-Database.png)

现在我有了Task实体类和数据库中对应的表。我在表中输入一些简单的任务：

![AppTasks表](/img/Introduction.Net.Core.EFCore.P1/apptasks-table.png)

注意，数据库**连接字符串**在 **.Web**项目的**appsettings.json**文件中定义。

### Task应用服务

[应用服务(Application Services)](http://www.aspnetboilerplate.com/Pages/Documents/Application-Services)用来将领域逻辑暴露给展现层。展现层通过DTO（数据传输对象）作为可选参数调用应用程序服务，使用领域对象执行某些特定业务逻辑（并可能将DTO返回到展现层）。

在 **.Application**项目中新建应用服务**TaskAppService**来执行任务相关应用逻辑。首先，为应用服务定义接口：

``` C# 
public interface ITaskAppService : IApplicationService
{
    Task<ListResultDto<TaskListDto>> GetAll(GetAllTasksInput input);
}
```

虽然不是必须的，但是仍建议先定义接口。按照惯例，所有应用服务都**应该**在ABP中实现**IApplicationService**接口（它只是一个空标记接口）。我创建**GetAll**方法用来查询。为此我还定义了以下DTO：

``` C#
public class GetAllTasksInput
{
    public TaskState? State { get; set; }
}

[AutoMapFrom(typeof(Task))]
public class TaskListDto : EntityDto, IHasCreationTime
{
    public string Title { get; set; }

    public string Description { get; set; }

    public DateTime CreationTime { get; set; }

    public TaskState State { get; set; }
}
```

* **GetAllTasksInput**定义了**GetAll**应用服务方法的输入参数。我将状态字段添加在DTO对象中，而没有直接将它定义为方法参数。因此我可以在不修改现有客户端的情况下将其他参数添加到此DTO中（我们可以直接向该方法添加状态参数）。
* **TaskListDto**用于返回Task数据。它由只定义了**ID**字段的**EntityDto**派生，（我们可以在自己的Dto中添加ID字段，而不从EntityDto中派生）。我们定义了[ **AutoMapFrom** ]属性来创建从Task实体到TaskListDto的[AutoMapper](http://automapper.org/)映射。该属性在[Abp.AutoMapper NuGet包](http://nuget.org/packages/Abp.AutoMapper)中定义。
* 最后，**ListResultDto**是一个包含多项列表的简单类（我们可以直接返回一个List<TaskListDto>）。

现在，我们可以像下面这样实现**ITaskAppService**：

``` C#
using Abp.Application.Services.Dto;
using Abp.Domain.Repositories;
using Abp.Linq.Extensions;
using Acme.SimpleTaskApp.Tasks.Dtos;
using Microsoft.EntityFrameworkCore;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace Acme.SimpleTaskApp.Tasks
{
    public class TaskAppService : SimpleTaskAppAppServiceBase, ITaskAppService
    {
        private readonly IRepository<Task> _taskRepository;

        public TaskAppService(IRepository<Task> taskRepository)
        {
            _taskRepository = taskRepository;
        }

        public async Task<ListResultDto<TaskListDto>> GetAll(GetAllTasksInput input)
        {
            var tasks = await _taskRepository
                .GetAll()
                .WhereIf(input.State.HasValue, t => t.State == input.State.Value)
                .OrderByDescending(t => t.CreationTime)
                .ToListAsync();

            return new ListResultDto<TaskListDto>(
                ObjectMapper.Map<List<TaskListDto>>(tasks)
                );
        }
    }
}

```

* **TaskAppService**由开始模板中（由ABP的ApplicationService派生）的**SimpleTaskAppAppServiceBase**派生。这不是必须的，App Service可以是普通类。但**ApplicationService**基类带有一些预注入服务（比如这里用到的ObjectMapper）。
* 使用[依赖注入](http://www.aspnetboilerplate.com/Pages/Documents/Dependency-Injection)来获取[仓储库（Repositories）](http://www.aspnetboilerplate.com/Pages/Documents/Repositories)。
* **Repositories**用于抽象实体的数据库操作。ABP为每个实体创建预定义的仓储库（比如这里的IRepository\<Task\>）来执行常见任务。这里使用IRepository.GetAll()返回一个IQuerybale来查询实体。
* **WhereIf**是ABP的扩展方法，用于简化IQueryable.Where方法的使用条件。
* **ObjectMapper**（其中一些来自ApplicationService基类并默认通过AutoMapper实现）用于将Task对象列表映射到TaskListDtos对象列表。

### 测试TaskAppService

在创建用户界面前，我想先测试一下TaskAppService。如果你对自动化测试不感兴趣，可以跳过本节。

开始模板包含用于测试代码的 **.Tests** 项目。它使用EF Core提供的内存数据库代替SQL Server。因此我们的单元测试可以不依赖真实的数据库工作。它为每次测试创建单独的数据库。因此测试是彼此隔离的。开始测试前我们先使用TestDataBuilder类在内存数据库中添加一些初始测试数据。对TestDataBuilder作如下修改：

``` C#
using Acme.SimpleTaskApp.EntityFrameworkCore;
using Acme.SimpleTaskApp.Tasks;

namespace Acme.SimpleTaskApp.Tests.TestDatas
{
    public class TestDataBuilder
    {
        private readonly SimpleTaskAppDbContext _context;

        public TestDataBuilder(SimpleTaskAppDbContext context)
        {
            _context = context;
        }

        public void Build()
        {
            //在此处创建测试数据
            _context.Tasks.AddRange(
                new Task("跟着小白兔") { },
                new Task("打扫房间") { State = TaskState.已完成 }
            );
        }
    }
}
```

你可以阅读本项目的源码，理解TestDataBuilder的路径和用法。我在数据库中添加了两条任务（其中一条已完成）。因此我可以通过假设数据库中存在两条任务来编写测试。我的第一个集成测试就用来测试我们之前创建的TaskAppService.GetAll方法。

``` C#
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Tasks.Dtos;
using Shouldly;
using Xunit;

namespace Acme.SimpleTaskApp.Tests.Tasks
{
    public class TaskAppService_Tests : SimpleTaskAppTestBase
    {
        private readonly ITaskAppService _taskAppService;

        public TaskAppService_Tests()
        {
            _taskAppService = Resolve<ITaskAppService>();
        }

        [Fact]
        public async System.Threading.Tasks.Task Should_Get_All_Tasks()
        {
            // 行为
            var getAllTasksInput = new GetAllTasksInput();
            var output = await _taskAppService.GetAll(getAllTasksInput);

            // 正确结果
            output.Items.Count.ShouldBe(2);
        }

        [Fact]
        public async System.Threading.Tasks.Task Should_Get_Filtered_Tasks()
        {
            // 行为
            var getAllTasksInput = new GetAllTasksInput { State = TaskState.打开 };
            var output = await _taskAppService.GetAll(getAllTasksInput);

            // 正确结果
            output.Items.ShouldAllBe(t => t.State == TaskState.打开);
        }
    }
}

```

如上，我创建了两种不同的测试来测试GetAll方法。现在我可以打开资源管理器（VS菜单栏中的测试=》Windows=》测试资源管理器）来运行单元测试。

![测试资源管理器](/img/Introduction.Net.Core.EFCore.P1/test-explorer.png)

测试都已通过。最后一个是开始模板的预生成测试，我们可以忽略它。

请注意：ABP开始模板使用[xUnit](https://xunit.github.io/)和[Shouldly](http://shouldly.readthedocs.io/en/latest/)代替默认设置。因此我们可以使用它们来编写我们的测试。

### 任务列表视图

现在我确定TaskAppService可以正确的工作，我可以开始创建任务列表页面了。

#### 添加菜单选项

首先，我在顶部菜单栏添加一条选项：

``` C#
using Abp.Application.Navigation;
using Abp.Localization;

namespace Acme.SimpleTaskApp.Web.Startup
{
    /// <summary>
    /// 定义网页菜单
    /// </summary>
    public class SimpleTaskAppNavigationProvider : NavigationProvider
    {
        public override void SetNavigation(INavigationProviderContext context)
        {
            context.Manager.MainMenu
                .AddItem(
                    new MenuItemDefinition(
                        PageNames.Home,
                        L("HomePage"),
                        url: "",
                        icon: "fa fa-home"
                        )
                ).AddItem(
                    new MenuItemDefinition(
                        PageNames.About,
                        L("About"),
                        url: "Home/About",
                        icon: "fa fa-info"
                        )
                ).AddItem(
                new MenuItemDefinition(
                    PageNames.TaskList,
                    L("TaskList"),
                    url: "Tasks",
                    icon: "fa fa-tasks"
                    )
                );
        }

        private static ILocalizableString L(string name)
        {
            return new LocalizableString(name, SimpleTaskAppConsts.LocalizationSourceName);
        }
    }
}

```
之前介绍过，开始模板带有两个页面：主页和关于页。我们可以修改它们或新建页面。我更喜欢暂时不管它们而去新建一个菜单选项。

#### 创建Task控制器和视图模型

我在 **.Web**项目中新建了一个控制器类**TasksController**，如下所示：

``` C#
public class TasksController : <strong>SimpleTaskAppControllerBase</strong>
{
    private readonly ITaskAppService _taskAppService;

    public TasksController(<strong>ITaskAppService taskAppService</strong>)
    {
        _taskAppService = taskAppService;
    }

    public async Task<ActionResult> Index(GetAllTasksInput input)
    {
        var output = await _taskAppService.GetAll(input);
        var model = new <strong>IndexViewModel</strong>(output.Items);
        return View(model);
    }
}
```

* 它是由（由AbpController派生的）**SimpleTaskAppControllerBase**派生而来，包含一些控制器常用基础代码。
* 我注入**ITaskAppService**来获取任务列表数据。
* 我没有直接将GetAll方法的结果传递给视图，而是在.Web项目中创建了一个IndexViewModel类，如下所示：

``` C#
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Tasks.Dtos;
using System.Collections.Generic;

namespace Acme.SimpleTaskApp.Web.Models.Tasks
{
    public class IndexViewModel
    {
        public IReadOnlyList<TaskListDto> Tasks { get; }

        public IndexViewModel(IReadOnlyList<TaskListDto> tasks)
        {
            Tasks = tasks;
        }

        public string GetTaskLabel(TaskListDto task)
        {
            switch (task.State)
            {
                case TaskState.打开:
                    return "lable-success ";
                default:
                    return "lable-default ";
            }
        }
    }
}

```

这个简单视图模型在它的构造函数中获取（ITaskAppService提供的）任务列表。它还有GetTaskLabel方法，用于在视图中为给定任务选择Bootstrap标签类。

#### 任务列表页面

最终，Index视图如下所示：

``` html
@model Acme.SimpleTaskApp.Web.Models.Tasks.IndexViewModel

@{
    ViewBag.Title = L("TaskList");
    ViewBag.ActiveMenu = "TaskList"; //匹配SimpleTaskAppNavigationProvider中的菜单名来高亮当前菜单选项
}

<h2>@L("TaskList")</h2>

<div class="row">
    <div>
        <ul class="list-group" id="TaskList">
            @foreach (var task in Model.Tasks)
            {
                <li class="list-group-item">
                    <span class="pull-right">@L($"TaskState_{task.State}")</span>
                    <h4 class="list-group-item-heading">@task.Title</h4>
                    <div class="list-group-item-text">
                        @task.CreationTime.ToString("yyyy-MM-dd HH:mm:ss")
                    </div>
                </li>
            }
        </ul>
    </div>
</div>
```

我们使用传递的模型通过BootStrap的[List Group](http://getbootstrap.com/components/#list-group)组件来渲染视图。在这里，我们使用IndexViewModel.GetTaskLabel()方法来获取任务的标签类型。渲染后的页面如下：

![任务列表](/img/Introduction.Net.Core.EFCore.P1/task-list-view-1.png)

#### 本地化

我们在视图中使用ABP框架带有的**L**方法。它被用来本地化字符串。我们在 **.Core**项目的**Localization/Source**文件夹中用 **.json**文件定义本地字符串。英语本地化如下：

``` json
{
  "culture": "en",
  "texts": {
    "HelloWorld": "Hello World!",
    "ChangeLanguage": "Change language",
    "HomePage": "HomePage",
    "About": "About",
    "Home_Description": "Welcome to SimpleTaskApp...",
    "About_Description": "This is a simple startup template to use ASP.NET Core with ABP framework.",
    "TaskList": "Task List",
    "TaskState_打开": "Open",
    "TaskState_已完成": "Completed"
  }
}
```
开始模板带有大部分内容，可以删除。我添加的最后三行是之前视图中会用到的内容。ABP的本地化用法非常简单，你可以在[本地化文档](http://www.aspnetboilerplate.com/Pages/Documents/Localization)中查看更多本地化系统的信息。


#### 过滤任务

上面讲过，TaskController使用了GetAllTasksInput，可以用来过滤任务。因此，我们可以在任务列表视图中添加一个下拉框来过滤任务。首先，在视图中添加一个下拉框（我加在了header中）：

``` html
<h2>
    @L("TaskList")
    <span class="pull-right">
        @Html.DropDownListFor(
       model => model.SelectedTaskState,
       Model.GetTasksStateSelectListItems(LocalizationManager),
       new
            {
           @class = "form-control",
           id = "TaskStateCombobox"
       })
    </span>
</h2>
```
然后修改**IndexViewModel**，添加**SelectedTaskState**属性和**GetTasksStateSelectListItems**方法：

``` C#
using Abp.Localization;
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Tasks.Dtos;
using Microsoft.AspNetCore.Mvc.Rendering;
using System;
using System.Collections.Generic;
using System.Linq;

namespace Acme.SimpleTaskApp.Web.Models.Tasks
{
    public class IndexViewModel
    {
        public IReadOnlyList<TaskListDto> Tasks { get; }

        public TaskState? SelectedTaskState { get; set; }

        public IndexViewModel(IReadOnlyList<TaskListDto> tasks)
        {
            Tasks = tasks;
        }

        public List<SelectListItem> GetTasksStateSelectListItems(ILocalizationManager localizationManager)
        {
            var list = new List<SelectListItem>
            {
                new SelectListItem{
                    Text =localizationManager.GetString(SimpleTaskAppConsts.LocalizationSourceName,"AllTasks"),
                    Value ="",
                    Selected =SelectedTaskState==null
                }
            };

            list.AddRange(Enum.GetValues(typeof(TaskState))
                .Cast<TaskState>()
                .Select(state => new SelectListItem
                {
                    Text=localizationManager.GetString(SimpleTaskAppConsts.LocalizationSourceName,$"TaskState_{state}"),
                    Value=state.ToString(),
                    Selected=state==SelectedTaskState
                })
                );
            return list;
        }


        public string GetTaskLabel(TaskListDto task)
        {
            switch (task.State)
            {
                case TaskState.打开:
                    return "lable-success ";
                default:
                    return "lable-default ";
            }
        }
    }
}

```

我们应在控制器中设定SelectedTaskState：

``` C#
public async Task<ActionResult> Index(GetAllTasksInput input)
{
    var output = await _taskAppService.GetAll(input);
    var model = new IndexViewModel(output.Items)
    {
        <strong>SelectedTaskState = input.State</strong>
    };
    return View(model);
}
```

现在，我们可以运行应用，在视图右上角看到下拉框了：

![任务列表](/img/Introduction.Net.Core.EFCore.P1/task-list-view-2.png)

我添加了下拉框但还不能用。我需要写一些JavaScript代码实现改变下拉框选中项时刷新页面的功能。所以我在 **.Web**项目中创建了wwwroot\js\views\tasks\\**index.js** 文件：

```
(function ($) {
    $(function () {
        var _$taskStateCombobox = $('#TaskStateCombobox');

        _$taskStateCombobox.change(function () {
            location.href = 'Tasks?state=' + _$taskStateCombobox.val();
        });
    });
})(jQuery);
```

在把JavaScript文件添加到视图之前，我使用了VS插件 [Bundler & Minifier](https://github.com/madskristensen/BundlerMinifier)（它在ASP.NET Core项目中是默认缩小文件的方法）缩小它：

![缩小js文件](/img/Introduction.Net.Core.EFCore.P1/minify-js.png)

它会在 .Web项目的**bundleconfig.json**中加入下面的信息：

``` js
  {
    "outputFileName": "wwwroot/js/views/tasks/index.min.js",
    "inputFiles": [
      "wwwroot/js/views/tasks/index.js"
    ]
  }
```

并创建一个迷你版的script：

![迷你js文件](/img/Introduction.Net.Core.EFCore.P1/minified-js.png)

挡我修改index.js时，index.min.js会自动重构。现在我可以把它加进页面里了：


``` xml
@section scripts{
    <environment names="Development">
        <script src="~/js/views/tasks/index.js"></script>
    </environment>

    <environment names="Staging,Production">
        <script src="~/js/views/tasks/index.min.js"></script>
    </environment>
}
```
通过这样的代码，视图将会在开发环境中调用index.js，在生产环境中调用index.min.js（迷你版）。这是ASP.NET Core MVC项目中的常用方法。

###  自动测试任务列表页

ASP.NET Core MVC基础结构也集成了集成测试以便我们创建它。因此我们可以完整测试我们的服务端代码。如果你对自动化测试不感兴趣，可以跳过本节。

ABP开始模板含有 **.Web.Tests**项目可以用来测试。我创建一个简单的测试来请求**TaskController.Index**，并查看响应。

``` C#
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Web.Controllers;
using Shouldly;
using Xunit;

namespace Acme.SimpleTaskApp.Web.Tests.Controllers
{
    public class TasksController_Tests : SimpleTaskAppWebTestBase
    {
        [Fact]
        public async System.Threading.Tasks.Task Should_Get_Tasks_By_State()
        {
            // 行为
            var response = await GetResponseAsStringAsync(
                GetUrl<TasksController>(nameof(TasksController.Index), new
                {
                    state = TaskState.打开
                })
                );

            // 正确结果
            response.ShouldNotBeNullOrWhiteSpace();
        }
    }
}
```

**GetResponseAsStringAsync**和**GetUrl**方法是ABP的**AbpAspNetCoreIntegratedTestBase**类提供的辅助方法。我们可以直接使用Client（HttpClient的一个实例）属性发出请求。但使用这些快捷方法可以更简单。有关更多信息，请查看ASP.NET Core集成测试文档。

在我调试测试时，可以看到响应HTML：

![网页测试](/img/Introduction.Net.Core.EFCore.P1/web-test-1.png)

这表明Index页返回了没有异常的响应。但我们也许想继续请求，检查是否能返回正确的HTML。这里有一些库可以用来转换HTML，比如ABP开始模板的.Web.Tests项目中包含的[AngleSharp](https://anglesharp.github.io/)。因此我用它来检查返回的HTML代码：

``` C#
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Web.Controllers;
using AngleSharp.Html.Parser;
using Microsoft.EntityFrameworkCore;
using Shouldly;
using System.Linq;
using Xunit;

namespace Acme.SimpleTaskApp.Web.Tests.Controllers
{
    public class TasksController_Tests : SimpleTaskAppWebTestBase
    {
        [Fact]
        public async System.Threading.Tasks.Task Should_Get_Tasks_By_State()
        {
            // 行为
            var response = await GetResponseAsStringAsync(
                GetUrl<TasksController>(nameof(TasksController.Index), new
                {
                    state = TaskState.打开
                })
                );

            // 正确结果
            response.ShouldNotBeNullOrWhiteSpace();

            // 从数据库获取任务数据
            var tasksInDatabase = await UsingDbContextAsync(async dbContext =>
            {
                return await dbContext.Tasks
                .Where(t => t.State == TaskState.打开)
                .ToListAsync();
            });

            // 转换HTML响应来检查数据库中的任务数据是否返回
            var document = new HtmlParser().ParseDocument(response);
            var listItems = document.QuerySelectorAll("#TaskList li");

            // 检查任务数量
            listItems.Length.ShouldBe(tasksInDatabase.Count);

            // 检查是否返回正确的列表
            foreach (var item in listItems)
            {
                var header = item.QuerySelector(".list-group-item-heading");

                var taskTitle = header.InnerHtml.Trim();
                tasksInDatabase.Any(t => t.Title == taskTitle).ShouldBeTrue();
            }
        }
    }
}
```

你可以更进一步测试HTML。但通常检查基本标签就足够了。

## 第二部分

参见使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用 - 第二部分

## 源代码

在这里获取最新的源代码

## 编辑历史

* **2019-03-09**：翻译中文。