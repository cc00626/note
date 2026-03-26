### 概念

CSRF 是一种挟持用户终端，在用户不知情的情况下，以用户的名义向受信任网站发起恶意请求的攻击方式。

**简单来说：** 攻击者盗用了你的**身份（Cookie）**，以你的名义发送了一个 HTTP 请求（比如发帖、转账、改密码）



### 流程

要完成一次 CSRF 攻击，通常需要满足以下两个条件：

1. 用户已经登录了站点 A，并在浏览器中保留了 **登录凭证（Cookie）**。
2. 用户在没有登出 A 的情况下，访问了恶意站点 B。

**攻击过程：**

1. 你登录了银行网站 `bank.com`。
2. 黑客诱导你点击一个链接，进入了 `evil.com`。
3. `evil.com` 页面加载了一段代码（或一个隐藏的表单），自动向 `bank.com/transfer?to=hacker&amount=1000` 发起请求。
4. **关键点：** 浏览器发现这个请求是发往 `bank.com` 的，会自动把你在 `bank.com` 的 Cookie 带上。
5. 银行服务器收到请求，发现 Cookie 合法，认为是你在操作，转账成功。

###  CSRF 的常见表现形式

**A. GET 类型**

利用 `<img>`、`<a>` 标签的 `src` 或 `href` 属性。

HTML

```
<img src="http://bank.com/transfer?to=hacker&amount=1000">
```

浏览器在加载图片时会自动发起 GET 请求。

**B. POST 类型**

使用一个隐藏的表单。

HTML

```
<form id="hacker-form" action="http://bank.com/transfer" method="POST">
    <input type="hidden" name="to" value="hacker">
    <input type="hidden" name="amount" value="1000">
</form>
<script>document.getElementById('hacker-form').submit();</script>
```

页面一加载就自动提交表单，用户完全无感。



### 解决

**A. 验证 Referer 和 Origin (请求源检查)**

- **原理**：后端检查 HTTP 请求头中的 `Referer`（来源页面）或 `Origin`（请求域名）。
- **限制**：有些浏览器可以关闭 Referer，或者在 HTTPS 跳转 HTTP 时不带 Referer，且 Origin 并不总是存在。

**B. CSRF Token（主流方案）**

- **原理**：服务器给用户生成一个随机的 Token，存放在 Session 中。前端每次发起请求时（尤其是 POST），必须在请求体或自定义 Header 中带上这个 Token。
- **为什么有效**：黑客虽然能诱导浏览器带上 Cookie，但他**无法获取**到你页面里的 Token（受同源策略限制）。服务器校验 Token 不匹配，则拒绝请求。

**C. 双重 Cookie 校验 (Double Submit Cookie)**

- **原理**：利用黑客无法读取 Cookie 但浏览器会自动带上 Cookie 的特性。在请求参数中也带上一份 Cookie 中的值，后端对比两者。

**D. Samesite Cookie 属性（现代浏览器最强防御）**

- **原理**：给 Cookie 设置 `SameSite` 属性。
  - `Strict`：完全禁止第三方 Cookie，只有在本站发起的请求才会带 Cookie。
  - `Lax`（默认）：大多数情况不带，只有导航到目标网址的 GET 请求（如链接跳转）才带。
- **优点**：从浏览器底层直接切断了攻击路径。