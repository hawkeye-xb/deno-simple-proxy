## 一分钟将代理OpenAI 的API，无需搬梯子
作为前端开发，一直听说Deno的大名，但遗憾的是至今还没有亲自尝试过编写一个简单的"Hello World"程序。现在，我打算利用Deno来实现一个简单的API代理服务，这旨在用于技术交流和学习，才不是因
为我发现了可以通过[Deno Deploy](https://deno.com/deploy/pricing)的免费套餐来代理访问OpenAI的接口。这是一个探索和学习的过程，希望我们可以一起讨论和优化🐶。

### 第一步
登录[https://dash.deno.com/projects](https://dash.deno.com/projects)，New Playground

### 第二步
复制下面代码
```JS
const server = Deno.serve(async (request) => {
  if (request.method === "GET") {
    return new Response("Hello Deno\n from GET method");
  }
  if (request.method === "POST") {
    try {
        const body = await request.text();
        const response = await fetch("https://api.openai.com/v1/chat/completions", {
            method: "POST",
            headers: {
                "Content-Type": "application/json",
                "Authorization": "Bearer {{KEY}}"
            },
            body,
        });
        
        const responseBody = await response.text();
        return new Response(responseBody, {
            status: response.status,
            headers: response.headers,
        });
    } catch (error) {
        return new Response({ status: 500, body: `Server error: ${error.message}` });
    }
  }

  return new Response("hello world!\n");
}, { port: 8000 });
```

### 第三步
[https://platform.openai.com/api-keys](https://platform.openai.com/api-keys) 从里面来个OpenAI 的API key，填到代码上面。或者不填，请求这个接口时候带上。

### 第四步
Save & Deploy

### 验证
终端curl一哈
```shell
curl --location --request POST 'https://{{YOUR_PROJECT}}.deno.dev' \
--header 'Content-Type: application/json' \
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
省得自建云服务，还需要另外的节点倒一手才能请求了。