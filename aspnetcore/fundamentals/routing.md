---
title: ASP.NET Core 中的路由
author: ardalis
description: 了解 ASP.NET Core 路由功能如何负责将传入请求映射到路由处理程序。
ms.author: riande
ms.custom: mvc
ms.date: 10/01/2018
uid: fundamentals/routing
ms.openlocfilehash: d9ba96c7b2abd35b1b13c84814bf3f776e8d8731
ms.sourcegitcommit: 13940eb53c68664b11a2d685ee17c78faab1945d
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 10/01/2018
ms.locfileid: "47861052"
---
# <a name="routing-in-aspnet-core"></a>ASP.NET Core 中的路由

撰写者：[Ryan Nowak](https://github.com/rynowak)、[Steve Smith](https://ardalis.com/) 和 [Rick Anderson](https://twitter.com/RickAndMSFT)

路由功能负责将传入请求映射到路由处理程序。 路由在应用中定义，并在应用启动时进行配置。 路由可以选择从请求包含的 URL 中提取值，然后这些值便可用于处理请求。 通过使用应用中的路由信息，路由功能还能生成要映射到路由处理程序的 URL。 因此，路由可以根据 URL 查找路由处理程序，或者根据路由处理程序信息查找给定路由处理程序对应的 URL。

> [!IMPORTANT]
> 本文档介绍较低级别的 ASP.NET Core 路由。 有关 ASP.NET Core MVC 路由的信息，请参阅 <xref:mvc/controllers/routing>。

[查看或下载示例代码](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/routing/samples)（[如何下载](xref:tutorials/index#how-to-download-a-sample)）

## <a name="routing-basics"></a>路由基础知识

路由使用路由（<xref:Microsoft.AspNetCore.Routing.IRouter> 的实现）：

* 将传入请求映射到路由处理程序。
* 生成响应中使用的 URL。

通常情况下，应用具有一个路由集合。 请求到达时，将按顺序处理路由集合。 传入请求通过对路由集合中的每个可用路由调用 <xref:Microsoft.AspNetCore.Routing.IRouter.RouteAsync*> 方法来查找与请求 URL 匹配的路由。 与此相反，响应可根据路由信息使用路由生成 URL（例如，用于重定向或链接），并因此避免需要硬编码 URL，这有助于可维护性。

路由通过 <xref:Microsoft.AspNetCore.Builder.RouterMiddleware> 类连接到[中间件](xref:fundamentals/middleware/index)管道。 [ASP.NET Core MVC](xref:mvc/overview) 向中间件管道添加路由，作为其配置的一部分。 若要了解如何使用路由作为独立组件，请参阅[使用路由中间件](#use-routing-middleware)部分。

### <a name="url-matching"></a>URL 匹配

URL 匹配是一个过程，通过该过程，路由可向处理程序调度传入请求。 此过程基于 URL 路径中的数据，但可以进行扩展以考虑请求中的任何数据。 向单独的处理程序调度请求的功能是缩放应用的大小和复杂性的关键。

传入请求将进入 `RouterMiddleware`，后者将对序列中的每个路由调用 <xref:Microsoft.AspNetCore.Routing.IRouter.RouteAsync*> 方法。 <xref:Microsoft.AspNetCore.Routing.IRouter> 实例将选择是否通过将 [RouteContext.Handler](xref:Microsoft.AspNetCore.Routing.RouteContext.Handler*) 设置为非 NULL <xref:Microsoft.AspNetCore.Http.RequestDelegate> 来处理请求。 如果路由为请求设置处理程序，将停止路由处理，并调用处理程序来处理该请求。 如果尝试了所有路由，且请求未找到任何处理程序，中间件将调用 next，请求管道中的下一个中间件将被调用。

`RouteAsync` 的主要输入是与当前请求关联的 [RouteContext.HttpContext](xref:Microsoft.AspNetCore.Routing.RouteContext.HttpContext*)。 `RouteContext.Handler` 和 [RouteContext.RouteData](xref:Microsoft.AspNetCore.Routing.RouteContext.RouteData*) 是路由匹配后设置的输出。

`RouteAsync` 期间的匹配还可根据迄今执行的请求处理将 `RouteContext.RouteData` 的属性设为适当的值。 如果路由与请求匹配，`RouteContext.RouteData` 将包含有关结果的重要状态信息。

[RouteData.Values](xref:Microsoft.AspNetCore.Routing.RouteData.Values*) 是从路由中生成的路由值的字典。 这些值通常通过标记 URL 来确定，可用来接受用户输入，或者在应用内作出进一步的调度决策。

[RouteData.DataTokens](xref:Microsoft.AspNetCore.Routing.RouteData.DataTokens*) 是一个与匹配的路由相关的其他数据的属性包。 提供 `DataTokens` 以支持将状态数据与每个路由相关联，以便应用稍后可根据所匹配的路由作出决策。 这些值是开发者定义的，不会影响通过任何方式路由的行为。 此外，存储于数据令牌中的值可以属于任何类型，与路由值相反，后者必须能够轻松转换为字符串，或从字符串进行转换。

[RouteData.Routers](xref:Microsoft.AspNetCore.Routing.RouteData.Routers*) 是参与成功匹配请求的路由的列表。 路由可以相互嵌套。 `Routers` 属性可以通过导致匹配的逻辑路由树反映该路径。 通常情况下，`Routers` 中的第一项是路由集合，应该用于生成 URL。 `Routers` 中的最后一项是匹配的路由处理程序。

### <a name="url-generation"></a>URL 生成

URL 生成是通过其可根据一组路由值创建 URL 路径的过程。 这需要考虑处理程序与访问它们的 URL 之间的逻辑分隔。

URL 遵循类似的迭代过程，但开头是将用户或框架代码调用到路由集合的 <xref:Microsoft.AspNetCore.Routing.IRouter.GetVirtualPath*> 方法中。 然后，每个路由会有序地调用其 `GetVirtualPath` 方法，直到返回非 NULL 的 <xref:Microsoft.AspNetCore.Routing.VirtualPathData>。

`GetVirtualPath` 的主输入有：

* [VirtualPathContext.HttpContext](xref:Microsoft.AspNetCore.Routing.VirtualPathContext.HttpContext*)
* [VirtualPathContext.Values](xref:Microsoft.AspNetCore.Routing.VirtualPathContext.Values*)
* [VirtualPathContext.AmbientValues](xref:Microsoft.AspNetCore.Routing.VirtualPathContext.AmbientValues*)

路由主要使用 `Values` 和 `AmbientValues` 提供的路由值来决定可能生成 URL 的位置以及要包括的值。 `AmbientValues` 是路由值的集合，这些值是通过将当前请求与路由系统相匹配而产生的。 与此相反，`Values` 是指定如何为当前操作生成所需 URL 的路由值。 当路由需要获取与当前上下文关联的服务或其他数据时，需提供 `HttpContext`。

> [!TIP]
> 将 [VirtualPathContext.Values](xref:Microsoft.AspNetCore.Routing.VirtualPathContext.Values*) 视为 [VirtualPathContext.AmbientValues](xref:Microsoft.AspNetCore.Routing.VirtualPathContext.AmbientValues*) 的一组替代。 URL 生成尝试重复使用当前请求中的路由值，以便轻松使用相同路由或路由值生成链接的 URL。

`GetVirtualPath` 的输出是 `VirtualPathData`。 `VirtualPathData` 是 `RouteData` 的并行值。 `VirtualPathData` 包含输出 URL 的 `VirtualPath` 以及路由应该设置的某些其他属性。

[VirtualPathData.VirtualPath](xref:Microsoft.AspNetCore.Routing.VirtualPathData.VirtualPath*) 属性包含路由生成的虚拟路径。 你可能需要进一步处理路径，具体取决于你的需求。 如果要在 HTML 中呈现生成的 URL，请预置应用的基路径。

[VirtualPathData.Router](xref:Microsoft.AspNetCore.Routing.VirtualPathData.Router*) 是对已成功生成 URL 的路由的引用。

[VirtualPathData.DataTokens](xref:Microsoft.AspNetCore.Routing.VirtualPathData.DataTokens*) 属性是生成 URL 的路由的其他相关数据的字典。 这是 [RouteData.DataTokens](xref:Microsoft.AspNetCore.Routing.RouteData.DataTokens*) 的并性值。

### <a name="creating-routes"></a>创建路由

路由提供 <xref:Microsoft.AspNetCore.Routing.Route> 类，作为 <xref:Microsoft.AspNetCore.Routing.IRouter> 的标准实现。 `Route` 使用 route template 语法来定义调用 <xref:Microsoft.AspNetCore.Routing.IRouter.RouteAsync*> 时针对 URL 路径进行匹配的模式。 调用 `GetVirtualPath` 时，`Route` 使用同一路由模板生成 URL。

大多数应用通过调用 <xref:Microsoft.AspNetCore.Builder.MapRouteRouteBuilderExtensions.MapRoute*> 或 <xref:Microsoft.AspNetCore.Routing.IRouteBuilder> 上定义的一种类似的扩展方法来创建路由。 所有方法都会创建 <xref:Microsoft.AspNetCore.Routing.Route> 的实例，并将其添加到路由集合中。

`MapRoute` 并不使用路由处理程序参数。 `MapRoute` 仅添加由 <xref:Microsoft.AspNetCore.Routing.RouteBuilder.DefaultHandler*> 处理的路由。 由于默认处理程序是 `IRouter`，它可能决定不处理该请求。 例如，ASP.NET Core MVC 通常被配置为默认处理程序，仅处理与可用控制器和操作匹配的请求。 若要详细了解到 MVC 的路由，请参阅 <xref:mvc/controllers/routing>。

以下代码示例是典型 ASP.NET Core MVC 路由定义使用的一个 `MapRoute` 调用示例：

```csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home}/{action=Index}/{id?}");
```

此模板与 `/Products/Details/17` 等 URL 路径相匹配，并且提取路由值 `{ controller = Products, action = Details, id = 17 }`。 路由值是通过将 URL 路径拆分成段，并且将每段与路由模板中的路由参数名称相匹配来确定的。 路由参数已命名。 它们是通过将参数名称括在大括号 `{ ... }` 中进行定义的。

上述模板还可匹配 URL 路径 `/`，并且将生成值 `{ controller = Home, action = Index }`。 这是因为 `{controller}` 和 `{action}` 路由参数具有默认值，且 `id` 路由参数是可选的。 路由参数名称为参数定义默认值后，等号 `=` 后将有一个值。 路由参数名称后的问号 `?` 将参数定义为可选。 路由匹配时，具有默认值的路由参数始终会生成路由值。 如果没有相应的 URL 路径段，则可选参数不会生成路由值。

有关路由模板功能和语法的详细说明，请参阅 [route-template-reference](#route-template-reference)。

此示例包括路由约束：

```csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home}/{action=Index}/{id:int}");
```

此模板与类似于 `/Products/Details/17` 而不是 `/Products/Details/Apples` 的 URL 路径相匹配。 路由参数定义 `{id:int}` 为 `id` 路由参数定义[路由约束](#route-constraint-reference)。 路由约束实现 <xref:Microsoft.AspNetCore.Routing.IRouteConstraint> 并检查路由值，以验证它们。 在此示例中，路由值 `id` 必须可转换为整数。 有关框架提供的路由约束的更详细说明，请参阅 [route-constraint-reference](#route-constraint-reference)。

其他 `MapRoute` 重载接受 `constraints`、`dataTokens` 和 `defaults` 的值。 `MapRoute` 的此类附加参数被定义为 `object` 类型。 这些参数的典型用法是传递匿名类型化对象，其中匿名类型的属性名匹配路由参数名称。

以下两个示例可创建等效路由：

```csharp
routes.MapRoute(
    name: "default_route",
    template: "{controller}/{action}/{id?}",
    defaults: new { controller = "Home", action = "Index" });

routes.MapRoute(
    name: "default_route",
    template: "{controller=Home}/{action=Index}/{id?}");
```

> [!TIP]
> 定义约束和默认值的内联语法对于简单路由很方便。 但是，存在内联语法不支持的功能，例如数据令牌。

下面的代码示例演示了更多方案：

```csharp
routes.MapRoute(
    name: "blog",
    template: "Blog/{*article}",
    defaults: new { controller = "Blog", action = "ReadArticle" });
```

此模板与 `/Blog/All-About-Routing/Introduction` 等 URL 路径相匹配，并且提取值 `{ controller = Blog, action = ReadArticle, article = All-About-Routing/Introduction }`。 `controller` 和 `action` 的默认路由值由路由生成，即便模板中没有对应的路由参数。 可在路由模板中指定默认值。 根据路由参数名称前的星号  *外观，`article` 路由参数被定义为全方位*`*`。 全方位路由参数可捕获 URL 路径的其余部分，也能匹配空白字符串。

此示例可添加路由约束和数据令牌：

```csharp
routes.MapRoute(
    name: "us_english_products",
    template: "en-US/Products/{id}",
    defaults: new { controller = "Products", action = "Details" },
    constraints: new { id = new IntRouteConstraint() },
    dataTokens: new { locale = "en-US" });
```

此模板与 `/en-US/Products/5` 等 URL 路径相匹配，并且提取值 `{ controller = Products, action = Details, id = 5 }` 和数据令牌 `{ locale = en-US }`。

![本地 Windows 令牌](routing/_static/tokens.png)

### <a name="url-generation"></a>URL 生成

`Route` 类还可通过将一组路由值与其路由模板组合来生成 URL。 从逻辑上来说，这是匹配 URL 路径的反向过程。

> [!TIP]
> 为更好了解 URL 生成，请想象你要生成的 URL，然后考虑路由模板将如何匹配该 URL。 将生成什么值？ 这大致相当于 URL 在 `Route` 类中的生成方式。

此示例使用基本的 ASP.NET Core MVC 样式路由：

```csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home}/{action=Index}/{id?}");
```

使用路由值 `{ controller = Products, action = List }`，此路由将生成 URL `/Products/List`。 路由值将替换为相应的路由参数，以形成 URL 路径。 由于 `id` 属于可选路由参数，因此它可以没有值。

使用路由值 `{ controller = Home, action = Index }`，此路由将生成 URL `/`。 提供的路由值与默认值相匹配，因此可以安全忽略这些值对应的段。 已生成的两个 URL 将往返这个路由定义，并生成用于生成该 URL 的相同路由值。

> [!TIP]
> 使用 ASP.NET Core MVC 应用应该使用 <xref:Microsoft.AspNetCore.Mvc.Routing.UrlHelper> 生成 URL，而不是直接调用到路由。

有关 URL 生成的详细信息，请参阅 [url-generation-reference](#url-generation-reference)。

## <a name="use-routing-middleware"></a>使用路由中间件

::: moniker range=">= aspnetcore-2.1"

引用应用项目文件中的 [Microsoft.AspNetCore.App 元包](xref:fundamentals/metapackage-app)。

::: moniker-end

::: moniker range="= aspnetcore-2.0"

引用应用项目文件中的 [Microsoft.AspNetCore.All 元包](xref:fundamentals/metapackage)。

::: moniker-end

::: moniker range="< aspnetcore-2.0"

引用应用项目文件中的 [Microsoft.AspNetCore.Routing](https://www.nuget.org/packages/Microsoft.AspNetCore.Routing/)。

::: moniker-end

将路由添加到 `Startup.ConfigureServices` 中的服务容器：

::: moniker range=">= aspnetcore-2.0"

[!code-csharp[](routing/samples/2.x/RoutingSample/Startup.cs?name=snippet_ConfigureServices&highlight=3)]

::: moniker-end

::: moniker range="< aspnetcore-2.0"

[!code-csharp[](routing/samples/1.x/RoutingSample/Startup.cs?name=snippet_ConfigureServices&highlight=3)]

::: moniker-end

必须在 `Startup.Configure` 方法中配置路由。 示例应用使用这些 API：

* `RouteBuilder`
* `Build`
* `MapGet` &ndash; 仅匹配 HTTP GET 请求。
* `UseRouter`

::: moniker range=">= aspnetcore-2.0"

[!code-csharp[](routing/samples/2.x/RoutingSample/Startup.cs?name=snippet_RouteHandler)]

::: moniker-end

::: moniker range="< aspnetcore-2.0"

[!code-csharp[](routing/samples/1.x/RoutingSample/Startup.cs?name=snippet_RouteHandler)]

::: moniker-end

下表显示了具有给定 URI 的响应。

| URI                  | 响应                                          |
| -------------------- | ------------------------------------------------- |
| /package/create/3    | Hello! Route values: [operation, create], [id, 3] |
| /package/track/-3    | Hello! Route values: [operation, track], [id, -3] |
| /package/track/-3/   | Hello! Route values: [operation, track], [id, -3] |
| /package/track/      | &lt;失败，无任何匹配&gt;                    |
| GET /hello/Joe       | Hi, Joe!                                          |
| POST /hello/Joe      | &lt;失败，仅匹配 HTTP GET&gt;       |
| GET /hello/Joe/Smith | &lt;失败，无任何匹配&gt;                    |

如果要配置单个路由，请在 `IRouter` 实例中调用 <xref:Microsoft.AspNetCore.Builder.RoutingBuilderExtensions.UseRouter*> 传入。 无需使用 <xref:Microsoft.AspNetCore.Routing.RouteBuilder>。

框架可提供一组扩展方法，用于创建如下路由：

* `MapRoute`
* `MapGet`
* `MapPost`
* `MapPut`
* `MapDelete`
* `MapVerb`

某些方法（如 `MapGet`）需要提供 `RequestDelegate`。 路由匹配时，`RequestDelegate` 会用作路由处理程序。 此系列中的其他方法允许配置中间件管道，将其用作路由处理程序。 如果 Map 方法不接受处理程序（如 `MapRoute`），则它将使用 <xref:Microsoft.AspNetCore.Routing.RouteBuilder.DefaultHandler*>。

`Map[Verb]` 方法将使用约束来将路由限制为方法名称中的 HTTP 谓词。 有关示例，请参阅 <xref:Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapGet*> 和 <xref:Microsoft.AspNetCore.Routing.RequestDelegateRouteBuilderExtensions.MapVerb*>。

## <a name="route-template-reference"></a>路由模板参考

如果路由找到匹配项，大括号 (`{ ... }`) 内的令牌定义绑定的路由参数。 可在路由段中定义多个路由参数，但必须由文本值隔开。 例如，`{controller=Home}{action=Index}` 不是有效的路由，因为 `{controller}` 和 `{action}` 之间没有文本值。 这些路由参数必须具有名称，且可能指定了其他属性。

路由参数以外的文本（例如 `{id}`）和路径分隔符 `/` 必须匹配 URL 中的文本。 文本匹配区分大小写，并且基于 URL 路径已解码的表示形式。 若要匹配文本路由参数分隔符 `{` 或 `}`，可通过重复该字符（`{{` 或 `}}`）对其进行转义。

尝试捕获具有可选文件扩展名的文件名的 URL 模式还有其他注意事项。 例如，考虑模板 `files/{filename}.{ext?}`。 当 `filename` 和 `ext` 都存在时，将填充这两个值。 如果 URL 中仅存在 `filename`，则路由匹配，因为尾随句点 `.` 是可选的。 以下 URL 与此路由相匹配：

* `/files/myFile.txt`
* `/files/myFile`

你可以使用 `*` 字符作为路由参数的前缀，以绑定到 URI 的其余部分。 这称为 catch-all 参数。 例如，`blog/{*slug}` 将匹配以 `/blog` 开头且其后带有任何值（将分配给 `slug` 路由值）的 URI。 全方位参数还可以匹配空字符串。

::: moniker range=">= aspnetcore-2.2"

使用路由生成 URL（包括路径分隔符 (`/`)）时，catch-all 参数会转义相应的字符。 例如，路由值为 `{ path = "my/path" }` 的路由 `foo/{*path}` 生成 `foo/my%2Fpath`。 请注意转义的正斜杠。 要往返路径分隔符，请使用 `**` 路由参数前缀。 `{ path = "my/path" }` 的路由 `foo/{**path}` 生成 `foo/my/path`。

::: moniker-end

路由参数可能具有指定的默认值，方法是在参数名称后使用等号 (`=`) 隔开以指定默认值。 例如，`{controller=Home}` 将 `Home` 定义为 `controller` 的默认值。 如果参数的 URL 中不存在任何值，则使用默认值。 除默认值外，路由参数可能是可选的，如 `id?` 中所示，通过将问号 (`?`) 追加到参数名称末尾来指定。 可选值和默认路径参数的区别在于具有默认值的路由参数始终会生成一个值，而可选参数仅当请求 URL 提供值时才会具有一个值。

::: moniker range=">= aspnetcore-2.2"

路由参数可能具有约束，必须匹配从 URL 中绑定的路由值。 在路由参数后面添加一个冒号 (`:`) 和约束名称可指定路由参数上的内联约束。 如果约束需要参数，将以在约束名称后括在括号 `( )` 中的形式提供。 通过追加另一个冒号 (`:`) 和约束名称，可指定多个内联约束。 约束名称和参数将传递给 <xref:Microsoft.AspNetCore.Routing.IInlineConstraintResolver> 服务，以创建 <xref:Microsoft.AspNetCore.Routing.IRouteConstraint> 的实例，用于处理 URL。 如果约束构造函数要求服务，通过依赖关系注入的应用服务解决它们。 例如，路由模板 `blog/{article:minlength(10)}` 使用参数 `10` 指定 `minlength` 约束。 有关路由约束详情以及框架提供的约束列表，请参阅[路由约束引用](#route-constraint-reference)部分。

::: moniker-end

::: moniker range="< aspnetcore-2.2"

路由参数可能具有约束，必须匹配从 URL 中绑定的路由值。 在路由参数后面添加一个冒号 (`:`) 和约束名称可指定路由参数上的内联约束。 如果约束需要参数，将以在约束名称后括在括号 `( )` 中的形式提供。 通过追加另一个冒号 (`:`) 和约束名称，可指定多个内联约束。 约束名称和参数将传递给 <xref:Microsoft.AspNetCore.Routing.IInlineConstraintResolver> 服务，以创建 <xref:Microsoft.AspNetCore.Routing.IRouteConstraint> 的实例，用于处理 URL。 例如，路由模板 `blog/{article:minlength(10)}` 使用参数 `10` 指定 `minlength` 约束。 有关路由约束详情以及框架提供的约束列表，请参阅[路由约束引用](#route-constraint-reference)部分。

::: moniker-end

::: moniker range=">= aspnetcore-2.2"

路由参数还可以具有参数转换器，用于在生成链接以及将操作和页面匹配到 URI 时转换参数的值。 与约束类似，可以通过在路由参数名称后面添加冒号 (`:`) 和转换器名称，将参数变换器内联添加到路径参数。 例如，路由模板 `blog/{article:slugify}` 指定 `slugify` 转换器。

::: moniker-end

下表演示某些路由模板及其行为。

| 路由模板                         | 匹配 URL 示例  | 说明                                                                  |
| -------------------------------------- | --------------------- | ---------------------------------------------------------------------- |
| hello                                  | /hello                | 仅匹配单个路径 `/hello`                                  |
| {Page=Home}                            | /                     | 匹配并将 `Page` 设置为 `Home`                                      |
| {Page=Home}                            | /Contact              | 匹配并将 `Page` 设置为 `Contact`                                   |
| {controller}/{action}/{id?}            | /Products/List        | 映射到 `Products` 控制器和 `List` 操作                       |
| {controller}/{action}/{id?}            | /Products/Details/123 |  映射到 `Products` 控制器和 `Details` 操作。  将 `id` 设置为 123 |
| {controller=Home}/{action=Index}/{id?} | /                     |  映射到 `Home` 控制器和 `Index` 方法；将忽略 `id`。       |

使用模板通常是进行路由最简单的方法。 还可在路由模板外指定约束和默认值。

> [!TIP]
> 启用[日志记录](xref:fundamentals/logging/index)查看内置路由实现（如 `Route`）如何匹配请求。

## <a name="reserved-routing-names"></a>保留的路由名称

以下关键字是保留的名称，它们不能用作路由名称或参数：

* `action`
* `area`
* `controller`
* `handler`
* `page`

## <a name="route-constraint-reference"></a>路由约束参考

如果 `Route` 匹配传入 URL 的语法并将 URL 路径标记化为路由值，将执行路由约束。 路径约束通常检查通过路径模板关联的路径值，并对该值是否可接受做出是/否决定。 某些路由约束使用路由值以外的数据来考虑是否可以路由请求。 例如，<xref:Microsoft.AspNetCore.Routing.Constraints.HttpMethodRouteConstraint> 可以根据其 HTTP 谓词接受或拒绝请求。

> [!WARNING]
> 避免使用输入验证约束，因为这样意味着无效输入将导致“404 - 未找到”响应，而不是含相应错误消息的“400 - 错误请求”。 路由约束用于消除类似路由间的歧义，而不是验证特定路由的输入。

下表演示某些路由约束及其预期行为。

| 约束 | 示例 | 匹配项示例 | 说明 |
| ---------- | ------- | --------------- | ----- |
| `int` | `{id:int}` | `123456789`, `-123456789`  | 匹配任何整数 |
| `bool` | `{active:bool}` | `true`, `FALSE` | 匹配 `true`或 `false`（区分大小写） |
| `datetime` | `{dob:datetime}` | `2016-12-31`, `2016-12-31 7:32pm`  | 匹配有效的 `DateTime` 值（位于固定区域性中 - 查看警告） |
| `decimal` | `{price:decimal}` | `49.99`, `-1,000.01` | 匹配有效的 `decimal` 值（位于固定区域性中 - 查看警告） |
| `double` | `{weight:double}` | `1.234`, `-1,001.01e8` | 匹配有效的 `double` 值（位于固定区域性中 - 查看警告） |
| `float` | `{weight:float}` | `1.234`, `-1,001.01e8` | 匹配有效的 `float` 值（位于固定区域性中 - 查看警告） |
| `guid` | `{id:guid}` | `CD2C1638-1638-72D5-1638-DEADBEEF1638`, `{CD2C1638-1638-72D5-1638-DEADBEEF1638}` | 匹配有效的 `Guid` 值 |
| `long` | `{ticks:long}` | `123456789`, `-123456789` | 匹配有效的 `long` 值 |
| `minlength(value)` | `{username:minlength(4)}` | `Rick` | 字符串必须至少为 4 个字符 |
| `maxlength(value)` | `{filename:maxlength(8)}` | `Richard` | 字符串不得超过 8 个字符 |
| `length(length)` | `{filename:length(12)}` | `somefile.txt` | 字符串必须正好为 12 个字符 |
| `length(min,max)` | `{filename:length(8,16)}` | `somefile.txt` | 字符串必须至少为 8 个字符，且不得超过 16 个字符 |
| `min(value)` | `{age:min(18)}` | `19` | 整数值必须至少为 18 |
| `max(value)` | `{age:max(120)}` | `91` | 整数值不得超过 120 |
| `range(min,max)` | `{age:range(18,120)}` | `91` | 整数值必须至少为 18，且不得超过 120 |
| `alpha` | `{name:alpha}` | `Rick` | 字符串必须由一个或多个字母字符（`a`-`z`，区分大小写）组成 |
| `regex(expression)` | `{ssn:regex(^\\d{{3}}-\\d{{2}}-\\d{{4}}$)}` | `123-45-6789` | 字符串必须匹配正则表达式（参见有关定义正则表达式的提示） |
| `required` | `{name:required}` | `Rick` |  用于强制在 URL 生成过程中存在非参数值 |

可向单个参数应用多个由冒号分隔的约束。 例如，以下约束将参数限制为大于或等于 1 的整数值：

```csharp
[Route("users/{id:int:min(1)}")]
public User GetUserById(int id) { }
```

> [!WARNING]
> 验证 URL 的路由约束并将转换为始终使用固定区域性的 CLR 类型（例如 `int` 或 `DateTime`）。 这些约束假定 URL 不可本地化。 框架提供的路由约束不会修改存储于路由值中的值。 从 URL 中分析的所有路由值都将存储为字符串。 例如，`float` 约束会尝试将路由值转换为浮点数，但转换后的值仅用来验证其是否可转换为浮点数。

## <a name="regular-expressions"></a>正则表达式

ASP.NET Core 框架将向正则表达式构造函数添加 `RegexOptions.IgnoreCase | RegexOptions.Compiled | RegexOptions.CultureInvariant`。 有关这些成员的说明，请参阅 <xref:System.Text.RegularExpressions.RegexOptions>。

正则表达式与路由和 C# 计算机语言使用的分隔符和令牌相似。 必须对正则表达式令牌进行转义。 若要在路由中使用正则表达式 `^\d{3}-\d{2}-\d{4}$`，则表达式必须要有如 C# 源文件中键入的 `\\` 一样的 `\` 字符，以转义 `\` 字符串转义字符（除非使用 [verbatim 字符串文本](/dotnet/csharp/language-reference/keywords/string)）。 `{`、`}`、`[` 和 `]` 字符必须成双进行转义，以转义路由参数分隔符字符。 下表显示正则表达式和转义的版本。

| 表达式            | 已转义                        |
| --------------------- | ------------------------------ |
| `^\d{3}-\d{2}-\d{4}$` | `^\\d{{3}}-\\d{{2}}-\\d{{4}}$` |
| `^[a-z]{2}$`          | `^[[a-z]]{{2}}$`               |

路由中使用的正则表达式通常以 `^` 字符（匹配字符串的起始位置）开头，以 `$` 字符（匹配字符串的结束位置）结尾。 `^` 和 `$` 字符可确保正则表达式匹配整个路由参数值。 如果没有 `^` 和 `$` 字符，正则表达式将匹配字符串内的所有子字符串，而这通常是不需要的。 下表显示部分示例，并说明它们为何匹配或未能匹配。

| 表达式   | String    | 匹配 | 注释               |
| ------------ | --------- |  ---- |  -------------------- |
| `[a-z]{2}`   | hello     | 是   | 子字符串匹配     |
| `[a-z]{2}`   | 123abc456 | 是   | 子字符串匹配     |
| `[a-z]{2}`   | mz        | 是   | 匹配表达式    |
| `[a-z]{2}`   | MZ        | 是   | 不区分大小写    |
| `^[a-z]{2}$` |  hello    | 否    | 参阅上述 `^` 和 `$` |
| `^[a-z]{2}$` | 123abc456 | 否    | 参阅上述 `^` 和 `$` |

有关正则表达式语法的详细信息，请参阅 [.NET Framework 正则表达式](/dotnet/standard/base-types/regular-expression-language-quick-reference)。

若要将参数限制为一组已知的可能值，可使用正则表达式。 例如，`{action:regex(^(list|get|create)$)}` 仅将 `action` 路由值匹配到 `list`、`get` 或 `create`。 如果传递到约束字典中，字符串 `^(list|get|create)$` 将等效。 已传递到约束字典（不与模板内联）且不匹配任何已知约束的约束还将被视为正则表达式。

::: moniker range=">= aspnetcore-2.2"

## <a name="parameter-transformer-reference"></a>参数转换器参考

参数转换器在为 `Route` 生成链接时执行。 参数转换器获取参数的路由值并将其转换为新的字符串值。 转换后的值用于生成的链接。 例如，路由模式 `blog\{article:slugify}`（具有 `Url.Action(new { article = "MyTestArticle" })`）中的自定义 `slugify` 参数转换器生成 `blog\my-test-article`。 参数转换器实现 `Microsoft.AspNetCore.Routing.IOutboundParameterTransformer` 并使用 <xref:Microsoft.AspNetCore.Routing.RouteOptions.ConstraintMap> 进行配置。

框架还使用参数转换器来转换端点解析的 URI。 例如，ASP.NET Core MVC 使用参数转换器来转换用于匹配 `area``controller``action` 和 `page` 的路由值。

```csharp
routes.MapRoute(
    name: "default",
    template: "{controller=Home:slugify}/{action=Index:slugify}/{id?}");
```

使用上述路由，操作 `SubscriptionManagementController.GetAll()` 与 URI `/subscription-management/get-all` 匹配。 参数转换器不会更改用于生成链接的路由值。 `Url.Action("GetAll", "SubscriptionManagement")` 输出 `/subscription-management/get-all`。

ASP.NET Core MVC 还附带 `Microsoft.AspNetCore.Mvc.ApplicationModels.RouteTokenTransformerConvention` API 约定。 该约定将指定的参数转换器应用于应用程序中的所有属性路由令牌。

::: moniker-end

## <a name="url-generation-reference"></a>URL 生成参考

以下示例演示如何在给定路由值字典和 <xref:Microsoft.AspNetCore.Routing.RouteCollection> 的情况下生成路由链接。

::: moniker range=">= aspnetcore-2.0"

[!code-csharp[](routing/samples/2.x/RoutingSample/Startup.cs?name=snippet_Dictionary)]

::: moniker-end

::: moniker range="< aspnetcore-2.0"

[!code-csharp[](routing/samples/1.x/RoutingSample/Startup.cs?name=snippet_Dictionary)]

::: moniker-end

上述示例末尾生成的 `VirtualPath` 为 `/package/create/123`。 字典提供“跟踪包路由”模板 `package/{operation}/{id}` 的 `operation` 和 `id` 路由值。 有关详细信息，请参阅[使用路由中间件](#use-routing-middleware)部分或[示例应用](https://github.com/aspnet/Docs/tree/master/aspnetcore/fundamentals/routing/samples)中的示例代码。

`VirtualPathContext` 构造函数的第二个参数是环境值的集合。 通过限制特定请求上下文中开发者必须指定的值数，环境值可提供便利。 当前请求的当前路由值被视为链接生成的环境值。 在 ASP.NET Core MVC 应用中，如果处于 `HomeController` 的 `About` 操作中，则无需指定控制器路由值即可链接到 `Index` 操作&mdash;将使用 `Home` 环境值。

URL 中从左至右，将忽略不匹配参数的环境值，如果显式提供的值替代环境值，该环境值也将被忽略。

显式提供但不匹配任何对象的值将添加到查询字符串。 下表显示使用路由模板 `{controller}/{action}/{id?}` 时的结果。

| 环境值                | 显式值                   | 结果                  |
| ----------------------------- | --------------------------------- | ----------------------- |
| controller="Home"             | action="About"                    | `/Home/About`           |
| controller="Home"             | controller="Order",action="About" | `/Order/About`          |
| controller="Home",color="Red" | action="About"                    | `/Home/About`           |
| controller="Home"             | action="About",color="Red"        | `/Home/About?color=Red` |

如果路由具有不对应于参数的默认值，且该值以显式方式提供，则它必须与默认值相匹配：

```csharp
routes.MapRoute("blog_route", "blog/{*slug}",
    defaults: new { controller = "Blog", action = "ReadPost" });
```

当提供控制器和操作的匹配值时，链接生成只生成此路由的链接。
