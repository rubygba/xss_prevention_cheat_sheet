# XSS（跨站脚本）Prevention Cheat Sheet

本文提供了使用转义/编码输出来积极防止XSS的方法。虽然XSS攻击向量想当庞大，但遵循一些简单的规则就可以完全抵御这种严重的攻击。本文不探讨XSS的技术或业务的影响，我只想说，它可以导致攻击者获得受害者浏览器全部操作和权限。

[reflected XSS 和 stored XSS]() 都可以通过适当的验证和服务器端转码来解决。[基于DOM的XSS]() 可以通过遵循[DOM based XSS Prevention Cheat Sheet]() 这里写到一些规则来规避。

与XSS攻击向量有关的cheatsheet，请参阅[XSS Filter Evasion Cheat Sheet]() 。更多的浏览器的安全性信息和各种浏览器的背景可以在[Browser Security Handbook]() 查阅。

在阅读本文的cheat sheet之前，有必要对注入攻击有一个基础的了解。

## 一种积极的XSS防御模式

本文把一个HTML页面视作一个有很多插槽的模板，开发者在插槽内可以插入不受控的数据。这些插槽覆盖绝大多数常见的插入不受信任数据的地方。在HTML其他地方插入不可信数据是禁止的。这是一个“白名单”模式，即否认未明确允许的一切。

考虑到浏览器解析HTML的方式，每一个不同的插槽类型都有稍微不同的安全规则。当你把不可信数据放到这些插槽里，你需要采取一定的措施来确保数据不会突破插槽、进入上下文、进而成为执行代码。在某种程度上，本文所讲的防御XSS的方法把一个HTML文档视作一个参数数据库查询 - 将数据保存在特定的地方，并通过转义使数据独立于代码上下文。

本文列出了最常见的插槽和安全的将数据插入HTML的规则。基于各种枚举，或者称之为XSS向量，并且在各种流行的浏览器手动测试，我们已经确定这里所写的规则都是是安全的。

每个插槽被明确定义，并且提供了数个示例。开发者**不应该**把数据放到任何其他没有经过分析以确保安全的插槽当中。浏览器解析是非常诡谲的，许多看起来无害的字符在相应的代码上下文里能变得异常重要。


## 为什么直接用字符实体替换不可信数据不能保证安全？

在HTML文档标签内使用字符实体编码不可信数据（如`<div>`标签内，`&lt;`表示`<`）是OK的。如果你严格遵守对标签属性（attribute）添加双引号的习惯，在标签属性内这样嵌入不可信数据也基本OK。但是，如果你在`<script>`标签、或在事件处理属性（如：onmouseover）、或者CSS、又或者在一个URL里这样嵌入不可信数据，并不能解决问题。所以，即使你使用了HTML字符实体编码，你还是极易受到XSS攻击。 **您必须使用转义语法处理不可信数据，再插入HTML文档。** 本文随后所谈的都是这些转义规则。

## 你需要一个安全的编码库

写这些编码是不是非常困难，但也有不少隐藏的陷阱。例如，你可能会使用一些JavaScript中的快速转义格式（如：`\"`）。但是，这些值是危险的，可能被浏览器中的嵌套解析器误解，你也可能忘记对转义字符再次转义，恶意攻击可以借此抵消转义。OWASP（本网站）建议使用专业安全编码库，以确保这些规则的正确实施。

微软提供了一个名为[Microsoft Anti-Cross Site Scripting Library]()反跨站点脚本库为.NET平台和ASP.NET框架内置[ValidateRequest]()方法，可提供**有限的**防范。

OWASP的[OWASP Java Encoder Project]()为Java提供了高性能的编码库。

## XSS防范规则

以下规则旨在所有应用程序中防止XSS。虽然这些规则不允许在HTML文档中绝对自由地插入数据，但它们应该涵盖绝大多数常见的情况。你不必在项目中使用**所有**规则。许多企业发现，**只遵循规则＃1和规则＃2**已经足够满足他们的需求。如果您有额外的常见使用转义的环境请在讨论页面留言。

不要简单地转义各规则提供的示例字符表。仅仅尊享示例列表并不安全。黑名单的方法是相当脆弱的。这里的白名单规则经过精心设计，以提供保护，甚至预计了浏览器未来的漏洞。

### 规则＃0 - 只在一些允许的位置插入不受信数据

