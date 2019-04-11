# ASP.NET Core与Entity Framework Core介绍 - 第一部分

> 从[GitHub仓库](https://github.com/aspnetboilerplate/aspnetboilerplate-samples/tree/master/SimpleTaskSystem-Core)获取源码

## 简介

本文是“使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用”系列文章的第二部分，查看其他部分：

* [使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用 - 第一部分](./Introduction.Net.Core.EFCore.P1.md)
* 使用ASP.NET Core,Entity Framework Core和ABP框架创建多层网页应用 - 第二部分（本文）

## 开发应用

### 创建个人实体

我将个人的概念添加到应用中以将任务分配给个人。因此我定义了一个Person实体：

``` C#
using Abp.Domain.Entities.Auditing;
using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace Acme.SimpleTaskApp.People
{
    [Table("AppPersons")]
    public class Person : AuditedEntity<Guid>
    {
        public const int MaxNameLength = 32;

        [Required]
        [StringLength(MaxNameLength)]
        public string Name { get; set; }

        public Person() { }

        public Person(string name)
        {
            Name = name;
        }
    }
}
```

这一次作为示范，我将ID（主键）类型设定为Guid。我还从AuditedEntity（具有CreationTime，CreaterUserId，LastModificationTime和LastModifierUserId属性）派生而没有用基础实体类。

### 关联Person到Task实体

我还在Task实体中添加了**AssignedPerson**属性：
``` C#
    public class Task : Entity, IHasCreationTime
    {
        // ...

        [ForeignKey(nameof(AssignedPersonId))]
        public Person AssignedPerson { get; set; }
        public Guid? AssignedPersonId { get; set; }

        public Task(string title, string description = null, Guid? assignedPersonId = null)
            : this()
        {
            Title = title;
            Description = description;
            AssignedPersonId = assignedPersonId;
        }
         }
```

AssignedPerson是**可选的**。因此任务可以指定给个人，也可以不指定。

### 添加Person到DbContext

最后，把新的Person实体添加到DbContext中：

``` C#
public class SimpleTaskAppDbContext : AbpDbContext
    {
        // ...

        public DbSet<Person> People { get; set; }

        // ...
    }
```

### 为Person实体添加新的迁移

在**程序包管理器控制台**中执行下面的命令：

![添加个人迁移](/img/Introduction.Net.Core.EFCore.P2/migration-add-person-2.png)

它会在项目中创建一个新的迁移类：

``` C#
namespace Acme.SimpleTaskApp.Migrations
{
    public partial class Added_Person : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.AddColumn<Guid>(
                name: "AssignedPersonId",
                table: "AppTasks",
                nullable: true);

            migrationBuilder.CreateTable(
                name: "AppPersons",
                columns: table => new
                {
                    Id = table.Column<Guid>(nullable: false),
                    CreationTime = table.Column<DateTime>(nullable: false),
                    CreatorUserId = table.Column<long>(nullable: true),
                    LastModificationTime = table.Column<DateTime>(nullable: true),
                    LastModifierUserId = table.Column<long>(nullable: true),
                    Name = table.Column<string>(maxLength: 32, nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_AppPersons", x => x.Id);
                });

            migrationBuilder.CreateIndex(
                name: "IX_AppTasks_AssignedPersonId",
                table: "AppTasks",
                column: "AssignedPersonId");

            migrationBuilder.AddForeignKey(
                name: "FK_AppTasks_AppPersons_AssignedPersonId",
                table: "AppTasks",
                column: "AssignedPersonId",
                principalTable: "AppPersons",
                principalColumn: "Id",
                onDelete: ReferentialAction.Restrict);
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropForeignKey(
                name: "FK_AppTasks_AppPersons_AssignedPersonId",
                table: "AppTasks");

            migrationBuilder.DropTable(
                name: "AppPersons");

            migrationBuilder.DropIndex(
                name: "IX_AppTasks_AssignedPersonId",
                table: "AppTasks");

            migrationBuilder.DropColumn(
                name: "AssignedPersonId",
                table: "AppTasks");
        }
    }
}
```

我把ReferentialAction.Restrict改成了ReferentialAction.SetNull。它的作用是：如果我删除了一个人，那么指派给这个人的任务会变成未指派。对这个Demo来说并不重要。但是我想告诉你可以根据需要修改迁移代码。实际上，在应用迁移到数据库之前你总是应该审阅生成的代码。之后，我们可以**应用迁移**到数据库：

![Update-Database](/img/Introduction.Net.Core.EFCore.P2/migration-person-update-2.png)

打开数据库，可以看到添加的表和列，我们可以添加一些测试数据。

![个人表](/img/Introduction.Net.Core.EFCore.P2/person-table.png)

我添加了一条个人数据并指派给第一个任务：

![任务表](/img/Introduction.Net.Core.EFCore.P2/tasks-table-with-person.png)

### 在任务列表返回指派人

修改TaskAppService来返回指派人信息。首先，


<p>I&#39;ll change the <strong>TaskAppService</strong> to return assigned person information. First, I&#39;m adding two properties to <strong>TaskListDto</strong>:</p>

<pre lang="cs">
[AutoMapFrom(typeof(Task))]
public class TaskListDto : EntityDto, IHasCreationTime
{
    //...

<strong>    public Guid? AssignedPersonId { get; set; }

    public string AssignedPersonName { get; set; }</strong>
}</pre>

<p>And including the Task.AssignedPerson property to the query. Just added the <strong>Include</strong> line:</p>

<pre lang="cs">
public class TaskAppService : SimpleTaskAppAppServiceBase, ITaskAppService
{
    //...

    public async Task<ListResultDto<TaskListDto>> GetAll(GetAllTasksInput input)
    {
        var tasks = await _taskRepository
            .GetAll()
            <strong>.Include(t => t.AssignedPerson)</strong>
            .WhereIf(input.State.HasValue, t => t.State == input.State.Value)
            .OrderByDescending(t => t.CreationTime)
            .ToListAsync();

        return new ListResultDto<TaskListDto>(
            ObjectMapper.Map<List<TaskListDto>>(tasks)
        );
    }
}</pre>

<p>Thus, GetAll method will return Assigned person information with the tasks. Since we used AutoMapper, new properties will also be copied to DTO automatically.</p>

<h3 id="ArticleUnitTestAssignedPerson">Change Unit Test to Test Assigned Person</h3>

<p>At this point, we can change unit tests to see if assigned people are retrieved while getting the task list. First, I changed initial test data in the TestDataBuilder class to assign a person to a task:</p>

<pre lang="cs">
public class TestDataBuilder
{
    //...

    public void Build()
    {
<strong>        var neo = new Person("Neo");
        _context.People.Add(neo);
        _context.SaveChanges();</strong>

        _context.Tasks.AddRange(
            new Task("Follow the white rabbit", "Follow the white rabbit in order to know the reality.", <strong>neo.Id</strong>),
            new Task("Clean your room") { State = TaskState.Completed }
            );
    }
}</pre>

<p>Then I&#39;m changing TaskAppService_Tests.Should_Get_All_Tasks() method to check if one of the retrieved tasks has a person assigned (see the last line added):</p>

<pre lang="cs">
[Fact]
public async System.Threading.Tasks.Task Should_Get_All_Tasks()
{
    //Act
    var output = await _taskAppService.GetAll(new GetAllTasksInput());

    //Assert
    output.Items.Count.ShouldBe(2);
    <strong>output.Items.Count(t => t.AssignedPersonName != null).ShouldBe(1);
</strong>}</pre>

<p>Note: Count extension method requires <em>using System.Linq;</em> statement.</p>

<h3 id="ArticleShowPersonInTaskList">Show Assigned Person Name in the Task List Page</h3>

<p>Finally, we can change <strong>Tasks\Index.cshtml</strong> to show <strong> AssignedPersonName</strong>:</p>

<pre lang="xml">
@foreach (var task in Model.Tasks)
{
    <li class="list-group-item">
        <span class="pull-right label label-lg @Model.GetTaskLabel(task)">@L($"TaskState_{task.State}")</span>
        <h4 class="list-group-item-heading">@task.Title</h4>
        <div class="list-group-item-text">
            @task.CreationTime.ToString("yyyy-MM-dd HH:mm:ss")<strong> | @(task.AssignedPersonName ?? L("Unassigned"))</strong>
        </div>
    </li>
}</pre>

<p>When we run the application, we can see it in the task list:</p>

<p><img alt="Task list with person name" height="311" src="task-list-assigned-person.png" width="446" /></p>

<h3 id="ArticleTaskCreateService">New Application Service Method for Task Creation</h3>

<p>We can list tasks, but we don&#39;t have a <strong>task creation page</strong> yet. First, adding a <strong>Create</strong> method to the <strong>ITaskAppService</strong> interface:</p>

<pre lang="cs">
public interface ITaskAppService : IApplicationService
{
    //...

    System.Threading.Tasks.Task Create(CreateTaskInput input);
}</pre>

<p>And implementing it in TaskAppService class:</p>

<pre lang="cs">
public class TaskAppService : SimpleTaskAppAppServiceBase, ITaskAppService
{
    private readonly IRepository<Task> _taskRepository;

    public TaskAppService(IRepository<Task> taskRepository)
    {
        _taskRepository = taskRepository;
    }

    //...

<strong>    public async System.Threading.Tasks.Task Create(CreateTaskInput input)
    {
        var task = ObjectMapper.Map<Task>(input);
        await _taskRepository.InsertAsync(task);
    }</strong>
}</pre>

<p>Create method automatically <strong>maps</strong> given input to a Task entity and inserting to the database using the repository. <strong>CreateTaskInput</strong> <a href="http://www.aspnetboilerplate.com/Pages/Documents/Data-Transfer-Objects" target="_blank">DTO</a> is like that:</p>

<pre lang="cs">
using System;
using System.ComponentModel.DataAnnotations;
using Abp.AutoMapper;

namespace Acme.SimpleTaskApp.Tasks.Dtos
{
    <strong>[AutoMapTo(typeof(Task))]</strong>
    public class CreateTaskInput
    {
        [Required]
        [StringLength(<strong>Task.MaxTitleLength</strong>)]
        public string Title { get; set; }

        [StringLength(<strong>Task.MaxDescriptionLength</strong>)]
        public string Description { get; set; }

        public Guid? AssignedPersonId { get; set; }
    }
}</pre>

<p>Configured to map it to Task entity (using AutoMapTo attribute) and added data annotations to apply <a href="http://www.aspnetboilerplate.com/Pages/Documents/Validating-Data-Transfer-Objects" target="_blank"> validation</a>. We used constants from Task entity to use same max lengths.</p>

<h3 id="ArticleTaskCreateServiceTest">Testing Task Creation Service</h3>

<p>I&#39;m adding some integration tests into TaskAppService_Tests class to test the Create method:</p>

<pre lang="cs">
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Tasks.Dtos;
using Shouldly;
using Xunit;
using System.Linq;
using Abp.Runtime.Validation;

namespace Acme.SimpleTaskApp.Tests.Tasks
{
    public class TaskAppService_Tests : SimpleTaskAppTestBase
    {
        private readonly ITaskAppService _taskAppService;

        public TaskAppService_Tests()
        {
            _taskAppService = Resolve<ITaskAppService>();
        }

        //...

        [Fact]
        public async System.Threading.Tasks.Task <strong>Should_Create_New_Task_With_Title</strong>()
        {
            await _taskAppService.Create(new CreateTaskInput
            {
                Title = "Newly created task #1"
            });

            UsingDbContext(context =>
            {
                var task1 = context.Tasks.FirstOrDefault(t => t.Title == "Newly created task #1");
                <strong>task1.ShouldNotBeNull();</strong>
            });
        }

        [Fact]
        public async System.Threading.Tasks.Task <strong>Should_Create_New_Task_With_Title_And_Assigned_Person</strong>()
        {
            <strong>var neo = UsingDbContext(context => context.People.Single(p => p.Name == "Neo"));
</strong>
            await _taskAppService.Create(new CreateTaskInput
            {
                Title = "Newly created task #1",
                <strong>AssignedPersonId = neo.Id</strong>
            });

            UsingDbContext(context =>
            {
                var task1 = context.Tasks.FirstOrDefault(t => t.Title == "Newly created task #1");
                task1.ShouldNotBeNull();
                <strong>task1.AssignedPersonId.ShouldBe(neo.Id);</strong>
            });
        }

        [Fact]
        public async System.Threading.Tasks.Task <strong>Should_Not_Create_New_Task_Without_Title</strong>()
        {
            await <strong>Assert.ThrowsAsync<AbpValidationException></strong>(async () =>
            {
                await _taskAppService.Create(new CreateTaskInput
                {
                    Title = null
                });
            });
        }
    }
}</pre>

<p>First test creates a task with a <strong>title</strong>, second one creates a task with a <strong>title</strong> and <strong>assigned person</strong>, the last one tries to create an <strong>invalid</strong> task to show the <strong> exception</strong> case.</p>

<h3 id="ArticleTaskCreatePage">Task Creation Page</h3>

<p>We know that TaskAppService.Create is properly working. Now, we can create a page to add a new task. Final page will be like that:</p>

<p><img alt="Create task page" height="450" src="create-task-page-view.png" width="742" /></p>

<p>First, I added a <strong>Create</strong> action to the TaskController in order to prepare the page above:</p>

<pre lang="cs">
using System.Threading.Tasks;
using Abp.Application.Services.Dto;
using Acme.SimpleTaskApp.Tasks;
using Acme.SimpleTaskApp.Tasks.Dtos;
using Acme.SimpleTaskApp.Web.Models.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Rendering;
using System.Linq;
using Acme.SimpleTaskApp.Common;
using Acme.SimpleTaskApp.Web.Models.People;

namespace Acme.SimpleTaskApp.Web.Controllers
{
    public class TasksController : SimpleTaskAppControllerBase
    {
        private readonly ITaskAppService _taskAppService;
        private readonly ILookupAppService _lookupAppService;

        public TasksController(
            ITaskAppService taskAppService,
            <strong>ILookupAppService lookupAppService</strong>)
        {
            _taskAppService = taskAppService;
            _lookupAppService = lookupAppService;
        }

        //...
        
<strong>        public async Task<ActionResult> Create()
        {
            var peopleSelectListItems = (await _lookupAppService.GetPeopleComboboxItems()).Items
                .Select(p => p.ToSelectListItem())
                .ToList();

            peopleSelectListItems.Insert(0, new SelectListItem { Value = string.Empty, Text = L("Unassigned"), Selected = true });

            return View(new CreateTaskViewModel(peopleSelectListItems));
        }</strong>
    }
}</pre>

<p>I injected ILookupAppService that is used to get people combobox items. While I could directly inject and use IRepository<Person, Guid> here, I preferred this to make a better layering and re-usability. ILookupAppService.GetPeopleComboboxItems is defined in application layer as shown below:</p>

<pre lang="cs">
public interface ILookupAppService : IApplicationService
{
    Task<ListResultDto<ComboboxItemDto>> GetPeopleComboboxItems();
}

public class LookupAppService : SimpleTaskAppAppServiceBase, ILookupAppService
{
    private readonly IRepository<Person, Guid> _personRepository;

    public LookupAppService(<strong>IRepository<Person, Guid> personRepository</strong>)
    {
        _personRepository = personRepository;
    }

<strong>    public async Task<ListResultDto<ComboboxItemDto>> GetPeopleComboboxItems()
    {
        var people = await _personRepository.GetAllListAsync();
        return new ListResultDto<ComboboxItemDto>(
            people.Select(p => new ComboboxItemDto(p.Id.ToString("D"), p.Name)).ToList()
        );
    }</strong>
}</pre>

<p><strong>ComboboxItemDto</strong> is a simple class (defined in ABP) to transfer a combobox item data. TaskController.Create method simply uses this method and converts the returned list to a list of <strong>SelectListItem</strong> (defined in AspNet Core) and passes to the view using CreateTaskViewModel class:</p>

<pre lang="cs">
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc.Rendering;

namespace Acme.SimpleTaskApp.Web.Models.People
{
    public class CreateTaskViewModel
    {
        <strong>public List<SelectListItem> People { get; set; }</strong>

        public CreateTaskViewModel(List<SelectListItem> people)
        {
            People = people;
        }
    }
}</pre>

<p>Create view is shown below:</p>

<pre lang="xml">
@using Acme.SimpleTaskApp.Web.Models.People
@model CreateTaskViewModel

@section scripts
{
    <environment names="Development">
        <script src="~/js/views/tasks/create.js"></script>
    </environment>

    <environment names="Staging,Production">
        <script src="~/js/views/tasks/create.min.js"></script>
    </environment>
}

<h2>
    @L("NewTask")
</h2>

<form id="TaskCreationForm">
    
<strong>    <div class="form-group">
        <label for="Title">@L("Title")</label>
        <input type="text" name="Title" class="form-control" placeholder="@L("Title")" required maxlength="@Acme.SimpleTaskApp.Tasks.Task.MaxTitleLength">
    </div>

    <div class="form-group">
        <label for="Description">@L("Description")</label>
        <input type="text" name="Description" class="form-control" placeholder="@L("Description")" maxlength="@Acme.SimpleTaskApp.Tasks.Task.MaxDescriptionLength">
    </div>

    <div class="form-group">
        @Html.Label(L("AssignedPerson"))
        @Html.DropDownList(
            "AssignedPersonId",
            Model.People,
            new
            {
                @class = "form-control",
                id = "AssignedPersonCombobox"
            })
    </div></strong>

    <button type="submit" class="btn btn-default">@L("Save")</button>

</form></pre>

<p>I included <strong>create.js</strong> defined like that:</p>

<pre lang="js">
(function($) {
    $(function() {

        var _$form = $(&#39;<strong>#TaskCreationForm</strong>&#39;);

        _$form.find(&#39;input:first&#39;).focus();

        _$form.<strong>validate()</strong>;

        _$form.find(&#39;button[type=submit]&#39;)
            .click(function(e) {
                e.preventDefault();

                if (<strong>!_$form.valid()</strong>) {
                    return;
                }

                var input = <strong>_$form.serializeFormToObject()</strong>;
                <strong>abp.services.app.task.create</strong>(input)
                    .done(function() {
                        location.href = &#39;/Tasks&#39;;
                    });
            });
    });
})(jQuery);</pre>

<p>Let&#39;s see what&#39;s done in this JavaScript code:</p>

<ul>
	<li>Prepares <strong>validatation</strong> for the form (using <a href="https://jqueryvalidation.org/" target="_blank">jquery validation</a> plugin) and validates it on Save button&#39;s click.</li>
	<li>Uses <strong>serializeFormToObject</strong> jquery plugin (defined in jquery-extensions.js in the solution) to convert form data to a JSON object (I included jquery-extensions.js to the _Layout.cshtml as the last script file).</li>
	<li>Uses <strong>abp.services.task.create</strong> method to call TaskAppService.Create method. This is one of the important features of ABP. We can use application services from JavaScript code just like calling a JavaScript method in our code. See <a href="http://www.aspnetboilerplate.com/Pages/Documents/AspNet-Core#application-services-as-controllers"> details</a>.</li>
</ul>

<p>
Here is the content of jquery-extensions.js:
</p>
<pre lang="js">
(function ($) {
    //serializeFormToObject plugin for jQuery
    $.fn.serializeFormToObject = function () {
        //serialize to array
        var data = $(this).serializeArray();

        //add also disabled items
        $(&#x27;:disabled[name]&#x27;, this)
            .each(function () {
                data.push({ name: this.name, value: $(this).val() });
            });

        //map to object
        var obj = {};
        data.map(function (x) { obj[x.name] = x.value; });

        return obj;
    };
})(jQuery);
</pre>

<p>Finally, I added an "Add Task" button to the <em>task list</em> page in order to navigate to the <em>task creation page</em>:</p>

<pre lang="xml">
<a class="btn btn-primary btn-sm" asp-action="<strong>Create</strong>">@L("AddNew")</a></pre>

<h2 id="ArticleRemoveHomeAndAbout">Remove Home and About Page</h2>

<p>We can remove Home and About page from the application if we don&#39;t need. To do that, first change <strong>HomeController</strong> like that:</p>

<pre lang="cs">
using Microsoft.AspNetCore.Mvc;

namespace Acme.SimpleTaskApp.Web.Controllers
{
    public class HomeController : SimpleTaskAppControllerBase
    {
        public ActionResult Index()
        {
            <strong>return RedirectToAction("Index", "Tasks");</strong>
        }
    }
}</pre>

<p>Then delete <strong>Views/Home</strong> folder and remove menu items from SimpleTaskApp<strong>NavigationProvider</strong> class. You can also remove unnecessary keys from localization JSON files.</p>


<h2 id="ArticleSourceCode">Source Code</h2>

<p>You can get the latest source code here <a href="https://github.com/aspnetboilerplate/aspnetboilerplate-samples/tree/master/SimpleTaskSystem-Core">https://github.com/aspnetboilerplate/aspnetboilerplate-samples/tree/master/SimpleTaskSystem-Core</a></p>

	
<h2 id="ArticleHistory" class="abp-invisible">Article History</h2>

<ul class="abp-invisible">
	<li><strong>2018-02-14</strong>: Upgraded source code to ABP v3.4 and updated the download link.</li>
	<li><strong>2017-07-30</strong>: Replaced ListResultOutput by ListResultDto in the article.</li>
	<li><strong>2017-06-02</strong>: Changed article and solution to support .net core.</li>
	<li><strong>2016-08-09</strong>: Revised article based on feedbacks.</li>
	<li><strong>2016-08-08</strong>: Initial publication.</li>
</ul>