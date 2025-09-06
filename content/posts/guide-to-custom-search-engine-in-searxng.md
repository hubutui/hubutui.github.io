---
title: "searxng 自定义搜索引擎配置指南：实战配置 USTC 搜索引擎"
date: 2025-09-06T00:00:00+08:00
draft: false
toc: false
images:
tags:
  - searxng
  - 搜索引擎
---

## 前言

[searxng](https://github.com/searxng/searxng) 是一个以保护隐私为主要特点的自由的元搜索引擎，他不会收集用户的信息，也不会追踪用户的搜索记录．

searxng 已经集成了几百个搜索引擎，同时也也有良好的可扩展性，允许用于添加自定义的搜索引擎．

## 工作原理

从简单角度去理解，searxng 的搜索是这样实现的：

1. 用户输入搜索指令．
2. searxng 模拟用户在浏览器访问搜索引擎网站时的操作，把搜索指令转换为对应的 HTTP GET 或者 POST 请求，发送给对应的搜索引擎．
3. 在发送请求给搜索引擎的时候，searxng 会通过伪造 HTTP 请求头、剔除 cookie 等方法，对搜索引擎隐藏用户的信息．
4. 接收到搜索引擎的响应结果之后，解析搜索请求结果，并展示给用户．

从以上工作原理可以看出，searxng 基本上模拟的本来就是用户在搜索引擎网站的正常操作，只是把作为一个中介，帮用户对搜索引擎隐藏了用户的个人身份信息，避免搜索引擎的追踪，从而保护用户的隐私．

## 自定义的搜索引擎

在 searxng 中，我们可以添加自定义的搜索引擎．关键的地方在于：

1. 如何构造正确的搜索请求．
2. 如何解析搜索引擎返回的结果．

对于第一点，我们一般会通过在浏览器查看网络请求的方式获取到搜索请求的目标地址和请求参数．

对于第二点，如果请求返回的是 HTML 页面，我们可以使用 xpath 来提取内容．当然，前提是 HTML 页面是完整的搜索结果内容，而非返回一个简单 HTML 页面然后使用 JS 动态加载内容的那种．如果返回的是 JSON，则可以解析 JSON 的内容，很方便地从中提取出来我们想要的搜索结果．

### 使用 xpath 类型引擎添加中国科学技术大学官网搜索引擎实战

searxng 提供了一个通用的搜索引擎类型 xpath，让用户只需要修改配置文件，不需要编写代码，就可以添加自己的搜索引擎．

以添加[中国科学技术大学官网](https://ustc.edu.cn)的内容搜索为例，我们应该如何添加自定义的搜索引擎呢？

1. 访问[中国科学技术大学官网](https://ustc.edu.cn)，并执行搜索，假设我们搜索关键词为"毕业典礼"．
2. 可以看到，搜索之后结果页的网址为 `https://ustc.edu.cn/ssjgy.jsp?wbtreeid=1001`，这个看起来没有关键词的信息，没有什么用．
3. 我们点击查看第二页，此时搜索结果页面的地址为 `https://ustc.edu.cn/ssjgy.jsp?wbtreeid=1001&searchScope=0&currentnum=2&sitenewskeycode=5q%2BV5Lia5YW456S8`．
4. 从这个网址可以看出来，`currentnum` 就是页码，并且我们知道页面从 1 开始计数．
5. `5q%2BV5Lia5YW456S8` 做 URL 解码之后得到 `5q+V5Lia5YW456S8`．看不出来是什么东西？没关系，丢给 deepseek 尝试解码，他提示可能是 base64 编码的结果，我们使用 base64 解码然后在用 utf-8 解码，得到了 `毕业典礼`，也就是我们的搜索关键字．
6. 至此，我们就知道了，在中科大官网搜索，实际上就是发送一个 HTTP GET 请求到 `https://ustc.edu.cn/ssjgy.jsp?wbtreeid=1001&searchScope=0&currentnum={pageno}&sitenewskeycode={query}`，其中 `pageno` 为页码，`query` 为先 base64 编码再 URL 编码的搜索关键字．
7. 下一步是确定用于提取搜索结果的 xpath，我们需要确定的有：

  - `results_xpath`: 结果应该是一个列表，列表中的每一个元素为一个搜索结果对应的标签．
  - `title_xpath`：搜索结果的标题，一般要从当前搜索结果开始定位．
  - `url_xpath`: 搜索结果的标题，一般就是找 `a` 标签的 `href` 属性即可，比较简单．
  - `content_xpath`: 搜索结果的内容或者摘要．

8. 在浏览器的开发者工具中，使用左上角的元素审查工具，点击搜索结果里的合适位置，即可在右侧开发者工具的元素标签页查看到这个搜索结果的 HTML 代码，右键复制还能直接给出 xpath，辅助我们进行定位．经过检查，我们很容易发现，这个搜索结果实际上是一个表格．由于浏览器会自动做一些修正，这里拿到的 xpath 可能不准确．例如我们直接复制一个搜索结果所在的 tr 标签的 xpath，得到的结果是 `//*[@id="searchlistform1"]/table[1]/tbody/tr[1]`，但是实际上 `tbody` 是浏览器自己给添加上去的，实际的源码里根本没有．因此，这里建议是把网页源码保存为 HTML 文件，然后通过查看源码的方式来查找合适的 xpath．
9. 从保存好的 HTML 文件中，我们很容易可以得到 `results_xpath` 可以取：`//table[@class='listFrame']//tr[.//span[@class='titlefontstyle250969']]`，`url_xpath` 直接取 `.//a/@href`，`title_xpath` 直接取 `.//td`．
10. `content_xpath` 实际上要取的是搜索结果的兄弟标签，也就是 `following-sibling::tr[1]`．

整理一下，可以得到 ustc 的搜索引擎配置为：

```yaml
- name: ustc
  engine: xpath
  shortcut: ustc
  first_page_num: 1
  paging: true
  search_url: https://ustc.edu.cn/ssjgy.jsp?wbtreeid=1001&searchScope=0&currentnum={pageno}&sitenewskeycode={query}
  results_xpath: //table[@class='listFrame']//tr[.//span[@class='titlefontstyle250969']]
  url_xpath: .//a/@href
  title_xpath: .//td
  content_xpath: following-sibling::tr[1]
  categories: [general]
  about:
    website: https://ustc.edu.cn
    results: HTML
    description: 提供中国科学技术大学官网的内容搜索
```

如果我们直接使用这个配置来添加搜索引擎，然后执行搜索，你会发现它搜索不到我们想要的内容．问题出在关键词的处理上，默认的 xpath 只会做 URL 编码，不会做 base64 编码．因此我们还需要修改 xpath 引擎的源代码[searx/engines/xpath.py](https://github.com/searxng/searxng/blob/master/searx/engines/xpath.py)．这里我们直接复制为 `searx/engines/ustc.py`，然后把 `request` 里的参数 `query` 进行 base64 编码即可，主要修改内容如下：

```python
import base64

def request(query, params):
    # 只需要在一开始对 query 参数进行 base64 编码，其他代码保持不变
    query = base64.b64encode(query.encode('utf-8')).decode('ascii')
    '''Build request parameters (see :ref:`engine request`).'''
    lang = lang_all
    if params['language'] != 'all':
        lang = params['language'][:2]
```

然后重新修改配置，只需要修改 `engine` 为 `ustc`，完整配置如下：

```yaml
- name: ustc
  engine: ustc
  shortcut: ustc
  first_page_num: 1
  paging: true
  search_url: https://ustc.edu.cn/ssjgy.jsp?wbtreeid=1001&searchScope=0&currentnum={pageno}&sitenewskeycode={query}
  results_xpath: //table[@class='listFrame']//tr[.//span[@class='titlefontstyle250969']]
  url_xpath: .//a/@href
  title_xpath: .//td
  content_xpath: following-sibling::tr[1]
  categories: [general]
  about:
    website: https://ustc.edu.cn
    results: HTML
    description: 提供中国科学技术大学官网的内容搜索
```

重新启动 searxng 服务，这次应该就可以正常搜索了．

注意，这里 `searx/engines/ustc.py` 必须要复制 `searx/engines/xpath.py` 的源码，而不是直接导入 xpath 模块来使用．因为 xpath 模块里使用了全局变量来设置如 `search_url` 等参数，如果你直接导入，则 xpath 里的 `request` 函数访问的 `search_url` 等变量依然是 xpath 模块里的全局变量，并不能被我们的配置覆盖，导致出错．

## 总结

本文以增加中国科学技术大学官网的搜索引擎得到 searxng 中的实战记录，详细描述如何给 searxng 添加自定义的搜索引擎．
