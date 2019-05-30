## 授权

### 介绍

几乎所有的企业应用都会在某种程度上使用授权。授权用于检查是否允许用户执行应用中的某些操作。ABP定义了一个**基于权限**的基础设施来实现授权。

#### 关于IPermissionChecker

授权系统使用**IPermissionChecker**来检查权限。在**Module Zero**项目中已经完全实现了它，不过你也可以用自己的方式实现它。若未实现它，将会使用NullPermissionChecker向所有用户授予完全权限。

### 定义权限

唯一**权限**用来定义每个需要授权的操作。权限在使用前需要先定义。ABP被设计为[模块化](/Overall/Module-System)，因此不同的模块可以拥有不同的权限。模块应该创建一个派生自**AuthorizationProvider**的类来定义权限。示例如下：

``` C#
    public class MyAuthorizationProvider : AuthorizationProvider
    {
        public override void SetPermissions(IPermissionDefinitionContext context)
        {
            var administration = context.CreatePermission("Administration");

            var userManagement = administration.CreateChildPermission("Administration.UserManagement");
            userManagement.CreateChildPermission("Administration.UserManagement.CreateUser");

            var roleManagement = administration.CreateChildPermission("Administration.RoleManagement");
        }
    }
```                

**IPermissionDefinitionContext**提供了创建和获取权限的方法。

权限定义了这些属性：

* **Name**：系统级唯一的名称。最好为权限名定义一个常量字符串，而不是变量字符串。我们倾向使用“.”符号用于有层次的名称，但这不是强制的。你可以按自己的喜好命名，唯一要求是名称必须唯一。
* **Display name**:用于在用户界面显示权限名称的本地字符串。
* **Description**：用于在用户界面显示权限描述的本地字符串。
* **MultiTenancySides**：对于多租户应用，租户和租主可以使用同一个权限，因为它是一个**Flags**枚举。
* **featureDependency**：可以为[功能](/Application.Layer/Feature-Management)声明依赖。因此可以仅在满足功能依赖时才能授予权限。它等待实现IFeatureDependency的对象。默认实现是SimpleFeatureDependency类。用法如下：`new SimpleFeatureDependency("MyFeatureName")`

权限可以有父权限和子权限。这不会影响权限验证，但可以在用户界面中对权限分组管理。

定义权限后，我们应该在模块的预初始化方法中注册它：

``` C#
    Configuration.Authorization.Providers.Add<MyAuthorizationProvider>();
```

权限定义类会被自动注册到[依赖注入](/Common.Structures/Dependency-Injection)。权限定义类可以注入到任意依赖中（如仓储）来使用其他资源创建权限定义。

### 权限验证

#### 使用AbpAuthorize属性

**AbpAuthorize**（MVC控制器中的**AbpMvcAuthorize**与Web API控制器中的**AbpApiAuthorize**）属性是最简单也是最常用的验证方式。比如下面的[应用服务](/Application.Layer/Application-Services)方法:

``` C#
    [AbpAuthorize("Administration.UserManagement.CreateUser")]
    public void CreateUser(CreateUserInput input)
    {
        // 如果用户未被授予“Administration.UserManagement.CreateUser”权限，那么他不能执行此方法。
    }
```

CreateUser方法不能被未被授予“Administration.UserManagement.CreateUser”权限的用户调用。

AbpAuthorize属性还会验证当前用户是否登录（使用[IAbpSession.UserId](/Common.Structures/Abp-Session)）。如果我们为方法声明AbpAuthorize，它将只验证登录：

``` C#
    [AbpAuthorize]
    public void SomeMethod(SomeMethodInput input)
    {
        //未登录用户不能执行此方法。
    }
```

##### AbpAuthorize属性注意事项

ABP使用了强大的动态方法拦截权限。因此使用AbpAuthorize属性有一些限制。

* 不能用于私有方法。
* 不能用于静态方法。
* 不能用于非注入类方法（必须使用[依赖注入](/Common.Structures/Dependency-Injection)）。

另外：

* 可以用于通过接口调用的任何公开（public）方法，比如通过接口使用应用服务。
* 如果方法是从类的引用直接调用的，则它应该是**虚拟方法（virtual）**，比如ASP.NET MVC或Web API控制器。
* 保护方法（protected）应该是**虚拟方法（virtual）**。

**注意**：授权属性共有四种类型：
* 在应用服务（应用层）中使用**Abp.Authorization.AbpAuthorize**属性。
* 在MVC控制器中使用**Abp.Web.Mvc.Authorization.AbpMvcAuthorize**属性。
* 在ASP.NET Web API中使用**Abp.WebApi.Authorization.AbpApiAuthorize**属性。
* 在ASP.NET Core中使用**Abp.AspNetCore.Mvc.Authorization.AbpMvcAuthorize**属性。

这些差异来源于继承。应用层中完全是ABP的实现，没有扩展任何类。对于MVC和Web API，它继承自框架的Authorize属性。

##### 禁止授权

你可以在应用服务中添加**AbpAllowAnonymous**属性为方法或类停用授权。在MVC、Web API和ASP.NET Core控制器中则使用框架原生的**AllowAnonymous**属性。

#### 使用IPermissionChecker

虽然AbpAuthorize属性适用于大多数情况，但是可能会有需要在方法内部检查权限的情况。我们可以注入并使用**IPermissionChecker**，如下所示：

``` C#
    public void CreateUser(CreateOrUpdateUserInput input)
    {
        if (!PermissionChecker.IsGranted("Administration.UserManagement.CreateUser"))
        {
            throw new AbpAuthorizationException("你没有创建用户的权限!");
        }

        //未被授予“Administration.UserManagement.CreateUser”权限的用户无法进入这一步。
    }
```

**IsGranted**（也有异步Async版）仅返回布尔值，因此你可以编写任何逻辑代码。如果你仅仅只是检查权限并像上面那样抛出异常，你可以使用**Authorize**方法。：

``` C#
    public void CreateUser(CreateOrUpdateUserInput input)
    {
        PermissionChecker.Authorize("Administration.UserManagement.CreateUser");

        //未被授予“Administration.UserManagement.CreateUser”权限的用户无法进入这一步。
    }
```

由于授权应用广泛，**ApplicationService**和一些公共基类已经注入并定义PermissionChecker属性。 因此，权限验证可以在不注入应用程序服务的情况下使用。

#### 在Razor视图中使用

视图基类定义了IsGranted方法检验当前用户权限。因此我们可以有条件地渲染视图。例如：

``` C#
    @if (IsGranted("Administration.UserManagement.CreateUser"))
    {
        <button id="CreateNewUserButton" class="btn btn-primary"><i class="fa fa-plus"></i> @L("CreateNewUser")</button>
    }
```

#### 客户端(JavaScript)

在客户端，我们可以使用定义在abp.auth命名空间下的API。多数情况下，我们需要验证当前用户的特定权限（使用权限名）。例如：

``` javascript
    abp.auth.isGranted('Administration.UserManagement.CreateUser');
```

你也可以使用**abp.auth.grantedPermissions**获取所有已授权限或**abp.auth.allPermissions**获取应用中所有可用权限名。运行时在**abp.auth**命名空间下查看其它方法。

### 权限管理

如果需要定义权限，可以[注入](/Common.Structures/Dependency-Injection)并使用**IPermissionManager**。