---
title: "HTTP的安全机制"
date: 2020-08-21T23:17:27+08:00
draft: false
tags:
    - what
    - why
    - HTTP
    - CORS
    - HTTPS
---

最近在遇到了一个跨域问题，所以看了一些HTTP安全相关的资料。
对HTTP又有了一些新的理解 ~~并深深的自我怀疑我真的了解HTTP吗？~~ 。

# HTTP的安全机制要解决的问题

HTTP安全机制要解决的问题就是如何在**公开的**信道上，**从零开始**构建一个**可信的**传输通道。

在这个问题有几个关键的点需要考虑：

1. 公开的
1. 从零开始
1. 可信的

## 公开的

简单理解，公开的就是指的就是在因特网上进行通信。这表明了传输中的数据很容易被监听、篡改以及伪造。

## 从零开始

因为浏览器 *(其实是指 HTTP 的 User Agent,下文不区分两个概念)* 是易于获得的，又因为互联网的用户在不断的新增，这对HTTP的安全机制提出了期望：

最好的情况是安全机制能仅基于HTTP通信建立起来一个可信的传输通道。

## 可信的

如何定义可信的传输通道呢？我把传输通道拆分为客户端，服务端和传输过程三个部分。

服务端希望能够鉴别客户端的真实性，包括

1. 真的是用户甲吗？
1. 真的是用户甲想执行这个请求吗？

对于传输过程来说，服务端与客户端都希望

1. 防篡改
1. 防伪造
1. 防窃取

客户端希望能鉴别服务端的真实性，主要是真的是我想要访问的网站吗？

# 解决问题的两个核心机制

为了解决上述的问题，HTTP引入了TLS层和浏览器实现的同源策略两个核心机制。

## HTTPS

HTTPS解决了除服务端对客户端的要求之外的问题。

对于HTTPS的实现流程不再赘述，资料有很多。主要探讨下HTTPS是如何解决上述问题的。

1. 对于从零开始这个问题，通过非对称加密和引入了根证书来解决。

1. 显然能够解决传输过程的问题。

1. 客户端能够通过根证书与HTTPS证书来鉴别返回的报文是否真的是想要访问的网站返回的。

## 同源策略 Same-Origin policy

同源策略解决了服务端对用户端的两个要求，且是对防窃取的一种补充。

> 同源策略是一个重要的安全策略，它用于限制一个origin的文档或者它加载的脚本如何能与另一个源的资源进行交互。它能帮助阻隔恶意文档，减少可能被攻击的媒介。

先说一个观点，同源策略能够保证**任何第三方不能在服务端和用户都不知情或不配合的情况下，访问非同源的数据或代替用户发起不安全的请求或读取请求的结果**。

### 首先解释几个名词。

