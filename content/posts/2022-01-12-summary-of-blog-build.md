---
title: 个人blog构建总结
author: wingstone
date: 2022-01-12
categories:
- 总结
metaAlignment: center
coverMeta: out
---

这篇文章记录个人blog的建立经历，包括主要的技术方案汇总，建立过程中遇到的坑点，以及对应的解决方案；
<!--more-->

## 技术方案

### github pages

使用github pages来托管自己的个人网站，[github pages](https://pages.github.com/)依赖于github建立，可以实现静态网站的搭建，免费，且不限制网站个数，只要建立对应的仓库即可；

### hugo

使用hugo来构建自己的个人网站，[hugo](https://gohugo.io/)作为静态网站生成器，可以非常快速的生产我们设计好的网站；

hugo的安装与使用也非常方便，首先下载可执行文件，然后将可执行文件所在目录添加至环境变量的path中，之后我们就可以在命令行中进行网站的手动构建了；不需要再配置其它的环境，对新人非常友好；

并且提供了大量的免费，供我们来使用和扩充；我们可以下载喜欢的主题，放置到themes文件夹下，然后配置对应toml文件，我们使用我们喜欢的主题，并进行模块的自由配置；

### xmin

使用xmin主题作为基础的hugo主题选择，[xmin](https://github.com/yihui/hugo-xmin)主题代码量非常少，只有大约150行左右，非常适合作为初学者的第一个主题，随后我们对此主题进行我们需要功能的扩充；

xmin基本的功能其实已经足够作为一个简洁的个人博客来使用了；博客中常见的about、markdown、markdown公式、tag、category、个人网站链接、subscribe等功能都提供了建议的支持；只不过使用的过程中有一些坑点，值得细谈；

### github actions

使用github actions来自动化部署我们的网站，[github actions](https://github.com/peaceiris/actions-gh-pages)可以在我们提交源代码工程后触发网站的构建，以及网站的发布；

使用github actions来发布的方法有很多，我是通过开两个仓库来进行部署，网络上有文章分享使用两个分支来进行部署的方案，各个方法都有利弊，使用哪个看个人选择了~；

## 坑点

### mathjax支持的限制

hugo默认是没有添加mathjax的支持的，需要我们自己扩展；幸运的是，扩展mathjax的支持非常容易，xmin主题里提供了layout中foot_custom的扩展，来支持mathjax；

虽然markdown中支持了mathjax，但是仍有一定的限制：

- 行内公式需要使用反单引号括起来，即$xxxx$外要使用``括起来，否则无法生效；
- 多行公式如果有换行的话，不能用\\\\，应该用\\\\\\\\，否则无法生效;

hugo的内置markdownhandler为goldmark，因此goldmark的一些限制，我们只能通过geek来达到，主要是页脚的限制；

- 页脚上标添加中括号，需要对主题的css文件进行更改：

```css
a.footnote-ref::before {
  content: '[';
}

a.footnote-ref::after {
  content: ']';
}
```

- 页脚跳转需要替换符号，需要对主题的css文件进行更改：

```css

.footnote-backref {
  visibility: hidden;
}
.footnote-backref::before {
  content: '^';
  visibility: visible;
}
```

### 代码修改失效

我们调整代码时，经常会遇到修改css文件后，打开重新构建的网站，发现没有变化；原因是浏览器对css进行的缓存，要想生效，需要清除浏览器的css缓存；可以去google应用商店里下载一些插件，来达到清除缓存的目的；

### md中使用图片

markdown中使用图片有诸多限制，一是图片会以原尺寸显示，二是图片会自动左对齐；

- 图片尺寸问题的解决方案为，准备图片素材时，就对分辨率进行设计，就跟ui一样，保证图片的分辨率素材与显示一致；这样反而会避免md对图片缩放的影响；
- 图片自动的左对齐的问题相对简单，因为goldmark默认为居中对齐的，刚好可以满足我们的需要；
