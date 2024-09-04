### 1. Razor 基础语法
- **HTML 默认输出**：Razor 视图中的 HTML 代码将直接呈现在最终输出中。
- **C# 代码嵌入**：通过 `@` 符号，可以将 C# 代码嵌入到 HTML 中。例如，`<p>@DateTime.Now</p>` 会输出当前时间。

### 2. Razor 表达式
- **隐式表达式**：使用 `@` 符号直接调用简单的 C# 表达式，如 `@DateTime.Now`。
- **显式表达式**：使用 `@()` 来包含更复杂的表达式，如 `@(DateTime.Now.AddDays(7))`。

### 3. Razor 代码块
- **代码块**：用 `@{}` 包围的代码块可以包含多行 C# 代码，这些代码不会直接输出，而是用于逻辑处理。例如：
  ```csharp
  @{
      var name = "World";
  }
  <p>Hello, @name!</p>
  ```

### 4. 控制结构
- **条件语句**：使用 `@if`, `@else if`, `@else` 实现条件分支。例如：
  ```csharp
  @if (isAdmin)
  {
      <p>Welcome, Admin!</p>
  }
  else
  {
      <p>Welcome, User!</p>
  }
  ```

- **循环语句**：支持 `@for`, `@foreach`, `@while` 等循环结构。例如：
  ```csharp
  @foreach (var item in items)
  {
      <p>@item.Name</p>
  }
  ```

### 5. 指令
- **@model**：用于指定视图的模型类型，例如：`@model MyViewModel`。
- **@functions** 和 **@code**：用于在视图中定义 C# 方法或属性。
- **@inherits**：允许视图继承自指定的基类。
- **@inject**：用于将服务注入到视图中，例如：`@inject ILogger<MyPage> Logger`。

### 6. 输出与编码
- **HTML 编码**：Razor 会自动对输出的字符串进行 HTML 编码，防止 XSS 攻击。
- **Raw 输出**：使用 `@Html.Raw()` 可以输出未编码的 HTML 字符串。

### 7. 注释
- **Razor 注释**：用 `@* *@` 包围的内容在视图中不会被渲染，也不会出现在最终的 HTML 输出中。

### 8. 条件属性渲染
- **属性渲染**：如果 Razor 表达式的值为 `null` 或 `false`，则不会渲染对应的 HTML 属性。例如：
  ```csharp
  <input type="checkbox" checked="@isChecked" />
  ```
  如果 `isChecked` 为 `false`，则 `checked` 属性不会出现在最终 HTML 中。

### 9. 本地函数与模板
- **本地函数**：可以在 Razor 代码块中定义并调用本地函数，用于渲染重复的 HTML 片段。例如：
  ```csharp
  @{
      void RenderPerson(string name)
      {
          <p>@name</p>
      }
  
      RenderPerson("John");
  }
  ```
