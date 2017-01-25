---
title: Hexo Next博客优化
categories:
  - Hexo
tags:
  - hexo
  - next
date: 2017-01-25 21:30:58
---

进一步打造自己的Blog。

<!-- more -->


# 搜索服务

安装 `hexo-generator-searchdb`

在站点的根目录下执行以下命令：

```
$ npm install hexo-generator-searchdb --save
```

编辑主题配置文件_config.yml，新增以下内容到任意位置：

```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```

# 开启打赏功能 

主题的_config.yml:

```
reward_comment: 坚持原创技术分享，您的支持将鼓励我继续创作！
wechatpay: /img/archives/ss.png
alipay: /img/archives/ss.png
```

# 字体

**全局字体：**编辑 next/source/css/_variables/custom.styl,

```
$font-size-base = 16px; 
```

# 背景图片

next/source/css/_custom/custom.styl文件:

``` 
body { background:url(/images/backGround.jpg);}
```

# 背景颜色

定义颜色变量：themes/next/source/css/_variables/custom.styl

```
// 背景颜色
$body-bg-color = #f5f5d5;
$header-bg-color = #f5f5d5;
$footer-bg-color = #f5f5d5;
```

定义完后，其实body的背景已经换了。

**头部背景色：**

themes/next/source/css/_schemes/Mist/sidebar/_header.styl：

```
.header { background: $header-bg-color; }
```

**footer背景色：**

```
.footer {
  margin-top: 80px;
  padding: 10px 0;
  background: $footer-bg-color;
  color: $grey-dim;
}
```

# 代码风格

主题_config.yml:

```
highlight_theme: night blue
```

# 内容宽度

themes/next/source/css/_variables/custom.styl

```
// 修改成你期望的宽度
$content-desktop = 900px
// 当视窗超过 1600px 后的宽度
$content-desktop-large = 1100px
```

# 文章目录序号关闭

主题_config.yml:

```
#Table Of Contents in the Sidebar
toc:
  enable: true

  # Automatically add list number to toc.
  number: false
```

# 无序列表

不喜欢空心的，我们换成实心的列表：

文章列表：
source/css/_common/components/post/post-expand.styl:
```
  ul li { list-style: disc; }
```

页面列表：
next/source/css/_custom/custom.styl:

```
ul {
list-style-type: disc;  // 空心圆，实心圆为 disc
}
```

参考：[https://github.com/iissnan/hexo-theme-next/issues/559](https://github.com/iissnan/hexo-theme-next/issues/559)

# 文章访问次数

主题_config.yml:

```
busuanzi_count:
  # count values only if the other configs are false
  enable: true
  # custom uv span for the whole site
  site_uv: true
  site_uv_header: <i class="fa fa-user"></i>访问人数
  site_uv_footer:
  # custom pv span for the whole site
  site_pv: true
  site_pv_header: <i class="fa fa-eye"></i>总访问量
  site_pv_footer: 次
  # custom pv span for one page only
  page_pv: true
  page_pv_header: <i class="fa fa-file-o"></i>浏览
  page_pv_footer: 次
```

# 表格样式

这个背景色下面表格边界不太明显，我们修改为：

next/source/css/_variables/base.styl

```
// Table
// --------------------------------------------------
$table-width                    = 100%
$table-border-color             = $grey-dark
$table-font-size                = 14px
$table-content-alignment        = left
$table-content-vertical         = middle
$table-th-font-weight           = 700
$table-cell-padding             = 8px
$table-cell-border-right-color  = $grey-dark
$table-cell-border-bottom-color = $grey-dark
$table-row-odd-bg-color         = #f9f9f9
$table-row-hover-bg-color       = $whitesmoke
```
