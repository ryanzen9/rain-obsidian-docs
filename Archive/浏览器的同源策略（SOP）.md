
URL: https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy?utm_source=chatgpt.com
上次编辑时间: 2026年4月12日 17:09
创建时间: 2025年11月1日 17:13
标签: 知识记录
状态: 完成
目标追踪: 计算机知识 (https://www.notion.so/2a1a89f7e47d80c2a796c6001e84538a?pvs=21)

[](%E6%B5%8F%E8%A7%88%E5%99%A8%E7%9A%84%E5%90%8C%E6%BA%90%E7%AD%96%E7%95%A5%20%EF%BC%88SOP%EF%BC%89%20-%20%E5%AE%89%E5%85%A8%20MDN/stn-1Sa3ezoEA202vzlK73N1Jq55jqjSV4BuT3GFh4ER.svgxml)

## [源的定义](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E6%BA%90%E7%9A%84%E5%AE%9A%E4%B9%89)

如果两个 URL 的[协议](https://developer.mozilla.org/zh-CN/docs/Glossary/Protocol)、[端口](https://developer.mozilla.org/zh-CN/docs/Glossary/Port)（如果有指定的话）和[主机](https://developer.mozilla.org/zh-CN/docs/Glossary/Host)都相同的话，则这两个 URL 是_同源_的。这个方案也被称为“协议/主机/端口元组”，或者直接是“元组”。（“元组”是指一组项目构成的整体，具有双重/三重/四重/五重等通用形式。）

下表给出了与 URL `http://store.company.com/dir/page.html` 的源进行对比的示例：

| URL | 结果 | 原因 |
| --- | --- | --- |
| `http://store.company.com/dir2/other.html` | 同源 | 只有路径不同 |
| `http://store.company.com/dir/inner/another.html` | 同源 | 只有路径不同 |
| `https://store.company.com/secure.html` | 失败 | 协议不同 |
| `http://store.company.com:81/dir/etc.html` | 失败 | 端口不同（ |
| `http://news.company.com/dir/other.html` | 失败 | 主机不同 |

### [源的继承](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E6%BA%90%E7%9A%84%E7%BB%A7%E6%89%BF)

在页面中通过 `about:blank` 或 `javascript:` URL 执行的脚本会继承打开该 URL 的文档的源，因为这些类型的 URL 没有包含源服务器的相关信息。

例如，`about:blank` 通常作为父脚本写入内容的新的空白弹出窗口的 URL（例如，通过 [`Window.open()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/open)）。如果此弹出窗口也包含 JavaScript，则该脚本将从创建它的脚本那里继承对应的源。

`data:` URL 将获得一个新的、空的安全上下文。

### [文件源](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E6%96%87%E4%BB%B6%E6%BA%90)

现代浏览器通常将使用 `file:///` 模式加载的文件的来源视为_不透明的来源_。这意味着，假如一个文件包括来自同一文件夹的其他文件，它们不会被认为来自同一来源，并可能引发 [CORS](https://developer.mozilla.org/zh-CN/docs/Glossary/CORS) 错误。

请注意，[URL 规范](https://url.spec.whatwg.org/#origin)指出，文件的来源与实现有关，一些浏览器可能将同一目录或子目录下的文件视为同源文件，尽管这有[安全影响](https://www.mozilla.org/en-US/security/advisories/mfsa2019-21/#CVE-2019-11730)。

## [源的更改](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E6%BA%90%E7%9A%84%E6%9B%B4%E6%94%B9)

**警告：**这里描述的方法（使用 [`document.domain`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/domain) setter）已被弃用，因为它破坏了同源策略所提供的安全保护，并使浏览器中的源模型复杂化，导致互操作性问题和安全漏洞。

满足某些限制条件的情况下，页面是可以修改它的源。脚本可以将 [`document.domain`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/domain) 的值设置为其当前域或其当前域的父域。如果将其设置为其当前域的父域，则这个较短的父域将用于后续源检查。

例如：假设 `http://store.company.com/dir/other.html` 文档中的一个脚本执行以下语句：

这条语句执行之后，页面将会成功地通过与 `http://company.com/dir/page.html` 的同源检测（假设`http://company.com/dir/page.html` 将其 `document.domain` 设置为“`company.com`”，以表明它希望允许这样做——更多有关信息，请参阅 [`document.domain`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/domain)）。然而，`company.com` **不能**设置 `document.domain` 为 `othercompany.com`，因为它不是 `company.com` 的父域。

端口号是由浏览器另行检查的。任何对 `document.domain` 的赋值操作，包括 `document.domain = document.domain` 都会导致端口号被覆盖为 `null` 。因此 `company.com:8080` **不能**仅通过设置 `document.domain = "company.com"` 来与 `company.com` 通信。必须在它们双方中都进行赋值，以确保端口号都为 `null` 。

该机制有一些局限性。如果启用了 [`document-domain`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy) [`Permissions-Policy`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Permissions-Policy)，或该文档在沙箱 [`<iframe>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/iframe) 下，它将抛出一个“`SecurityError`” [`DOMException`](https://developer.mozilla.org/zh-CN/docs/Web/API/DOMException)，并且用这种方法改变源并不影响 Web API 使用的源检查（例如 [`localStorage`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/localStorage)、[`indexedDB`](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API)、[`BroadcastChannel`](https://developer.mozilla.org/zh-CN/docs/Web/API/BroadcastChannel)、[`SharedWorker`](https://developer.mozilla.org/zh-CN/docs/Web/API/SharedWorker)）。更详尽的失败案例列表可以在 [Document.domain 的错误章节](https://developer.mozilla.org/zh-CN/docs/Web/API/Document/domain#%E5%BC%82%E5%B8%B8)找到。

**备注：**使用 `document.domain` 来允许子域安全访问其父域时，需要在父域和子域中设置 `document.domain` 为_相同_的值。这是必要的，即使这样做只是将父域设置回其原始值。不这样做可能会导致权限错误。

## [跨源网络访问](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E8%B7%A8%E6%BA%90%E7%BD%91%E7%BB%9C%E8%AE%BF%E9%97%AE)

同源策略控制不同源之间的交互，例如在使用 [`XMLHttpRequest`](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest) 或 [`<img>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/img) 标签时则会受到同源策略的约束。这些交互通常分为三类：

- 跨源**写操作**（Cross-origin writes）一般是被允许的。例如链接、重定向以及表单提交。特定少数的 HTTP 请求需要添加[预检请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/CORS#%E9%A2%84%E6%A3%80%E8%AF%B7%E6%B1%82)。
- 跨源**资源嵌入**（Cross-origin embedding）一般是被允许的（后面会举例说明）。
- 跨源**读操作**（Cross-origin reads）一般是不被允许的，但常可以通过内嵌资源来巧妙的进行读取访问。例如，你可以读取嵌入图片的高度和宽度，调用内嵌脚本的方法，或[得知内嵌资源的可用性](https://bugzil.la/629094)。

以下是可能嵌入跨源的资源的一些示例：

- 使用 `<script src="…"></script>` 标签嵌入的 JavaScript 脚本。语法错误信息只能被同源脚本中捕捉到。
- 使用 `<link rel="stylesheet" href="…">` 标签嵌入的 CSS。由于 CSS 的松散的语法规则，CSS 的跨源需要一个设置正确的 `Content-Type` 标头。如果样式表是跨源的，且 MIME 类型不正确，资源不以有效的 CSS 结构开始，浏览器会阻止它的加载。
- 通过 [`<img>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/img) 展示的图片。
- 通过 [`<video>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/video) 和 [`<audio>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/audio) 播放的多媒体资源。
- 通过 [`<object>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/object) 和 [`<embed>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/embed) 嵌入的插件。
- 通过 [`@font-face`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@font-face) 引入的字体。一些浏览器允许跨源字体（cross-origin fonts），另一些需要同源字体（same-origin fonts）。
- 通过 [`<iframe>`](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Reference/Elements/iframe) 载入的任何资源。站点可以使用 [`X-Frame-Options`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/X-Frame-Options) 标头来阻止这种形式的跨源交互。

### [如何允许跨源访问](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E5%A6%82%E4%BD%95%E5%85%81%E8%AE%B8%E8%B7%A8%E6%BA%90%E8%AE%BF%E9%97%AE)

可以使用 [CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/CORS) 来允许跨源访问。CORS 是 [HTTP](https://developer.mozilla.org/zh-CN/docs/Glossary/HTTP) 的一部分，它允许服务端来指定哪些主机可以从这个服务端加载资源。

### [如何阻止跨源访问](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E5%A6%82%E4%BD%95%E9%98%BB%E6%AD%A2%E8%B7%A8%E6%BA%90%E8%AE%BF%E9%97%AE)

- 阻止跨源写操作，只要检测请求中的一个不可推测的令牌（CSRF token）即可，这个标记被称为[跨站请求伪造（CSRF）](https://owasp.org/www-community/attacks/csrf)令牌。你必须使用这个令牌来阻止页面的跨源读操作。
- 阻止资源的跨源读取，需要保证该资源是不可嵌入的。阻止嵌入行为是必须的，因为嵌入资源通常向其暴露信息。
- 阻止跨源嵌入，需要确保你的资源不能通过以上列出的可嵌入资源格式使用。浏览器可能不会遵守 `Content-Type` 头部定义的类型。例如，如果你在 HTML 文档中指定 `<script>` 标记，则浏览器将尝试将标签内部的 HTML 解析为 JavaScript。当资源不是网站的入口点时，还可以使用 CSRF 令牌来防止嵌入。

## [跨源脚本 API 访问](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E8%B7%A8%E6%BA%90%E8%84%9A%E6%9C%AC_api_%E8%AE%BF%E9%97%AE)

JavaScript 的 API 中，如 [`iframe.contentWindow`](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLIFrameElement/contentWindow)、 [`window.parent`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/parent)、[`window.open`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/open) 和 [`window.opener`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/opener) 允许文档间直接相互引用。当两个文档的源不同时，这些引用方式将对 [`Window`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window) 和 [`Location`](https://developer.mozilla.org/zh-CN/docs/Web/API/Location) 对象的访问添加限制，如下两节所述。

为了能让不同源中的文档进行交流，可以使用 [`window.postMessage`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/postMessage)。

规范：[HTML 现行标准 & 跨源对象](https://html.spec.whatwg.org/multipage/browsers.html#cross-origin-objects) 。

### [Window](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#window)

允许以下对 `Window` 属性的跨源访问：

| 方法 |
| --- |
| `window.blur` |
| `window.close` |
| `window.focus` |
| `window.postMessage` |

| 属性 |  |
| --- | --- |
| `window.closed` | 只读。 |
| `window.frames` | 只读。 |
| `window.length` | 只读。 |
| `window.location` | 读/写。 |
| `window.opener` | 只读。 |
| `window.parent` | 只读。 |
| `window.self` | 只读。 |
| `window.top` | 只读。 |
| `window.window` | 只读。 |

某些浏览器允许访问除上述外更多的属性。

### [Location](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#location)

允许以下对 `Location` 属性的跨源访问：

| 方法 |
| --- |
| `location.replace` |

| 属性 |  |
| --- | --- |
| `HTMLAnchorElement.href` | 只写。 |

某些浏览器允许访问除上述外更多的属性。

## [跨源数据存储访问](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E8%B7%A8%E6%BA%90%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E8%AE%BF%E9%97%AE)

访问存储在浏览器中的数据，如 [Web Storage](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Storage_API) 和 [IndexedDB](https://developer.mozilla.org/zh-CN/docs/Web/API/IndexedDB_API)，是以源进行分割的。每个源都拥有自己单独的存储空间，一个源中的 JavaScript 脚本不能对属于其他源的数据进行读写操作。

[Cookie](https://developer.mozilla.org/zh-CN/docs/Glossary/Cookie) 使用不同的源定义方式。一个页面可以为本域和其父域设置 cookie，只要是父域不是公共后缀（public suffix）即可。Firefox 和 Chrome 使用 [Public Suffix List](https://publicsuffix.org/) 检测一个域是否是公共后缀。当你设置 cookie 时，你可以使用 `Domain`、`Path`、`Secure` 和 `HttpOnly` 标记来限定可访问性。当你读取 cookie 时，你无法知道它是在哪里被设置的。即使只使用安全的 https 连接，你所看到的任何 cookie 都有可能是使用不安全的连接进行设置的。

## [参见](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E5%8F%82%E8%A7%81)

- [W3C 介绍的同源策略](https://www.w3.org/Security/wiki/Same_Origin_Policy)
- [web.developers.google.cn 介绍的同源策略](https://web.developers.google.cn/articles/same-origin-policy)
- [`Cross-Origin-Resource-Policy`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Cross-Origin-Resource-Policy)
- [`Cross-Origin-Embedder-Policy`](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Reference/Headers/Cross-Origin-Embedder-Policy)

## Help improve MDN

[Learn how to contribute](https://developer.mozilla.org/zh-CN/docs/MDN/Community/Getting_started)

This page was last modified on ⁨2025年5月21日⁩ by [MDN contributors](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy/contributors.txt).

[View this page on GitHub](https://github.com/mdn/translated-content/blob/main/files/zh-cn/web/security/same-origin_policy/index.md?plain=1) • [Report a problem with this content](https://github.com/mdn/translated-content/issues/new?template=page-report-zh-cn.yml&mdn-url=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FSecurity%2FSame-origin_policy&metadata=%3C%21--+Do+not+make+changes+below+this+line+--%3E%0A%3Cdetails%3E%0A%3Csummary%3EPage+report+details%3C%2Fsummary%3E%0A%0A*+Folder%3A+%60zh-cn%2Fweb%2Fsecurity%2Fsame-origin_policy%60%0A*+MDN+URL%3A+https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FSecurity%2FSame-origin_policy%0A*+GitHub+URL%3A+https%3A%2F%2Fgithub.com%2Fmdn%2Ftranslated-content%2Fblob%2Fmain%2Ffiles%2Fzh-cn%2Fweb%2Fsecurity%2Fsame-origin_policy%2Findex.md%0A*+Last+commit%3A+https%3A%2F%2Fgithub.com%2Fmdn%2Ftranslated-content%2Fcommit%2Fed4aedbed53af0c102b497a1096b0d90dbf5ed96%0A*+Document+last+modified%3A+2025-05-21T04%3A56%3A02.000Z%0A%0A%3C%2Fdetails%3E)