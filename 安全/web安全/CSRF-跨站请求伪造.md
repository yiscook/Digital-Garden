---
title: CSRF-跨站请求伪造
tags:
  - web安全
creation date: 2024-03-19
done: true
---
# 1. CSRF概述  
---
>[!note]
>**跨站请求伪造**（Cross-Site Request Forgery），攻击者通过诱导用户点击链接或访问恶意网站，实现对用户 Web 应用的操作，例如修改账户信息、发送恶意请求等，从而达到攻击的目的。简单说：挟制用户在当前**已经登录**的 web 应用程序上执行非本意操作的攻击方法。  

**与 SSRF、XSS 漏洞的异同点：**  
- CSRF 针对用户，SSRF 针对服务器；  
- 与 XSS 都存在跨站，都需要诱导用户点击恶意链接等内容；  
- XSS 属于一种输入攻击，CSRF 基于 web 隐式身份验证机制。  

>[!tip]
>Web 的隐式身份验证机制：Web 应用程序在不需要用户显式提供身份验证凭据（账号密码等信息）的情况下，通过使用与用户相关的数据来验证用户身份的一种方法。这些数据可能包括用户的 IP 地址、User-Agent 字符串、Cookie 等。  
>例如在线购物网站，用户在登录后，其浏览器中的 Cookie 会保存其登录凭据。当用户访问该网站时，Web 应用程序会检查用户浏览器中的 Cookie，以验证用户的身份。  

