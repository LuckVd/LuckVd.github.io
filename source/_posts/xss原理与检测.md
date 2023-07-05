---
title: xss原理与检测
date: 2023-07-03 19:58:57
categories: 
- 安全基础
tags:
- XSS
cover: ../images/xss原理与检测/XSS.png
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
  - script 等tag中的contents

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



因此XSS触发，就是看后端是否对我们构造这几种payload做出限制。我们以[xsslab](https://github.com/LuckVd/xsslab)靶场中的case为例子。

对几种位置的注入点依次判断：

- 普通tag的contents

  普通tag的contents是指非`<script>`等tag中的contents，对于这类注入点，要实现xss注入的话必须闭合该tag，要闭合tag必须使用到尖括号`<>`，因此`<>`不能被转义。

  例如靶场xss/1.php和xss/2.php

  ```html
  //xss/1.php 无过滤
  <h2 align=center>".$str."</h2>;
  ```

  ```html
  //xss/2.php 尖括号过滤
  $str = str_replace("<", "&lt;", $str);
  $str = str_replace(">", "&gt;", $str);
  echo "<h2 align=center>".$str."</h2>";
  ```

  注入payload：`<script>alert(123)</script>`

  

- 可执行tag中无包裹的contents

  可执行tag的contents是指`<script>` 等tag中的contents，靶场xss/3.php和xss/4.php：

  ```html
  //xss/3.php 无过滤
  echo "<script>".$str."</script>";
  ```

  ```html
  //xss/4.php htmlspecialchars过滤
  echo "<script>".htmlspecialchars($str)."</script>";
  ```

  这类注入点相当于是用户直接控制js代码。不管如何过滤都不能保证用户输入完全安全。

  注入payload：`alert(123)`

  

- 普通属性

  在attribute中，可以利用伪协议，因此不需要构造`<script>`。但是普通属性无法利用伪协议，因此需要闭合该属性。如果能够闭合该属性则可以触发xss。例如靶场xss/5.php和xss/6.php，前者没有过滤可以利用，后者对双引号进行了过滤。

  ```html
  // xss/3.php无过滤
  <input name=keyword  value="'.$str.'">
  ```

  ```html
  //xss/4.php 双引号过滤
  $str = str_replace("\"", "&quot;", $str);
  <input name=keyword  value="'.$str.'">
  ```

  注入payload：`" onmouseover="alert(123)`

  同样的，靶场xss/7.php和xss/8.php则是单引号包含属性。

  

- on、src、href等属性

  如果用户可以控制这类属性，则一般的过滤无法防止XSS。

  例如靶场9-14，分别是三类属性未过滤的例子和使用`htmlspecialchars`过滤的例子，这些都能够触发xss。

  

- 可执行tag中有包裹的contents

  和xss/3.php、xss/4.php中不同的是。如果用户的输入被引号包裹，那么需要绕过引号才能触发XSS。要么闭合引号，要么闭合尖括号。

  xss/15.php是无过滤的情况，xss/16.php是引号过滤，xss/17.php是引号、尖括号过滤。

  ```html
  xss/15.php
  $str = $_GET["keyword"];
  echo "<script>\"".$str."\";</script>";
  ```

  因此这个14，15的payload如下：

  ```html
  //引号闭合
  ";alert(1);"
  //script标签闭合
  </script><iframe src='javascript:alert(1)'>click</iframe><script>
  ```

  ## 总结

   经过上述的例子，就可以不同触发点的XSS对应的过滤条件也不同，`htmlspecialchars()`的 作用是把预定义字符转换成html实体，防止在DOM树构建时被用户输入所影响，但是如果用户的输入在可执行的tag和可以构造javascript伪协议的点时，行为发生在DOM树构造之后，因此这种过滤也就不起效果。总得来说，XSS的防护便可分为两部分:

  - 防止用户对html结构造成修改。

  - 防止用户对可执行的点进行修改。

    

  # XSS防护

  对于XSS的防护，需要从上章提到的两部分着手。首先使用htmlspecialchars()对用户的输入进行转义。防止用户的输入对html结构造成更改。然后对上文提到的特定的点做特定的防护。例如对于href检测输入时候符合url格式。

  

  

  

  

  





