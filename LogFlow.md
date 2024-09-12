# LogFlow

## 后端

1. 将前端项目打包资源内嵌进项目。
   - 使用 `EmbeddedFileProvider` 将打包好的前端资源嵌入到程序集内，以便后端能够直接访问这些资源，而无需依赖外部文件系统。
   - 嵌入的文件路径由常量 `EmbeddedFileNamespace = "LogFlow.Dashboard.Pages.dist"` 指定。

2. 判断请求路径和方法，根据是否请求静态文件执行相应操作，最终处理非静态文件时返回 `index.html`。
   - 通过 `Invoke(HttpContext)` 方法，根据请求的 `HTTP` 方法和路径，决定如何处理请求。
   - 如果请求路径为 `/LogFlow` 且带有文件扩展名，则调用 `StaticFileMiddleware` 处理嵌入式静态文件请求。
   - 如果请求路径没有文件扩展名（非静态文件请求），则返回嵌入式的 `index.html` 页面，并根据 `AspNetCoreDashboardOptions` 配置替换其中的动态占位符（如文档标题、头部内容等）。
   
3. 使用 `AspNetCoreDashboardOptions` 进行动态内容替换：
   - `DocumentTitle`：网页的标题。
   - `HeadContent`：注入到 `<head>` 中的额外内容。
   - `StylesPath`：前端样式文件路径。
   - `ScriptBundlePath`：前端脚本捆绑包的路径。

4. 利用 `StaticFileMiddleware` 提供静态文件：
   - 通过 `CreateStaticFileMiddleware()` 方法创建 `StaticFileMiddleware`，它使用嵌入式文件系统来服务静态资源。
   - 设置 `RequestPath = "/LogFlow"`，确保请求 `/LogFlow` 路径下的资源被正确处理。

## 前端

1. 路由需以“/LogFlow”为前缀。
2. 设置vite.config.ts文件的base属性为“/LogFlow”。