---
title: "Hugo 静态博客中迄今为止最佳数学公式支持-利用 MathJax & MMark"
date: 2017-10-09T00:00:00+08:00
markup: mmark
---

## HTML 中的数学公式

要在 HTML 网页中显示数学公式并不是很简单的一件事情，一般最简单的方法自然就是直接用其他程序将写好的数学公式输出成图片，然后在网页中显示．这是最为稳妥，也能完全保证兼容性的办法，但是还是耗时耗力，不讨好．

目前比较流行的方法是交由 [MathJax](http://www.mathjax.com) 来进行渲染，效果和兼容性都还不错，除非你还在用老掉牙的 IE6．

## Markdown 后端

Hugo 默认使用的 markdown 处理后端是 [Blackfriday](https://github.com/russross/blackfriday)，此外也支持使用 [Mmark](https://github.com/miekg/mmark)．后者是从前者 fork 而来的．为了避免麻烦的配置，建议在写包含数学公式的博文时使用 Mmark．只需要在 markdown 文件的 front matter 里加上一行：

```markdown
markup: mmark
```

即可切换到 Mmark．推荐使用 Mmark 的原因只要是无需更多的配置就可以开始写数学公式了，数学公式全部用两个美元符号包围起来，就像这样 `$$ f(x) = \sin(x) $$`，显示的效果为 $$ f(x) = \sin(x) $$．行内公式和行间公式都是一样的，区别在于行间公式前后加上一个空行，这样就可以让 Mmark 自动判断你需要的是行内公式还是行间公式了．行间公式的例子：

```latex

$$
f(x) = \sin(x)
$$

```

显示效果为：

$$
f(x) = \sin(x)
$$

## Hugo 的设置

这个很简单，只需要在 hugo 的主题目录里，加上一行代码到你的博文页面一定会包含的文件里即可．一般可以选择加入到 `themes/paperback/layouts/partials/footer.html` 里，这里的 paperback 是我使用的 hugo 主题名称．需要添加的一行代码为：

```javascript
<script src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.2/MathJax.js?config=TeX-MML-AM_SVG"></script>
```

MathJax 还有很多可以设置的选项，这里不讲，因为我也很少用，有兴趣的可以参考 MathJax 的[配置](http://docs.mathjax.org/en/latest/index.html#mathjax-configuration-options)．

## 数学公式的书写

接下来就可以愉快的在 markdown 中书写数学公式了．


## 参考
* Hugo 的文档 [MathJax with hugo](https://gohugo.io/content-management/formats/#mathjax-with-hugo)
* MathJax 的[文档](http://docs.mathjax.org/en/latest/index.html)
