### 概念

黑客把恶意的脚本代码注入到你的网页中，而浏览器无法分辨这是正常代码还是攻击代码，从而在受害者的浏览器上执行。

### 分类

A. 存储型 XSS（Stored XSS）

- **攻击过程**：恶意代码被发送到服务器并永久**存储在数据库**中。
- **触发场景**：评论区、个人信息修改、论坛发帖。
- **危害**：最大。只要有用户访问那个含有恶意代码的页面，就会中招。

B. 反射型 XSS（Reflected XSS）

- **攻击过程**：恶意脚本通过 **URL 参数**传递给服务器，服务器不加处理地将脚本随响应返还给浏览器。
- **触发场景**：搜索框、筛选功能。
- **例子**：用户点击了 `http://xxx.com/search?q=<script>alert('攻击')</script>`。

C. DOM 型 XSS

- **攻击过程**：完全发生在**前端浏览器**中。不经过后端服务器。
- **原理**：前端 JS 代码直接从 URL（如 `location.hash`）获取数据，并通过 `innerHTML` 或 `document.write` 等危险操作插入到页面中。
- **关键点**：它和反射型的区别在于，恶意代码不会到达服务器。

### 攻击者能拿 XSS 做什么

- **窃取 Cookie**：通过 `document.cookie` 获取会话令牌（Token），伪造登录。
- **劫持流量**：修改 `window.location` 强制跳转到钓鱼网站。
- **修改 UI**：在页面插入假的登录框，诱导用户输入账号密码。
- **监听行为**：使用 `addEventListener` 监听用户的键盘输入（Keylogging）。

### 如何防御 XSS

A. 对输入进行过滤与转义（最基础）

- **转义关键字符**：将 `<` 变成 `<`，`>` 变成 `>`。
- **使用库**：不要自己写正则，推荐使用成熟的库如 **DOMPurify**，它可以把 HTML 中的危险标签（如 `<script>`, `<iframe>`）和属性（如 `onclick`）清洗掉。

B. 使用安全的 API

- **Vue / React 优势**：它们默认就是安全的。比如 Vue 的 `{{ }}` 插值会自动进行转义。
- **避开危险操作**：少用 `v-html` (Vue) 或 `dangerouslySetInnerHTML` (React)。尽量使用 `innerText` 或 `textContent`。

C. 设置 HTTP 响应头

- **HttpOnly**：给敏感 Cookie 加上 `HttpOnly` 标识。这样 JS 就无法通过 `document.cookie` 读取到它，即使被 XSS 攻击了，Token 也拿不走。
- **CSP (Content Security Policy)**：**这是目前防御 XSS 的终极手段**。
  - 通过白名单限制浏览器只能加载特定域名的脚本、图片、插件。
  - 例如：`Content-Security-Policy: script-src 'self'` 表示只允许加载同源脚本，禁止一切外链脚本和内联脚本。