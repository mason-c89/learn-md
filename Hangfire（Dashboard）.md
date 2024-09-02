# Hangfire -- Dashboard

- 仅记录Hangfire中Dashboard的实现方式。

Hangfire Dashboard 是 Hangfire 框架提供的一个用于监控和管理后台任务的 Web 界面。它允许开发者查看任务队列、任务状态、失败的任务等详细信息。Hangfire Dashboard 的实现原理主要包括以下几个方面：

### 1. **中间件的注册和配置**
   Hangfire Dashboard 作为一个 ASP.NET 中间件实现。当你在 ASP.NET 应用程序中配置 Hangfire Dashboard 时，它被注册为一个中间件，并嵌入到你的 ASP.NET 管道中。你可以通过调用 `app.UseHangfireDashboard()` 方法将其添加到请求处理管道中。

- `HangfireApplicationBuilderExtensions`：其内有扩展方法UseHangfireDashboard，用于将Dashboard注册到管道中。

  ```c#
  public static IApplicationBuilder UseHangfireDashboard(
              [NotNull] this IApplicationBuilder app,
              [NotNull] string pathMatch = "/hangfire",
              [CanBeNull] DashboardOptions options = null,
              [CanBeNull] JobStorage storage = null)
          {
              if (app == null) throw new ArgumentNullException(nameof(app));
              if (pathMatch == null) throw new ArgumentNullException(nameof(pathMatch));
  
              HangfireServiceCollectionExtensions.ThrowIfNotConfigured(app.ApplicationServices);
  
              var services = app.ApplicationServices;
  
              storage = storage ?? services.GetRequiredService<JobStorage>();
              options = options ?? services.GetService<DashboardOptions>() ?? new DashboardOptions();
              options.TimeZoneResolver = options.TimeZoneResolver ?? services.GetService<ITimeZoneResolver>();
  
              var routes = app.ApplicationServices.GetRequiredService<RouteCollection>();
  
              app.Map(new PathString(pathMatch), x => x.UseMiddleware<AspNetCoreDashboardMiddleware>(storage, options, routes));
  
              return app;
          }

### 2. **请求路由**
   Hangfire Dashboard 拦截特定路径（通常是 `/hangfire`）下的 HTTP 请求，并根据请求的路径解析出相应的页面或 API 请求。Dashboard 的所有资源（如 HTML、CSS、JavaScript 文件）和 API 端点都通过这个路由进行管理。

- **`DashboardRoutes`**: 这个类负责定义 Dashboard 的路由规则，并将不同的路径映射到对应的处理器（如页面或 API 端点）。

- **`AspNetCoreDashboardContext`**: 提供了在处理 HTTP 请求时需要的上下文信息。

### 3. **Authorization 机制**
   Hangfire Dashboard 提供了简单的授权机制。你可以通过设置 `DashboardOptions` 来定义授权规则。例如，你可以定义只有特定角色或用户组可以访问 Dashboard 界面。这是通过实现 `IDashboardAuthorizationFilter` 接口来完成的。

### 4. **UI 生成和视图引擎**
   Hangfire Dashboard 使用 Razor 模板引擎生成 HTML 页面。这些页面包含了有关任务的信息，包括任务的执行历史、失败的任务详情、队列统计等。Dashboard 页面还会包含一些 JavaScript 代码，用于实现动态功能，比如自动刷新任务状态。

- **`RazorPage`**: Hangfire Dashboard 使用 RazorPage 生成 HTML 页面。

- **`RazorPageDispatcher`**: 负责渲染 Razor 页面并将其输出到 HTTP 响应中。

- **`HtmlHelper`**: 提供辅助方法生成 Dashboard 的 HTML 元素。

### 5. **数据访问和展示**
   Hangfire Dashboard 直接从 Hangfire 的存储（通常是 SQL Server 或 Redis）中提取数据。这些数据包括任务的状态、执行时间、失败次数等。Dashboard 通过定期从数据库中查询这些信息，并将其展示在用户界面中。

- **`JobStorage`**: 这是 Hangfire 的核心类之一，负责访问底层存储（如 SQL Server 或 Redis）中的任务数据。

- **`MonitoringApi`**: 提供对任务状态和统计信息的访问接口。Dashboard 通过这个接口来获取和展示数据。

- **`SqlServerMonitoringApi` / `RedisMonitoringApi`**: 这些是具体的实现类，根据使用的存储类型不同而使用不同的类。

### 6. **实时更新**
   Dashboard 通过定期的 AJAX 请求来刷新任务状态。这些请求通常会触发后台数据的再次查询，并将最新的数据更新到 UI 上。因此，用户可以实时看到任务的执行进度和状态变化。

- **`DashboardDispatcher`**: 处理从客户端发起的 AJAX 请求，用于定期刷新任务状态。

- **`Polling`**: 一个 JavaScript 模块，用于在客户端定期发送请求并更新 UI。

### 7. **扩展性**
   Hangfire Dashboard 允许开发者通过编写自定义的 Dashboard 页面或组件来扩展其功能。你可以添加自定义的统计信息页面，或者通过编写插件来增加更多的功能。

- **`DashboardRoutes`**: 除了默认的路由外，你可以向这个类中添加自定义路由来扩展 Dashboard。
- **`RazorPageDispatcher`**: 可以扩展来渲染自定义的页面。
- **`DashboardContext`**: 可以用来传递自定义数据到你的扩展页面中。

### 8. **错误处理和日志记录**
   Dashboard 提供了查看和管理失败任务的功能。当任务执行失败时，Hangfire 会记录错误信息，并将其展示在 Dashboard 上。你可以直接从 Dashboard 中查看错误详情，并对失败的任务进行重试或删除操作。

- **`FailedJobsPage`**: 这是一个 Razor 页面，专门用于展示失败任务的信息。

- **`JobDetailsRenderer`**: 用于渲染任务的详细信息，包括错误日志。

- **`JobRetryDispatcher`**: 处理对失败任务的重试操作。