## 1.1 CSRF 攻击流程  
![CSRF攻击流程](https://image.yiscook.top/blog-image/202403191716367.png)  

## 1.2 CSRF 攻击条件  
- 一个相关的动作。应用程序中存在攻击者有理由诱导的操作。这可能是特权操作（例如修改其他用户的权限）或针对用户特定数据的任何操作（例如更改用户自己的密码）。  
- 基于 cookie 的会话处理。执行该操作涉及发出一个或多个 HTTP 请求，应用程序仅依赖会话 cookie 来识别发出请求的用户。没有其他机制来跟踪会话或验证用户请求。  
- 没有不可预测的请求参数。执行操作的请求不包含任何参数-其值攻击者无法确定或猜测。例如，当导致用户更改密码时，如果攻击者需要知道现有密码的值，则该函数不易受到攻击。  

## 1.3 CSRF 分类  
### **1.3.1 GET 请求-资源包含**  
需要尽可能知道各个参数所代表的含义，构造完整的 url 链接，诱导用户点击连接，或者通过其他方式加载连接（钓鱼邮件、图片等）。  
一些简单的 CSRF 攻击可以完全独立使用易受攻击网站上的单个 URL，这种情况就不需要使用外部占点，可以直接构造恶意 URL。例如：  
1. 假设一个网站有一个修改密码的功能，用于允许已经登录的用户更改他们的密码。该网站使用 GET 请求来执行密码更改，URL 为 `https://example.com/password.php?new_password=123456`。  
2. 攻击者在另一个网站上放置了一个链接，链接地址为`https://example.com/password.php?new_password=malicious_password`  
3. 当用户访问攻击者的网站，并单击链接时，就会在用户不知情的情况下修改他们在原始网站上的密码。  

### **1.3.2 POST 请求-基于表单**  
**常见的攻击流程如下：**  
1. 攻击者构造恶意页面，包含一个伪造的表单；  
2. 该表单中包含了攻击者需要实现的恶意操作相关参数；  
3. 用户访问该页面，表单自动提交，向服务器发送 POST 请求；  
4. 如果用户已经登录了受攻击站点，并且在攻击者构造的表单中包含了一些能够触发敏感操作的参数，那么这个操作就会在用户不知情的情况下被执行。  
**常用payload方法：**  
- 利用 `img` 标签：  
    会向受害者的 `/profile` 页面发送一个 POST 请求，请求内容为 `email=attacker@email.com`，即将受害者的邮箱地址修改为攻击者的邮箱地址。  
    `<img>` 标签中设置了 `style="display:none"`，意味着该标签将不会在页面上显示出来，因此受害者并不会察觉到任何异常。但是，由于该标签中的 `onload` 事件，当该标签被浏览器加载时，JavaScript 代码会被执行，发送 POST 请求修改受害者的邮箱地址。  
    ```HTML  
    <img src="https://example.com/profile" style="display:none"  
         onload="var xhr = new XMLHttpRequest();  
                 xhr.open('POST', 'https://example.com/profile');  
                 xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');  
                 xhr.send('email=attacker@email.com');">  
    ```  
- 利用 `form` 标签：  
    创建了一个隐藏的表单，该表单会在页面加载时自动提交，从而触发 POST 请求来修改受害者的邮箱地址。  
    由于表单可以被放置在任何页面的任何位置，攻击者可以将其隐藏得很好，使其不易被发现。  
    ```HTML  
    <form action="https://example.com/profile" method="POST" style="display:none">  
      <input type="hidden" name="email" value="attacker@email.com">  
    </form>  
    <script>  
      document.forms[0].submit();  
    </script>  
    ``` 
 
# 2. 漏洞发现  
---
CSRF 可以说存在于任何的增删改操作中，常见的有：  
- 冒充分身：订阅、关注、转发投票等操作，删除文件，更改配置等。  
- 账户接管：修改密码、重新绑定邮箱、第三方账户关联。  
- 其他：登录、注册、注销等  
- 安全设计原则：登录有token未更新等。  

## 2.1 referer 检测  
HTTP 头中有一个 referer 字段，这个字段用以标明请求来源于哪个地址。在处理敏感数据请求时，会对 referer 字段进行验证。如果发现空 referer 或者非指定域名跳转会拦截此类请求。关注每一个关键操作的请求包，若参数中没有 CSRF 令牌参数，篡改 referer 仍然返回正常，则大概率存在 CSRF 漏洞。  
绕过方法：  
- 修改 referer 或者空 referer：将 http 请求头的 referer 字段进行修改或者删除，在发包观察响应包的内容。  
- 包含 referer：若只检测是否包含目标网址，可以在自己的网站上创建网站的文件或文件夹。  
    例如：`https://www.minewebsite.com/https://www.targetwebsite.com/`  

## 2.2 CSRF token 检测  
CSRF Token 是一种用于保护 Web 应用程序免受跨站点请求伪造攻击（CSRF 攻击）的技术。它是一个随机生成的字符串，由 Web 应用程序在页面中生成，并在后续的请求中通过 cookie 或请求参数传递给服务器端。服务器端在处理每个请求时都会检查 CSRF Token 是否有效，以确保该请求是由合法的用户发送的。  
绕过方法：  
- 删除 token：删除 cookie 或者参数中的 token，尝试绕过服务器的验证。  
- token 共享：使用两个不同的账户，替换 token 查看是否可以相互公用。  
- 篡改 token 值：如果系统检测 token 不够严格，可以尝试篡改 token，保留长度相同。  
- 解码 token：**尝试进行 MD5**或者**base64**解码，观察期格式，并且自行构造。    

## 2.3 CORS 检测  
>[!note]
>**跨站资源共享**（Cross Origin Resource-Sharing），CORS 是一种用于跨域访问资源的机制。在 Web 应用程序中，由于浏览器的同源策略，普通的跨域请求是不被允许的，CORS 提供了一种机制来让 Web 应用程序在跨域访问时能够安全地获取对指定资源的访问权。  
>当网站允许跨域资源共享（CORS）时，攻击者可以构造恶意请求，利用网站对跨域请求的信任来发起 CSRF 攻击。  

**如果网站使用了CORS，请求头中常会添加以下字段：**  
- `Origin`：该字段标识了当前请求的源站信息，包含了协议、主机名和端口号等信息。  
- `Access-Control-Request-Method`：当浏览器发起跨域请求时，会先发送一个预检请求，该字段标识了实际请求使用的 HTTP 方法，例如 GET、POST等。  
- `Access-Control-Request-Headers`：当实际请求需要额外的自定义请求头时，例如 Authorization 等，该字段就会将这些请求头告诉服务器。  

**当服务器确认该请求合法后，会在响应头中添加以下字段：**  
- `Access-Control-Allow-Origin`：该字段指示了当前响应可以被哪些源站访问，可以是 * 或具体的源站地址。  
- `Access-Control-Allow-Methods`：该字段标识了当前响应支持的 HTTP 方法。  
- `Access-Control-Allow-Headers`：该字段标识了当前响应支持的自定义请求头。  
- `Access-Control-Allow-Credentials`：该字段指示了是否允许发送 Cookie 等敏感信息。  
### 2.3.1 绕过方法  
>[!example]
>假设存在如下情况：  
>`vulnerable.com`：目标网站  
>`/api/v1/updateEmail`：更新邮箱的api  
>`attacker%40example.com`：攻击者邮箱  

- Origin 篡改：攻击者可以通过修改请求头中的 Origin 字段，将其设置为被攻击网站的域名。如果受害者在此网站登录过，浏览器会在请求时自动携带登录凭证，导致被攻击网站认为请求是合法的，从而执行攻击者设定的恶意请求。  
    ```HTML  
    POST /api/v1/updateEmail HTTP/1.1  
    Host: vulnerable.com  
    Origin: https://vulnerable.com  
    Content-Type: application/x-www-form-urlencoded  
    Content-Length: 33  
      
    email=attacker%40example.com  
    ```  
- 跳转攻击者页面：攻击者网站核心代码如下。这部分代码缺点是没有直接返回到更新邮箱的页面。  
    ```HTML  
    <script>  
        var xhr = new XMLHttpRequest();  
        xhr.open('POST', 'http://vulnerable.com/api/v1/updateEmail', true);  
        xhr.withCredentials = true;  
        xhr.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');  
        xhr.setRequestHeader('Origin', 'https://attacker.com');  
        xhr.send('email=attacker@example.com');  
    </script>  
    ```  
    进阶方法：这个 payload 会在页面加载后首先发送一个 GET 请求到 `https://vulnerable.com/profile`，以获取**CSRF 令牌**。它使用正则表达式从响应中提取令牌，并将其添加到 POST 请求中以更新电子邮件地址。最后，它创建一个新的隐藏表单并在其中添加 CSRF 令牌，然后将表单提交以完成攻击。注意，这个 payload 也设置了 `withCredentials` 属性来启用跨域资源共享（CORS）并使用了 POST 方法来发送请求。  
    ```HTML  
    <form id="csrf-form" action="https://vulnerable.com/api/v1/updateEmail" method="POST">  
      <input type="hidden" name="email" value="attacker@example.com" />  
    </form>  
    <script>  
      const form = document.getElementById('csrf-form');  
      const xhr = new XMLHttpRequest();  
      xhr.open('GET', 'https://vulnerable.com/profile');  
      xhr.withCredentials = true;  
      xhr.onload = () => {  
        const csrfToken = xhr.response.match(/name="csrf_token"\s*value="(\w+)"/)[1];  
        const input = document.createElement('input');  
        input.type = 'hidden';  
        input.name = 'csrf_token';  
        input.value = csrfToken;  
        form.appendChild(input);  
        document.body.appendChild(form);  
        form.submit();  
      };  
      xhr.send();  
    </script>  
    ```  
- 预检请求：预检请求是在发送 CORS 请求之前，先向服务器发送一个 OPTIONS 请求，目的是为了询问服务器是否允许实际请求，从而防止非同源网站的恶意请求。攻击者可以伪造 OPTIONS 请求，实现绕过预检请求。  
    ```HTML  
    OPTIONS /api/v1/updateEmail HTTP/1.1  
    Host: vulnerable.com  
    Access-Control-Request-Method: POST  
    Access-Control-Request-Headers: Content-Type  
    Origin: https://vulnerable.com  
    ```  

### 2.3.2 CORScanner  
CORS 跨域扫描器，用来检测网站的 CORS 配置是否安全。  
[GitHub - chenjj/CORScanner: 🎯 Fast CORS misconfiguration vulnerabilities scanner](https://github.com/chenjj/CORScanner)  

# 3. CSRF token  
---
CSRF token 是一种用于防范 CSRF 攻击的安全机制。是一个唯一的、秘密的、不可预测的值。它由服务器端的应用程序生成并发送给客户端。客户端后续的 http 请求都需要包含该 token。收到请求时，服务器端应用程序会验证是否包含预期的 token，如果 token 丢失或失效，拒绝请求。  
## 3.1 生成 CSRF token  
**生成 CSRF token 需要注意的规则：**  
1. 需要有很强的**不可预测性。**  
2. 每次生成的 token 必须是**不相同**的  
3. 使用加密安全的伪随机数生成器（CSPRNG），在创建的时候使用时间戳最为种子加上静态加密。  
4. 如果 token 的强度需要进一步加强，可以与用户 id 等用户信息进行连接生成用户单独的标记。并对整体进行强 hash 加密。  
需要注意：token长度越长，越难被猜测与破解。但是同时也会给服务器增加负担。  

## 3.2 传输 CSRF token  
CSRF token 应该被视为秘密，并在整个生命周期中以安全的方式进行处理。  
通常有效的方法是使用 POST 方式，在提交表单以隐藏字段将 token 传输到客户端。并且 CSRF token 不可在 cookie 中进行传输。  

# 4. 参考内容  
---
[同源策略与跨域请求](https://mp.weixin.qq.com/s?__biz=MzI5MDQ2NjExOQ==&mid=2247487543&idx=1&sn=924a5d5f37fa27d053187cf6a740ba8e&chksm=ec1e201fdb69a909bda97b7e5af3aaffb2853a95694310d2463fe139468b96a55cf7e25edc6c&mpshare=1&scene=23&srcid=#rd)  
[portswigger csrf lab 跨站请求伪造靶场wp - 🔰雨苁ℒ🔰](https://www.ddosi.org/csrf-lab/#CSRF_tokens)  

# 5. 漏洞修复和预防
---
- 通过 CSRF token 来检测用户的提交；  
- 验证请求中的 referer 字段和 content type 字段；  
- 避免使用通用的 cookie，最好设置较为严格的 cookie 域；  
- 限制 POST 请求的来源，对于敏感操作只接受来自特定的 POST 请求。