首条规则是**拒绝所有** —— 不要把不可信的数据放入你的HTML文档，除非它通过规则＃5插入规则＃1定义的插槽内。之所以有规则＃0，是因为有很多奇怪的HTML上下文环境使得转义非常复杂。我们想不出任何理由把不可信数据插入这些上下文，这包括“嵌套上下文”，就像一个JavaScript内的URL —— 那些位置上的编码规则棘手而且危险。如果硬要将不可信的数据进入嵌套的上下文，请做了很多的跨浏览器测试，并让我们知道你发现什么。

```
<script>...NEVER PUT UNTRUSTED DATA HERE...</script>   directly in a script

<!--...NEVER PUT UNTRUSTED DATA HERE...-->             inside an HTML comment

<div ...NEVER PUT UNTRUSTED DATA HERE...=test />       in an attribute name

<NEVER PUT UNTRUSTED DATA HERE... href="/test" />   in a tag name

<style>...NEVER PUT UNTRUSTED DATA HERE...</style>   directly in CSS
```

最重要的是，不要从不受信任的源接受JavaScript代码并运行它，例如名叫“callback”包含指定JavaScript代码片段的参数。再多的转义都无法弥补这种危害。

### *规则＃1 - 渲染HTML元素前对数据内容进行转义*

规则＃1：当你直接把不可信数据插入HTML标签，像div、p、b、td等常见标签。 大多数web框架都内置用于转义以下字符的方法。然而，对于其他其他HTML上下文，框架提供的方法是绝对不够的。 您需要实现以下详述的其他规则。

```
<body>...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...</body>

<div>...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...</div>

any other normal HTML elements
```

用HTML实体编码转义以下字符，以防止形成到任何执行上下文，如script，style，或绑定事件。建议使用十六进制实体编码规范。除5个在XML有解释意义的字符（＆，<，>，”，“），“/”也包括其中，因为它有助于结束一个HTML实体。

```
& --> &amp;
< --> &lt;
> --> &gt;
" --> &quot;
' --> &#x27;     并不推荐使用 &apos; 因为这不符合HTML规范 (See: section 24.4.1) &apos; 是 XML 和 XHTML 规范
/ --> &#x2F;     forward slash is included as it helps end an HTML entity
```

### *规则＃2 - 属性转义插入不可信数据转换成HTML公共属性之前*

规则＃2是用于使不可信数据到典型的属性值等宽度，姓名，价值等，这不应该被用于类似的href，SRC，风格，或任何事件处理程序等的onmouseover的复杂属性。这是非常重要的事件处理程序的属性应遵循规则＃3 HTML JavaScript的数据值。

```
<div attr=...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...>content</div>     inside UNquoted attribute

<div attr='...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...'>content</div>   inside single quoted attribute

<div attr="...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...">content</div>   inside double quoted attribute
```

除了字母和数字字符，转义所有ASCII值小于256的字符，使用 \xHH 16进制格式。以防止属性成为可执行内容。之所以定制这样一刀切的规则，是因为开发人员经常忘记给属性内容加引号。加了引号的属性只能被引号字符中断形成不可控上下文。而使用不加引号的属性可以被很多字符中断，包括 `[space] % * + , - / ; < = > ^ |`。

### *规则＃3 - JavaScript插入动态内容前转义字符串*

规则＃3涉及动态生成的JavaScript代码 —— 包括script和事件绑定属性。把不可信数据到插入代码的唯一安全方式是用引号包裹数据（如："some data"）。在JavaScript上下文插入不可信数据是相当危险的，因为它是极其容易转换成可执行语句 —— 只要加上（包括但不限于）分号，=，空格，+ 这些字符。所以js插入数据的时候一定要谨慎。

```
<script>alert('...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...')</script>     inside a quoted string

<script>x='...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...'</script>          one side of a quoted expression

<div onmouseover="x='...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...'"</div>  inside quoted event handler
```

请注意，有一些js函数即便传入转义后的字符串也很危险！

例如：

```
<script>
window.setInterval('...EVEN IF YOU ESCAPE UNTRUSTED DATA YOU ARE XSSED HERE...');
</script>
```

除了字母和数字字符，转义所有ASCII值小于256的字符，使用 \xHH 16进制格式。这样能够防止数据转化为可执行语句，进而影响到脚本上下文和其他属性。不要使用 `\"` 这种快速转义，因为引号字符（`"`）可能首先被HTML属性解析器相匹配。这种快速转义很容易被“对转义进行转义”的xss攻击破解，攻击者发送 `\"` 字符，转义后成了 `\\"` 这样引号就会生效。

