---
title: xss原理与检测
date: 2023-07-03 19:58:57
tags:
---





# 前置知识

详细了解xss的原理和流程需要首先了解浏览器的编码和解码过程。

主要为url编码、html编码和js编码。

## 浏览器的编码和解码

### url编码

标准的url结构是:scheme://login:password@address:port/path?quesry_string#fragment

当用户的输入作为url的一部分(如get请求)时，为了使用户输入不破坏语法，部分字符需要进行url编码。这部分url保留字符为 `:/?#[]@=+$,;"%{}`

上述保留字符所有的浏览器都遵守 但在测试时发现不同浏览器的url保留字符存在细微差别，例如版本号为108.0.5359.124的chrome对url中的圆括号`()`编码为`%28%29`。版本号为114.0.5735.199的chrome则不进行编码。

### html编码

我们拿常见的标签举例。

```bash
<p>Hello World</p>
```

跟url的问题类似，如果用户输入将被嵌入到 HTML 页面中，例如在表单字段、文本区域或注释中，浏览器会对其进行 HTML 编码。HTML 编码将特殊字符转换为对应的实体引用或实体编号，以避免与 HTML 标记发生冲突。于是就有了html编码，一般以一般以&开头，以分号 **;** 结尾。

html编码的字符包括`'"<>&`被编码为`&#39;&quot;&lt;&gt;&amp;` 。

### JS编码

同理，如果用户输入将被用作 JavaScript 的一部分，例如在脚本中使用用户输入的数据，浏览器会对其进行 JavaScript 编码。JavaScript 编码将特殊字符转换为对应的转义序列，以确保代码的正确执行。

## 浏览器的解析过程

浏览器的解析如下：

![image-20230703202738206](../images/xss原理与检测/image-20230703202738206.png)

解析顺序：

1. 解析 HTML 结构：浏览器首先从网络接收到 HTML 文件，然后按照自上而下的顺序解析 HTML 结构。解析过程包括识别标签、构建 DOM 树（文档对象模型）、解析 CSS 样式等。
2. 加载外部资源：在解析 HTML 结构的过程中，如果遇到外部资源（如 CSS 文件、JavaScript 文件、图片等），浏览器会启动额外的网络请求来加载这些资源。外部资源的加载可能是并行进行的，以提高页面加载的效率。
3. 解析 CSS 样式：当浏览器解析到 `<style>`标签或外部 CSS 文件时，会开始解析和应用 CSS 样式规则。解析 CSS 样式是一个逐级继承和计算的过程，最终确定每个元素的最终样式。
4. 构建渲染树：在解析 HTML 结构和 CSS 样式后，浏览器会将它们结合起来，构建渲染树（Render Tree）。渲染树是由可视化元素组成的树状结构，每个元素都包含了最终应用的样式信息。
5. 布局（Layout）：在构建渲染树后，浏览器会计算每个元素在页面上的准确位置和大小，即进行布局操作。这个过程也被称为回流（Reflow）。
6. 绘制（Paint）：在布局完成后，浏览器会将渲染树的每个元素转化为屏幕上的实际像素，进行绘制操作。这个过程也被称为重绘（Repaint）。
7. 合成（Composite）：最后，浏览器将绘制好的元素按正确的顺序合成在一起，并显示在屏幕上，形成最终的页面呈现。

我们主要关注前面三步，浏览器首先解析html，进行html解码。根据dom节点构造dom树。在解析的过程中，遇到`<script>`或内联事件等js相关的代码时，会使用js解析器对其进行解析，此时会进行js解码。如果浏览器遇到需要URL的上下文环境，这时URL解析器也会介入完成URL的解码工作，URL解析器的解码顺序会根据URL所在位置不同，可能在JavaScript解析器之前或之后解析。



# XSS

## XSS分类

XSS主要分为反射型、存储型和DOM型。但是这三个类型的分类标准并不一致。XSS形成的原因是对用户的输入没有进行严格校验，导致在客户端的渲染过程中触发了额外的JavaScript代码执行。

**反射型XSS：**用户的输入没有存入服务器，只是在输入并点击后单次触发。

**存储型XSS：**用户的输入存入了服务器，其他用户访问该页面时也会触发，导致持久化攻击。

**DOM XSS：** **DOM XSS** 没有进入服务器解析，只和前端有关。



## XSS触发位置

下图是XSS触发攻击结构图

<img src="https://www.hacking8.com/books/sectips/assets/image-20190903204716157.png" alt="image-20190903204716157" style="zoom:67%;" />

- contents中

  - 普通tag的contents
  - script tag中的contents

- attribute中

  - 普通属性
  - on属性
  - src属性
  - href属性

- 注释中

  

以下是一个用来分析html节点tag、属性和data的脚本。

```python
class ContextAnalyser(HTMLParser):
    def handle_starttag(self, tag, attrs):
        print("Encountered a start tag:", tag)
        for attr in attrs:
            print("    Attribute:", attr[0], "Value:", attr[1])

    def handle_data(self, data):
        print("        data:",data)

# 创建解析器实例并解析HTML文档
parser = ContextAnalyser()
# parser.feed('<div id="myDiv" class="container">Hello, World!</div>')
html = requests.request("get","www.target.com")
parser.feed(html.text)
```



## XSS触发

触发分为几种

- 利用或构造`<script></script>` 并在其中执行payload
- 利用或构造部分attribute构造伪协议



因此XSS触发，就是看后端是否对我们构造这几种payload做出限制。

那么对几种位置的注入点依次判断：

- 普通tag的contents

  普通tag的contents是指非`<script>`等tag中的contents，例如

  ```html
  <h2 align=center>injection</h2>
  ```

  对于这类注入点，要实现xss注入的话必须闭合该tag，要闭合tag必须使用到尖括号`<>`，因此`<>`不能被转义。

- 可执行tag中的contents

  可执行tag的contents是指`<script>` 等tag中的contents，例如：

  ```html
   <script>injection</script>
  ```

  这类注入点相当于是用户直接控制js代码。不管如何过滤都不能保证用户输入完全安全。

  





