## 一分钟将代理OpenAI 的API
作为前端开发，一直听说Deno的大名，但遗憾的是至今还没有亲自尝试过编写一个简单的"Hello World"程序。现在，我打算利用Deno来实现一个简单的API代理服务，这旨在用于技术交流和学习，才不是因
为我发现了可以通过[Deno Deploy](https://deno.com/deploy/pricing)的免费套餐来代理访问OpenAI的接口。这是一个探索和学习的过程，希望我们可以一起讨论和优化🐶。

### 第一步
登录[https://dash.deno.com/projects](https://dash.deno.com/projects)，New Playground

### 第二步
复制下面代码
```JS
const targetOrigin = "https://api.openai.com";

Deno.serve(async (request) => {
  // 获取原始请求的 URL 和方法
  const { method, url, headers } = request;
  const originalUrl = new URL(url);
  
  // 记录日志
  console.info(`Incoming request: ${method} ${originalUrl.pathname}${originalUrl.search}`);

  // 构建目标 URL
  const targetUrl = new URL(originalUrl.pathname + originalUrl.search, targetOrigin);
  
  try {
    // 准备转发请求的选项
    const requestOptions: RequestInit = {
      method,
      headers: new Headers(headers), // 复制 headers
    };

    // 对于 POST、PUT、PATCH 方法，透传请求体
    if (request.method === "POST" || request.method === "PUT" || request.method === "PATCH") {
      requestOptions.body = await request.arrayBuffer();
    }

    console.info(`targetUrl: ${targetUrl}`);
    // console.debug('requestOptions:', requestOptions); // 里面会有API KEY，可另外处理。
    // 向目标 API 发起请求
    const response = await fetch(targetUrl.toString(), requestOptions);

    // 检查是否是 SSE
    const contentType = response.headers.get("Content-Type");
    console.info(`${targetUrl} response content type is:`, contentType);
    if (contentType === "text/event-stream") {
      // 返回一个新的 ReadableStream，用于将事件流传递给客户端
      const stream = new ReadableStream({
        start(controller) {
          const reader = response.body.getReader();
          return (function pump() {
            reader.read().then(({ done, value }) => {
              if (done) {
                controller.close();
                return;
              }
              controller.enqueue(value);
              pump();
            }).catch(error => {
              console.error('Error while reading the stream', error);
              controller.error(error);
              controller.close();
            });
          })();
        },
        cancel() {
          reader.cancel();
        }
      });

      // 返回 Response 对象，透传 SSE
      return new Response(stream, {
        headers: {
          "Content-Type": "text/event-stream",
          "Cache-Control": "no-cache",
          "Connection": "keep-alive",
        },
      });
    } else {
      // 其他类型的响应，直接透传
      const responseBody = await response.arrayBuffer();
      return new Response(responseBody, {
        status: response.status,
        headers: response.headers,
      });
    }
  } catch (error) {
    console.error(`Error proxying request to ${targetUrl}:`, error);
    return new Response(`Server error: ${error.message}`, { status: 500 });
  }
}, { port: 8000 });

console.log(`Proxy server running at http://localhost:8000/`);
```

### 第三步
[https://platform.openai.com/api-keys](https://platform.openai.com/api-keys) 从里面来个OpenAI 的API key，发请求时候带上。
没Key的可以 [https://bewildcard.com/i/HB0O8Q0Y](https://bewildcard.com/i/HB0O8Q0Y) 从里面注册，按照教程生成，有提供限时虚拟机的梯子，充值可以直接使用支付宝，省事儿；不过手续费不便宜。

#### 客户端
可以去下载[chatBox](https://chatboxai.app/zh)，将当前服务的域名设置在API域名既可。

### 第四步
Save & Deploy

### 验证
终端curl一哈
```shell
curl --location --request POST 'https://{{YOUR_PROJECT}}.deno.dev' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer {{YOUR_API_KEY}}' \
--data-raw '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {
        "role": "system",
        "content": "You are a helpful assistant."
      },
      {
        "role": "user",
        "content": "Hello!"
      }
    ]
  }'
```
## 最后
当然，里面代码只是提供一些思路，仅供参考。Deno牛逼🐶。    