如果事件处理属性加上了引号，破坏原本的属性需要相应的引号。然而，我们有意地这条规则很宽泛，因为事件处理程序的属性往往是加引号。未引用属性可以与许多字符，包括[空格]％被分解出来的* +， - /; <=> ^和|。此外，</ SCRIPT>关闭标签将关闭脚本块，即使它是一个带引号的字符串中，因为HTML解析器的JavaScript分析器之前运行。

#### *规则＃3.1 - 在HTML中的上下文HTML转义JSON值和读取JSON.parse数据*

在Web 2.0时代，需要在一个JavaScript上下文具有由应用程序动态生成的数据是常见的。一种策略是使一个AJAX调用来获取值，但是这并不总是高性能。通常，JSON的初始块被加载到页以用作一个单一的地方来存储多个值。这个数据是非常棘手，但也不是不可能，要正确转义而不会破坏格式和值的内容。
确保返回的Content-Type头是application / JSON而不是text / html的。 这将指示浏览器不会误解的内容和执行脚本注入
坏的HTTP响应：

```
HTTP/1.1 200
Date: Wed, 06 Feb 2013 10:28:54 GMT
Server: Microsoft-IIS/7.5....
Content-Type: text/html; charset=utf-8 <-- bad
....
Content-Length: 373
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
{"Message":"No HTTP resource was found that matches the request URI 'dev.net.ie/api/pay/.html?HouseNumber=9&AddressLine
=The+Gardens<script>alert(1)</script>&AddressLine2=foxlodge+woods&TownName=Meath'.","MessageDetail":"No type was found
that matches the controller named 'pay'."}   <-- this script will pop!!
```

良好的HTTP响应

```
HTTP/1.1 200
Date: Wed, 06 Feb 2013 10:28:54 GMT
Server: Microsoft-IIS/7.5....
Content-Type: application/json; charset=utf-8 <--good
.....
.....
```

一个常见的反模式一眼就看出：

```
<script>
  var initData = <%= data.to_json %>; // Do NOT do this without encoding the data with one of the techniques listed below.
</script>
```

JSON实体编码
对于JSON编码的规则可以在发现输出编码规则总结。请注意，这不会让你使用由CSP提供1.0 XSS防护。
HTML实体编码
这种技术具有的优点是HTML实体转义广泛支持，并有助于从服务器端代码单独的数据而不穿过任何上下文的边界。考虑将JSON块在页面上作为一个正常的元素，然后解析innerHTML来获取内容。读取跨度中的JavaScript可以住在一个外部文件，从而使CSP执法更容易的实现。

```
<div id="init_data" style="display: none">
  <%= html_escape(data.to_json) %>
</div>
```

```
// external js file
var dataElement = document.getElementById('init_data');
// decode and parse the content of the div
var initData = JSON.parse(dataElement.textContent);
```

到逸出并直接在JavaScript JSON反向​​转义的替代，是通过将其传送给浏览器之前转换“<”到“\ u003c”正常化JSON服务器端。

### 规则＃4 - CSS逃逸并严格验证插入不可信数据转换成HTML样式属性值之前

规则＃4是当你想将不可信的数据进入样式或风格标签。CSS是异常强大的，可用于多种攻击。因此，你只有在一个属性中使用不可信数据是非常重要的价值，而不是为风格数据等场所。你应该把不可信数据到复杂的属性，如URL，行为和自定义（-moz-结合）望而却步。你也应该不会把不可信的数据进入IE的表达式属性值允许JavaScript。

```
<style>selector { property : ...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...; } </style>     property value

<style>selector { property : "...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE..."; } </style>   property value

<span style="property : ...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...">text</span>       property value
```

请注意，是永远不能安全地使用不可信的数据作为输入一些CSS背景- 即使得到妥CSS ESCAPED！你将不得不确保网址只以“http”而不是“JavaScript”和该属性从来没有与“表达”开始启动。

例如：

```
{ background-url : "javascript:alert(1)"; }  // and all other URLs
{ text-size: "expression(alert('XSS'))"; }   // only in IE
```

除了字母数字字符，转义ASCII值小于256与\ HH转义格式的所有字符。不要使用任何解码捷径像\“因为引用字符可能被先运行的HTML属性解析器相匹配。这些逃逸的快捷键也容易受到‘转义的逃逸’，其中攻击者发送\攻击”和有漏洞的代码原来是为\\”，使报价。

如果属性被引用，打破需要相应的报价。所有属性都应该被引用，但你的编码应为强大到足以防止XSS当不可信的数据被放置在不带引号的上下文。未引用属性可以与许多字符，包括[空格]％被分解出来的* +， - /; <=> ^和|。此外，</ style>标记将关闭风格的块，即使它是一个带引号的字符串中，因为HTML解析器的JavaScript分析器之前运行。请注意，我们建议激进的CSS编码和验证，以防止两种报价，未引用属性XSS攻击。

