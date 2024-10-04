# WebSocket、SSE在 .NET 中的实现

## 1. WebSocket

### 概述
- **WebSocket** 是一种双向、全双工的通信协议，允许客户端和服务器之间建立持久的连接。
- 适用于需要频繁交换数据的应用，如在线游戏、聊天应用、实时协作工具等。

### 实现步骤
- **Startup.cs 配置**：
  ```csharp
  public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
  {
      app.UseWebSockets();  // 启用 WebSocket 支持
  
      app.Use(async (context, next) =>
      {
          if (context.WebSockets.IsWebSocketRequest)  // 检查请求是否为 WebSocket 请求
          {
              var webSocket = await context.WebSockets.AcceptWebSocketAsync();  // 接受 WebSocket 连接
              await HandleWebSocketAsync(webSocket);  // 处理 WebSocket 连接
          }
          else
          {
              await next();  // 处理其他请求
          }
      });
  }
  
  private async Task HandleWebSocketAsync(WebSocket webSocket)
  {
      var buffer = new byte[1024 * 4];  // 创建缓冲区
      WebSocketReceiveResult result = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
      
      while (!result.CloseStatus.HasValue)  // 检查连接是否关闭
      {
          // 处理接收到的消息
          var message = Encoding.UTF8.GetString(buffer, 0, result.Count);
          Console.WriteLine($"Received: {message}");  // 打印接收到的消息
          
          // 发送消息
          await webSocket.SendAsync(new ArraySegment<byte>(buffer, 0, result.Count), result.MessageType, result.EndOfMessage, CancellationToken.None);
          
          result = await webSocket.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);  // 继续接收消息
      }
  
      // 关闭 WebSocket 连接
      await webSocket.CloseAsync(result.CloseStatus.Value, result.CloseStatusDescription, CancellationToken.None);
  }
  ```

### 优缺点
- **优点**：
  - 实时性好，延迟低。
  - 双向通信，适合需要频繁交互的场景。

- **缺点**：
  - 复杂性高，需要处理连接管理。
  - 不适合较低频率的消息发送。

### 应用场景
- 实时聊天应用
- 在线游戏
- 实时数据监控和仪表盘

## 2. Server-Sent Events (SSE)

### 概述
- **SSE** 是一种单向的通信方式，服务器可以主动向客户端推送数据。
- 适合于需要频繁更新而客户端不需要发送数据的场景。

### 实现步骤
- **创建 SSE 端点**：
  ```csharp
  [HttpGet]
  [Route("sse")]
  public async Task Sse()
  {
      Response.ContentType = "text/event-stream";  // 设置内容类型
      Response.Headers.Add("Cache-Control", "no-cache");  // 禁用缓存
      
      while (true)
      {
          await Response.WriteAsync($"data: {DateTime.Now}\n\n");  // 发送当前时间
          await Response.Body.FlushAsync();  // 刷新响应流
          await Task.Delay(1000);  // 每秒发送一次数据
      }
  }
  ```

### 优缺点
- **优点**：
  - 实现简单，使用 HTTP 协议，兼容性好。
  - 只需在服务器上实现，客户端只需接收数据。

- **缺点**：
  - 单向通信，不适合需要交互的场景。
  - 不支持多路复用，同一连接只能推送一种类型的数据。

### 应用场景
- 实时股票行情更新
- 新闻推送
- 实时监控和日志显示