- 什么是源？什么是同源？

    对于源这个概念，[MDN](https://developer.mozilla.org/zh-CN/)介绍的很好。

    源就是一个页面的URL的`<schema, domain, port>`的三元元组。

    同源就是这个三元元组相同。

- 什么是第三方？

    第三方就是不同源的页面（上的脚本）。

- 什么是访问非同源的数据？

    如访问非同源的页面的Cookie、localStorage、indexedDB等。

- 什么是不安全的请求？

    除了**简单请求**，就是不安全的请求，称作**需要执行预检的请求**。

    摘自[HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS#若干访问控制场景)

    > 某些请求不会触发 CORS 预检请求。本文称这样的请求为“简单请求”，请注意，该术语并不属于 Fetch （其中定义了 CORS）规范。若请求满足所有下述条件，则该请求可视为“简单请求”：
    > 
    > - 使用下列方法之一：
    >   - GET
    >   - HEAD
    >   - POST
    > 
    > - 除了被用户代理自动设置的首部字段（例如 Connection ，User-Agent）和在 Fetch 规范中定义为 禁用首部名称 的其他首部，允许人为设置的字段为 Fetch 规范定义的 对 CORS 安全的首部字段集合。该集合为：
    >   - Accept
    >   - Accept-Language
    >   - Content-Language
    >   - Content-Type （需要注意额外的限制）
    >   - DPR
    >   - Downlink
    >   - Save-Data
    >   - Viewport-Width
    >   - Width
    > - Content-Type 的值仅限于下列三者之一：
    >   - text/plain
    >   - multipart/form-data
    >   - application/x-www-form-urlencoded
    > - 请求中的任意XMLHttpRequestUpload 对象均没有注册任何事件监听器；XMLHttpRequestUpload 对象可以使用 XMLHttpRequest.upload 属性访问。
    > - 请求中没有使用 ReadableStream 对象。

### 为什么POST请求会被认为是简单请求？

因为历史上，表单的提交是跨域允许。为了前向兼容，所以做出这种设计。

那么带来的问题就是，对于Content-Type是text/plain等三种的POST请求要特别注意检查是否是不合预期的跨域访问，而且这个检查要在**执行影响数据的操作之前做**。

### 同源策略是如何运作的？

浏览器会阻止一个页面对不同源的页面进行如下操作：

*这些操作会不断调整来适应未来的变化*

- 发送请求（XMLHttpRequest或Fetch）
- 访问Cookie/localStorage/indexedDB
- 访问DOM

1. 浏览器通常会将发送请求分为**简单请求**和**需要预检的请求**来处理

    1. 简单请求会直接发送，如果返回的结果不包含符合预期的CORS头，那么会阻止脚本获得结果。
    1. 需要预检的请求会先发送一个`OPTIONS`请求，来获取CORS的配置，避免在不允许访问的时候，对数据产生影响。

    如此，可以保证只有在服务端知道的情况下，才能跨域访问相应的接口来获取数据或者执行操作。

    ```plantuml
    @startuml cors-origin request

    == 简单请求 == 

    actor 脚本 as script
    participant 浏览器 as browser
    participant 一个没有支持该源跨域访问的服务端 as server

    script -> browser: 一个请求其他域的简单请求
    activate browser
    browser -> server: 发送请求
    activate server
    note right: 请求会真实到达服务器
    server -> browser: 响应报文，携带的CORS配置不支持该源的请求
    deactivate server
    browser -> script: 错误：禁止跨域
    deactivate browser
    note right: 只是结果会被拦截

    == 非简单请求 ==


    script -> browser: 一个请求其他域的非简单请求
    activate browser
    browser -> server: 发送一个options请求
    activate server
    note right: 浏览器会先发送一个\noptions请求来查询\n服务器的跨域配置，\n真实的请求不会被发送到服务器
    server -> browser: 返回结果，携带的CORS配置不支持该源的请求
    deactivate server
    browser -> script: 错误：禁止跨域
    deactivate browser

    @enduml
    ```

1. 浏览器会阻止一个页面上的脚本访问非同源的localStorage/indexedDB或非同父域且非同源的Cookie

    这样就能够保证客户端的Cookie等不会在用户不知情的情况下被窃取，这是基于Cookie保存登录态的基础。

1. 浏览器会阻止一个页面上的脚本访问非同源的页面的DOM

    这样能够保证页面上的数据不被非同源的页面的脚本窃取。

### 在同源策略下，如何允许指定的非同源的操作呢？

*同源策略是一个积极的机制，对于HTTP是有效的补充，所以我个人不太喜欢**绕过**之类的描述。*

值得一提的是，这些操作的方式，都是需要被跨域的一侧进行支持的，即不满足**不知情**的条件。

本文暂不具体介绍各种方式实现的细节，仅提供一些参考。

1. 【推荐】【允许跨域请求】CORS Cross-Origin Resource Sharing

    这是协议推荐的标准实现。

1. 【允许跨域请求】 JSONP

    原理是`script`标签的加载，不被同源策略限制。

    服务端需要配合处理特定的js文件加载请求。

1. 【推荐】【允许访问DOM或存储】 html5的`postMessage()`

1. 【允许访问DOM或存储】 iframe通过fragment/name通信

### 一直在说的用户不知情的情况是什么呢？

因为HTTP请求是可以任意构造的，如果用户是个懂哥，就可以绕过这些限制。

# 参考

- [MDN对于同源策略的介绍](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
- [MDN关于HTTP访问控制CORS的文章](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