### 规则＃5 - URL逃跑之前插入不可信数据转换成HTML URL参数值

规则＃5是当你想把不可信数据到HTTP GET参数值。

```
<a href="http://www.somesite.com?test=...ESCAPE UNTRUSTED DATA BEFORE PUTTING HERE...">link</a >
```

除了字母数字字符，转义ASCII值的所有字符小于256与HH％逃脱格式。包括不可信数据数据：网址不应该被允许，因为是逃离防止切换出URL的禁止攻击没有什么好办法。所有的属性都应该被引用。未引用属性可以与许多字符，包括[空格]％被分解出来的* +， - /; <=> ^和|。需要注意的是实体编码是在这种情况下无用。

警告：不要完全编码或相对URL与URL编码！如果不受信任的输入是指被放置到HREF，SRC或其他基于URL的属性，它应该进行验证，以确保它不指向一个意外的协议，尤其是Javascript链接。的URL应该然后，可以基于显示的像任何其他块数据的上下文进行编码。例如，用户驱动的URL在HREF链接应该属性编码。例如：

```
String userURL = request.getParameter( "userURL" )
boolean isValidURL = Validator.IsValidURL(userURL, 255);
if (isValidURL) {
  <a href="<%=encoder.encodeForHTMLAttribute(userURL)%>">link</a>
}
```

规则＃6 - 清理HTML标记与专为招聘图书馆
如果您的应用程序处理的标记 - 这应该包含HTML不可信的输入 - 它可以是非常难以验证。编码也是困难的，因为它会破坏所有这些都应该是在输入标签。因此，你需要能够分析一个图书馆和干净的HTML格式的文本。有几个可以在OWASP是简单的使用方法：
HtmlSanitizer - https://github.com/mganss/HtmlSanitizer
一个开源的.Net库。该HTML清洗用白名单的方式。所有允许的标记和属性可以配置。该库单元与所述测试OWASP XSS过滤规避速查表
  VAR =消毒剂新HtmlSanitizer（）;
  sanitizer.AllowedAttributes.Add（ “类”）;
  VAR =消毒sanitizer.Sanitize（HTML）;
OWASP的Java HTML消毒剂 - OWASP的Java HTML消毒剂项目
  进口org.owasp.html.Sanitizers;
  进口org.owasp.html.PolicyFactory;
  PolicyFactory消毒剂= Sanitizers.FORMATTING.and（Sanitizers.BLOCKS）;
  串cleanResults = sanitizer.sanitize（ “<p>您好，<B>世界</ B>！”）;
有关OWASP的Java HTML消毒剂政策建设的更多信息，请参阅https://github.com/OWASP/java-html-sanitizer
Ruby on Rails的SanitizeHelper - http://api.rubyonrails.org/classes/ActionView/Helpers/SanitizeHelper.html
所述SanitizeHelper模块提供了一组用于擦洗不希望的HTML元素的文本的方法。
  <％=消毒@ comment.body，标签：％（重量）（强EM a）中，属性：％（重量）（HREF）％>
