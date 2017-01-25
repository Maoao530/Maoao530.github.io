---
title: 一步一步搭建hexo博客
date: 2016-02-23 15:16:27
categories:
- Hexo
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

# SEO

## 增加主页关键词

更改index.swig文件，文件路径是your-hexo-site\themes\next\layout，将下面代码：

```
{% block title %} {{ config.title }} {% endblock %}
```

改成

```
{% block title %} {{ config.title }} - {{ theme.description }} {% endblock %}
```

## 百度收录

（1）网站验证

先到百度站长注册账号，并且添加网站，然后进行网站验证，我们通过html标签验证，next主题已经给我们做好了：

打开_config.yml:
```
baidu_site_verification: *****
```

然后hexo deploy后，点击完成验证即可。

（2）提交baidusitemap.xml

打开百度站长工具，网页抓取-链接提交栏下，选择自动提交：

- 主动推送 最为快速的提交方式，将站点当天新产出链接立即通过此方式推送给百度，以保证新链接可以及时被百度收录。需要改写url链接带参数，操作起来不好搞，放弃。
- 自动推送 需要嵌入一段js代码，当我们每访问一次网站，则自动运行脚本，推送链接到百度 （next主题也为我们做好了）
- sitemap：自动提交baidusitemap.xml

（3）自动推送

打开_config.yml：
```
baidu_push: true
```

打开hexo-blog/themes/next/layout/_scripts/baidu-push.swig，替换为百度给我们的js代码：
```javascript
{% if theme.baidu_push %}
<script>
(function(){
    var bp = document.createElement('script');
    var curProtocol = window.location.protocol.split(':')[0];
    if (curProtocol === 'https') {
        bp.src = 'https://zz.bdstatic.com/linksubmit/push.js';        
    }
    else {
        bp.src = 'http://push.zhanzhang.baidu.com/push.js';
    }
    var s = document.getElementsByTagName("script")[0];
    s.parentNode.insertBefore(bp, s);
})();
</script>
{% endif %}
```


## 谷歌收录

（1）网站验证

验证方式基本一样，打开_config.yml:

```
google_site_verification: *******
```

（2）提交sitemap

点击站点地图，提交sitemap.xml即可，比较简单。


# 同时使用coding和github

补充一下，因为百度对github page不太友好，如果博客托管在github page上，很难被百度收录。
所以我们可以把网站托管到coding平台，即同时使用Coding提供的Pages服务和Github提供的Pages服务

安装deploy git：

```
npm install hexo-deployer-git --save
```

配置_config.yml：

```
deploy:
  type: git
  repo: 
      git@github.com:Maoao530/Maoao530.github.io.git
      coding: https://git.coding.net/xxxxxx,coding-pages
  branch: master
```

配置完成后直接`hexo deploy`,就可以了。

# 国内国外访问不同的Pages服务

如果托管到coding pages后，那么域名当然也和github pages不一样了，那么怎么办呢？

去dnspod买个域名，我们可以通过DnsPod让国内用户访问coding，让国外用户访问github。然后我们只要访问这个域名即可。





