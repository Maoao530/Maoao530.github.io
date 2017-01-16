---
title: 一步一步搭建hexo博客
date: 2016-02-23 15:16:27
categories:
- 其他
tags:
- Hexo
---

Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
<!-- more -->

# 准备环境
因为我是在windows下，所以需要先安装Git和Node.js：
- [Git for windows](https://github.com/git-for-windows/git/releases/download/v2.10.1.windows.1/Git-2.10.1-64-bit.exe)
- [Node.js](https://nodejs.org/dist/v4.6.1/node-v4.6.1-x64.msi)
安装过程就不多说了。

# 安装Hexo
打开git，cd到你需要安装hexo的目录，然后安装hexo

```
npm install -g hexo-cli
```

![安装hexo](/img/archives/npm-install-hexo-cli.png)

# 初始化Hexo项目
创建并且初始化hexo项目
```
hexo init <folder>
cd <folder>
npm install 
```
hexo默认的目录如下：
```
├── .deploy_git
├── public
├── scaffolds
├── scripts
├── source
|   ├── _drafts
|   └── _posts
├── themes
├── _config.yml
└── package.json
```
| 目录        | 描述           |
| ----------- | :------------  |
|.deploy_git  | 执行hexo deploy命令部署到GitHub上的内容目录|
|public	      | 执行hexo generate命令，输出的静态网页内容目录|
|scaffolds	  | layout模板文件目录，其中的md文件可以添加编辑|
|scripts	  | 扩展脚本目录，这里可以自定义一些javascript脚本|
|source	      | 文章源码目录，该目录下的markdown和html文件均会被hexo处理。该页面对应repo的根目录，404文件、favicon，ico文件，CNAME文件等都应该放这里，该目录下可新建页面目录。|
|_drafts	  | 草稿文章|
|_posts	      | 发布文章|
|themes	      | 主题文件目录|
|_config.yml  |	全局配置文件，大多数的设置都在这里|
|package.json |	应用程序数据，指明hexo的版本等信息，类似于一般软件中的关于按钮|


# hexo 命令
常用几个命令：
```
hexo new "postName" #新建文章
hexo new page "pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub
```

# 更换next主题
```
cd your-hexo-site
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
修改站点配置文件_config.yml：
```
theme: next
```
更多关于next主题可以参考：
- [使用文档](http://theme-next.iissnan.com/getting-started.html)
- [next常见问答](https://github.com/iissnan/hexo-theme-next/wiki)

# 安装插件
hexo支持插件，可以直接通过命令行安装即可：
```
npm install plugin-name --save
//更新插件
npm update
//卸载插件
npm uninstall plugin-name
```

下面推荐几个常用的插件：
```
//feed插件
npm install hexo-generator-feed --save
//站点地图
npm install hexo-generator-sitemap --save
//百度站点地图
npm install hexo-generator-baidu-sitemap --save
```

然后在 Hexo 根目录下的 _config.yml 里配置一下：
```
feed:
    type: atom      
    path: atom.xml
    limit: 20  # 最近20篇文章  
sitemap:
    path: sitemap.xml
baidusitemap:
    path: baidusitemap.xml
```

# 博客推广优化

为了博客有更好的展示率, 最好的方式是通过搜索引擎, 下面讲讲怎么让搜索引擎搜录你的博客.

面以百度为例:

- [百度网址提交入口]()

百度站长平台为站长提供单条url提交通道，您可以提交想被百度收录的url，百度搜索引擎会按照标准处理，不保证一定能够收录您提交的url。
建议验证网站所有权后，再提交url。

向百度提交 Sitemap 的过程如下：

> - 注册并登录百度站长平台.
> - 点击 我的网站=>站点管理, 添加你的域名, 类似上文中验证你的域名, 采用 文件验证 上传 html 文件的方式.
> - 验证好以后就可以在 数据提交 里面提交 Sitemap 了.

更详细的可以参考下面这篇文章：

- [franktly.com/2016/07/06/让Baidu和Google收录Hexo博客/](www.franktly.com/2016/07/06/让Baidu和Google收录Hexo博客/)

# 配置上传到github

安装deploy git：
```
npm install hexo-deployer-git --save
```
配置_config.yml：
```
deploy:
  type: git
  repo: git@github.com:Maoao530/Maoao530.github.io.git
  branch: master
```
配置完成后直接`hexo deploy`,就可以了。