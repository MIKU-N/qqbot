## API 代理配置使用指南

### 功能说明

通过配置 `apiProxy` 字段，可以让所有 QQ Bot API 请求都经过自定义的中转服务器，这样请求的源 IP 就会变成中转服务器的 IP。

### 配置方式

在你的配置文件中添加 `apiProxy` 字段：

```yaml
channels:
  qqbot:
    appId: "your_app_id"
    clientSecret: "your_secret"
    # 配置 API 代理服务器地址
    apiProxy: "https://your-proxy-server.com"
```

### 多账户场景

对于多个 QQBot 账户，可以为每个账户单独配置代理：

```yaml
channels:
  qqbot:
    accounts:
      account1:
        appId: "app1_id"
        clientSecret: "app1_secret"
        apiProxy: "https://proxy1.example.com"
      account2:
        appId: "app2_id"
        clientSecret: "app2_secret"
        apiProxy: "https://proxy2.example.com"
```

### 工作原理

设置代理后，以下请求会被重定向：

1. **Token 获取请求**：
   - 原始：`https://bots.qq.com/app/getAppAccessToken`
   - 代理后：`https://your-proxy-server.com/app/getAppAccessToken`

2. **API 请求**：
   - 原始：`https://api.sgroup.qq.com/v2/users/{openid}/messages`
   - 代理后：`https://your-proxy-server.com/v2/users/{openid}/messages`

### 中转服务器实现示例

你的中转服务器需要：

1. 接收来自 QQBot 客户端的请求
2. 转发这些请求到官方 QQ Bot API
3. 将响应返回给客户端

简单示例（Node.js）：

```javascript
const http = require('http');
const https = require('https');
const url = require('url');

const server = http.createServer((req, res) => {
  // 解析请求 URL
  const originalPath = req.url;
  
  // 构建转发 URL
  let apiUrl;
  if (originalPath.startsWith('/app/getAppAccessToken')) {
    apiUrl = `https://bots.qq.com${originalPath}`;
  } else {
    apiUrl = `https://api.sgroup.qq.com${originalPath}`;
  }
  
  // 转发请求
  const options = {
    method: req.method,
    headers: req.headers
  };
  delete options.headers.host;
  
  const proxyReq = https.request(apiUrl, options, (proxyRes) => {
    res.writeHead(proxyRes.statusCode, proxyRes.headers);
    proxyRes.pipe(res);
  });
  
  proxyReq.on('error', (err) => {
    res.writeHead(500);
    res.end('Proxy error');
  });
  
  req.pipe(proxyReq);
});

server.listen(3000);
```

### 编程方式设置代理

如果需要在代码中动态设置代理：

```typescript
import { setApiProxy } from 'qqbot';

// 设置代理
setApiProxy('https://your-proxy-server.com');

// 移除代理
setApiProxy(null);
```

### 常见问题

**Q: 代理服务器需要做什么？**
A: 代理服务器只需要简单地将收到的请求转发到官方 API，并将响应返回即可。

**Q: 如果不设置代理会怎样？**
A: 不设置或设置为 null 时，会直接连接到官方 QQ Bot API（`https://api.sgroup.qq.com`）。

**Q: 代理服务器的性能会影响 QQBot 吗？**
A: 会的，额外的网络跳转可能会增加延迟，建议将代理服务器放在良好的网络环境中。