提供HTML清理其他库包括：
PHP HTML净化器- http://htmlpurifier.org/
的JavaScript / Node.js的死神- https://github.com/ecto/bleach
Python的死神- https://pypi.python.org/pypi/bleach
规则＃7 - 基于DOM避免XSS
有关基于DOM的XSS是什么细节，以及对防御这种类型的XSS漏洞，请参阅在OWASP文章基于DOM的XSS预防小抄。
奖励规则＃1：使用中HTTPOnly cookie的标志
应用程序中的所有防跨站漏洞是很难的，因为你可以看到。为了帮助减轻的XSS漏洞的网站上的冲击，OWASP还建议您设置中HTTPOnly标志在你的会话cookie并没有被你写任何JavaScript访问您有任何自定义Cookie。这个cookie标志通常是在默认情况下在.NET应用程序，但在其他语言中，你必须手动设置。有关中HTTPOnly cookie的标志更多细节，包括它做什么，以及如何使用它，请参阅OWASP文章中HTTPOnly。
奖励规则＃2：实现内容安全策略
还有另外一个好复杂的解决方案，以减轻称为内容安全策略的XSS漏洞的影响。这是一个浏览器端的机制，它允许您为您的Web应用程序的客户端的资源，例如JavaScript中，CSS，图像等CSP通过特殊的HTTP标头指示浏览器只能从这些来源执行或使资源来源白名单。例如，这CSP
内容安全-政策：默认-SRC： '自我'; 脚本源：“自我” static.domain.tld
将指示Web浏览器只能从页面的起源和JavaScript源代码加载所有资源从static.domain.tld additionaly文件。有关内容安全策略的更多细节，包括它做什么，以及如何使用它，请参阅OWASP文章 Content_Security_Policy
奖励规则＃3：使用自动转义模板系统
许多Web应用程序框架提供了自动的上下文转义功能，如AngularJS严格的语境逃逸并转到模板。当你可以使用这些技术。
奖励规则＃4：使用X-XSS-保护响应头
该HTTP响应头能够内置到一些现代的Web浏览器的跨站点脚本（XSS）过滤器。这头通常是默认启用的，无论如何，所以这头的作用是，如果它被用户禁用重新启用该特定网站的过滤器。
XSS防范规则摘要
以下列出的HTML片段演示了如何安全地在各种不同情境的渲染不可信数据。
数据类型	上下文	代码示例	防御
串	HTML正文	<跨度> 不可信数据 </跨度>
HTML实体编码
串	安全HTML属性	<INPUT TYPE = “文本”名称= “fname”先值= “ 不可信数据 ”>
激进HTML实体编码
唯一的地方不可信的数据进入安全属性（如下所示）的白名单。
严格验证不安全的属性，如背景，编号和名称。
串	GET参数	<a href="/site/search?value= 不可信数据 "> clickme </A>
URL编码
串	在SRC和HREF属性不信任网址	<a href=" UNTRUSTED URL "> clickme </A>
<IFRAME SRC = “ UNTRUSTED URL ”/>
。规范化输入
URL验证
安全网址验证
白名单HTTP和HTTPS URL的唯一（避免的JavaScript协议打开一个新窗口）
属性编码器
串	CSS值	<DIV风格= “宽度：不可信数据 ;”>选择</ DIV>
严格的结构验证
CSS十六进制编码
CSS功能的好设计
串	JavaScript变数	<SCRIPT>变种CurrentValue的= ' 不可信数据 '; </ SCRIPT>
<SCRIPT> someFunction（' 不可信数据 '）; </ SCRIPT>
确保JavaScript的变量引用
JavaScript的十六进制编码
JavaScript的Unicode编码
避免反斜杠编码（\”或\”或\\）
HTML	HTML正文	<DIV> UNTRUSTED HTML </ DIV>
HTML验证（JSoup，AntiSamy，HTML消毒剂）
串	DOM XSS	<SCRIPT>文件撰写（“不可信的输入：” + document.location.hash）; <脚本/>
基于DOM的XSS预防小抄
安全HTML属性包括：对齐，ALINK，ALT，BGCOLOR，边境，CELLPADDING，CELLSPACING，等级，颜色，COLS，合并单元格，COORDS，目录，脸，高度，HSPACE，ISMAP，郎，MARGINHEIGHT，MARGINWIDTH，多个NOHREF，noresize ，noshade，NOWRAP，ref时，REL，REV，行，rowspan的，滚动，形状，跨度，摘要，tabindex属性，标题，USEMAP，VALIGN，值，Vlink时，VSPACE，宽度
输出编码规则摘要
输出编码的目的（因为它涉及跨站脚本）是不可信的输入转换成其中输入被显示为一个安全的形式的数据给用户，而不作为执行代码在浏览器中。下面的图表详细说明了停止跨站点脚本所需的关键输出编码方法的列表。
编码类型	编码机制
HTML实体编码	转换和到＆安培;
转换<至＆lt;
转换>到大于
转换“QUOT＆;
转换”到＆＃X27;
转换/到＆＃X2F;
HTML属性编码	除了字母数字字符，逃避与HTML实体＆＃XHH所有字符; 格式，包括空格。（HH =十六进制值）
URL编码	标准百分号编码，请参阅：http://www.w3schools.com/tags/ref_urlencode.asp。URL编码应该仅被用于编码的参数值，而不是URL的整个URL或路径片段。
的JavaScript编码	除了字母数字字符，逃避与为\ uXXXX的unicode逸出格式（X =整数）的所有字符。
CSS十六进制编码	CSS逃脱支持\ XX和\ XXXXXX。如果接下来的文字继续转义序列使用双字符转义可能会导致问题。有两个解决方案（一）添加CSS后逃逸的空间（将通过CSS解析器忽略）（b）以零填充值，用全额CSS逃脱的可能。