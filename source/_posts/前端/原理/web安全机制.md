---
title: web安全机制
category:
  - 前端
  - 原理
tags:
  - 前端
  - 原理
  - 安全
keywords: 安全
abbrlink: 8219c7af
date: 2019-10-20 06:00:00
updated: 2019-10-20 06:00:00
---

常见的前端安全问题包含 XSS（Cross Site Script，跨站脚本攻击）、CSRF（Cross-site Request Forgery，跨站请求伪造）、点击劫持等。

### 浏览器安全

#### 同源策略

浏览器的同源策略限制了 document 或脚本的跨源读写能力。若域名、子域名、端口、协议其中一个有所不同，浏览器就会认为两个站点是跨源的。当 script、img、iframe、link 等包含 src 属性的标签加载远程资源，即便资源提供的域名和页面的域名不一致，浏览器还是会认为资源的 origin 为当前页面的所在域。与 xhr 不同的是，通过 src 加载的资源，浏览器限制了 js 读写响应的能力。在不启用 CORS 功能的站点中，xhr 只能发起同源请求。除 xhr 外，cookie 也会受同源策略的限制，即，理论上不能在非源站点中获取原站点的 cookie。

#### sandbox

浏览器采用多进程架构隔离多个功能模块、tab。Chrome 的主要进程为：浏览器进程、渲染进程、插件进程、扩展进程。插件进程与浏览器进程严格隔离。渲染进程通过沙箱隔离，网页代码与浏览器内核进程通信、与操作系统通信都需要通过 IPC channel。沙箱使不可信的代码运行在一定的环境中，限制不可信的代码访问隔离区外的资源；如果一定要访问隔离区外的资源，就必须通过指定的数据通道，由特定的 API 校验数据的合法性。浏览器主要用于限制 js 在安全区内执行；多进程架构也使一个 tab 挂掉后，不会影响另一个 tab。

![image](multi-process.jpeg)

#### 恶意网址拦截

基于页面特征模型，浏览器厂家会将恶意网址加入黑名单中，并推送到客户端，保障用户访问的安全性。网址黑名单并不一定由浏览器厂家收集获得，也可以借助第三方安全厂商提供。

### XSS

XSS 由可解析执行的代码插入到页面中引起；在页面渲染过程中，这段代码会被执行，从而引起安全问题（如盗取数据、伪造账号等）。因为恶意代码可能是加载特定 js 资源的 script 标签，所以 XSS 攻击引起的问题包含通过 js 代码劫持 cookie、伪造请求等；XSS 也可以通过 js 画出登录框，把用户的提交数据发送给黑客电脑，从而造成账号密码的泄漏；XSS 可以通过 window.name 在非源站点中获取源站点的 cookie 信息（window 为浏览器窗体，不像 document 那样受同源策略的影响，因此将 cookie 赋值给 window.name 就可以在非源站点获取到 cookie 信息）；XSS 攻击也能通过 UserAgent 获取用户的操作系统和浏览器版本等信息，并逐步挖掘到用户安装的软件、浏览器的扩展程序，再通过浏览器漏洞植入木马。XSS 甚至可以通过 Java Applet、Flash、iTunes、Office Word、QuickTime 等第三方软件获取用户的 ip 地址。更有甚者，XSS Worm 蠕虫会利用社交平台的好友列表，能使恶意代码迅速扩散。

测试 XSS 攻击平台有 Attack API、BeFF、XSS-Proxy、

XSS 分为存储型 XSS、反射型 XSS、DOM XSS（MXSS）三种。存储型 XSS 是指恶意代码随着提交数据入库，再从数据库读取回显时引起恶意代码的执行。因此，存储型 XSS 能引起稳定持续的安全问题。反射型 XSS 是指将用户输入“反射给”浏览器，即需要一个交互行为才能使攻击成功。典型的交互行为如点击一个恶意链接，通过 url 参数植入恶意脚本，url 参数在使用过程时将执行恶意脚本（所以 react 设置了 dangerouslySetInnerHTML 属性，es6 提供模板字符串处理变量）。黑客也可以通过 location.hash 构造 url 参数，因为 hash 路由不会造成发包请求，这样服务器日志也不会留有记录，也就隐藏了黑客的真实意图。MXSS 由恶意代码段代插入 DOM 属性中引起，效果等同反射型 XSS。

XSS 的防范方式有以下几种：

1. 在 cookie 中设置 httpOnly 标识，禁止通过 js 访问 cookie，以免 cookie 被劫持。
2. 提交表单时通过验证码等交互行为限制接口调用的成功率，这样就能在一定程度上避免 XSS 伪造请求。
3. 验证所有输出到页面上的数据，并对必要的内容作转义处理（成熟的模板引擎需要防范使用模板变量进行 XSS 攻击）。
4. 服务端使用开源的 XSS Filter 检查输入。
5. 对 url 参数进行加密，避免攻击者伪造（加密后的 url 不利于用户收藏）。

以下是常见的转义处理：

```javascript
function htmlEncode(str){
  if (str.length === 0) return '';

  return str.replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/ /g, '&nbsp;')
    .replace(/\'/g, '&#39;')
    .replace(/\"/g, '&quot;')
    .replace(/\n/g, '<br>');
};

function htmlDecode(str){
  if (str.length === 0) return '';

  return str.replace(/&amp;/g, '&')
    .replace(/&lt;/g, '<')
    .replace(/&gt;/g, '>')
    .replace(/&nbsp;/g, ' ')
    .replace(/&#39;/g, '\'')
    .replace(/&quot;/g, '\"')
    .replace(/<br>/g, '\n');
};
```

### CSRF

CSRF 是非源站点携带着源站点的 cookie 数据，继而调用源站点的接口，这样就容易避过用户校验，并使请求执行成功（如篡改金额后调用支付接口）。浏览器 cookie 分为两种：指定失效时间的本地 cookie，浏览器在切换 tab 到子域时不会将该 cookie 带入另一个 tab；未指定失效时间的临时 cookie，在浏览器进程的生命周期中有效，且可以在不同 tab 子域中传播。部分浏览器并未禁止不同域的 iframe, img, script, link 等标签携带本地 cookie，但是所有浏览器允许携带临时 cookie。为嵌入第三方广告等 iframe，W3C 指定的 P3P 头允许 iframe 等标签携带本地 cookie，这样做既使第三方广告能获得 cookie 信息，也增加了遭受 CSRF 攻击的风险。在 CSRF 攻击中，攻击者会伪造嵌入源站点 iframe 的非源站点获取 cookie 信息，同时可以使用 js 脚本伪造 post 请求等。

CSRF 的防范方式有以下几种：

1. 提交表单时通过验证码等交互行为限制接口调用的成功率，这样就能在一定程度上避免 CSRF 伪造请求。
2. 校验 Referer 头是不是指定的页面，但是服务器未必能获取到 Referer 头。
3. 对 url 参数进行加密，避免攻击者伪造（加密后的 url 不利于用户收藏）。
4. 添加随机且加密且有一定实效的 token。token 由后端生成，可通过 html 模板变量、session、cookie 传给前端；再由前端添加到请求体或请求头中；提交数据后，后端将验证 Token 以判断用户的准确性。

### 点击劫持

* 点击劫持：在非源站点中使用隐藏的 iframe 加载源站点页面，再通过诱导用户点击源站点的按钮等，产生恶意行为。
图片覆盖攻击：与点击劫持相类，仍是用 iframe 加载源站点页面，再诱导用户点击绝对定位的图片。
* 拖拽劫持：拖拽不受同源策略的限制，在使用 ifame 加载源站点页面的情况下，就能通过拖拽脚本窃取源站点的信息。
* 触屏劫持：与点击劫持相类。

防御点击劫持的手段其一是禁止 iframe 嵌套，这种做法叫做 frame busting。frame busting 是可以被绕过的，详情参看 “Busting frame busting: a study of clickjacking vunlnerabilities at popluar site”。以下是 frame busting 的简单示例。防御点击劫持的手段其二是通过 X-Frame-Options 限制 iframe 加载，这样可以回避 frmae busting 的被绕过。

```javascript
// frame busting 条件语句
if (top != self)
if (top.location != self.location)
if (top.location != location)
if (parent.frames.length > 0)
if (window != top)
if (window.top !== window.self)
if (window.self != window.top)
if (parent && parent != window)
if (parent && parent.frames && parent.frames.length>0)
if((self.parent&&!(self.parent===self))&&(self.parent.frames.length!=0))

// frame busting 纠正错误语句
top.location = self.location
top.location.href = document.location.href
top.location.href = self.location.href
top.location.replace(self.location)
top.location.href = window.location.href
top.location.replace(document.location)
top.location.href = window.location.href
top.location.href = "URL"
document.write('')
top.location = location
top.location.replace(document.location)
top.location.replace('URL')
top.location.href = document.location
top.location.replace(window.location.href)
top.location.href = location.href
self.parent.location = document.location
parent.location.href = self.document.location
top.location.href = self.location
top.location = window.location
top.location.replace(window.location.pathname)
window.top.location = window.self.location
setTimeout(function(){document.body.innerHTML='';},1);
window.self.onload = function(evt){document.body.innerHTML='';}
const url = window.location.href; top.location.replace(url)

// 示例
if(top.location != location){
  top.location = self.location;
}
```

### html5

在 html5 中，新标签可能会产生 XSS 攻击，比如通过 video 的绑定事件执行恶意代码。为此，HTML5 Security Cheatsheet 项目统计了一些安全问题。

html5 为 iframe 添加了 sandbox 属性，可选值 allow-same-origin 允许同源访问、allow-top-navigation 允许访问顶层窗口、allow-forms 允许提交表单、allow-scripts 允许执行脚本。这样就大大提高了使用 iframe 的安全性。

html5 为 a、area 的 rel 属性添加了 noreferer 值，即在跳转链接时不携带源站点的地址信息，以免敏感信息的泄漏。

html5 的其他安全问题包含：利用 canvas 可以破解图片验证码；CORS 当后端设置 Access-Control-Allow-Origin: * 时，源站点允许任意的跨域请求；window.postMessage 在不检验 domain 域的前提下，可以接受来自任何页面的消息；localStorage、sessionStorage 提供了保存恶意代码的可能，容易引起 XSS 攻击。

### 参考

《白帽子讲安全》
[Busting Frame Busting: a Study of Clickjacking Vulnerabilities on Popular Sites](https://www.cnblogs.com/LittleHann/p/3386055.html)