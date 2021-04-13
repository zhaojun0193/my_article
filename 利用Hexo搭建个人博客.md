---
title: 利用Hexo搭建个人博客
date: 2017-12-21 23:17:49
tags: 随笔
toc: true
---

### 前言
想自己弄个博客做一下学习记录,以及分享学习心得,了解到github的pages功能,然后选择了Hexo,期间遇到了一些问题.写此博客记录一下.  
文章包含以下内容:  
1. 安装NodeJs  
2. 安装hexo  
3. 初始化  
4. 生成静态页面  
5. 本地启动  
6. 更换主题  
7. 部署到github
8. 更改头像
9. 新建文章

### 安装NodeJs

访问[NodeJs官网下载页面](http://nodejs.cn/download/)选择合适的版本进行下载.  

### 安装hexo
    $ npm install -g hexo
然后输入命令`hexo -v` 若输出hexo的版本号即为安装成功.

### 初始化
在某个目录下新建一个项目文件夹(比如blog),然后进入blog目录,以下所有的命令操作也都是在这个目录下进行的.
  
    hexo init
### 生成静态页面
初始化完成之后,就可以生成hexo默认的hello world文章了.现在执行以下命令把文章编译为静态页面.

    hexo generate

### 本地启动
把文章编译为静态页面后,需要执行以下命令启动本地的服务,在浏览器中输入[http://localhost:4000/](http://localhost:4000/)查看页面生成的效果.

    hexo server

显示的效果如果是这样的话那么就成功了.

![效果图](http://oucja4p5v.bkt.clouddn.com/home.png)

<!-- more -->
`注:如果启动后访问不了,有可能是端口被占用,查找到是哪个进程占用了4000端口,kill掉就好了.`

### 更换主题
上面的博客效果是hexo自己默认的主题 `landscape` ,我选择的是 `yilia` 这个主题.  

在目录下执行clone命令:

    git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia


修改 `blog/_config.yml` 文件：

    theme: yilia    //默认为landscape

修改 `themes/yilia/_config.yml` 文件：

    # Header

    menu:
      主页: /
      随笔: /tags/随笔/
    
    # SubNav
    subnav:
      github: "https://github.com/zhaojun0193"
      weibo: "https://weibo.com/u/3884931222"
      rss: "#"
      zhihu: "#"
      #qq: "#"
      #weixin: "#"
      #jianshu: "#"
      #douban: "#"
      #segmentfault: "#"
      #bilibili: "#"
      #acfun: "#"
      #mail: "mailto:litten225@qq.com"
      #facebook: "#"
      #google: "#"
      #twitter: "#"
      #linkedin: "#"
    
    rss: /atom.xml
    
    # 是否需要修改 root 路径
    # 如果您的网站存放在子目录中，例如 http://yoursite.com/blog，
    # 请将您的 url 设为 http://yoursite.com/blog 并把 root 设为 /blog/。
    root: /
    
    # Content
    
    # 文章太长，截断按钮文字
    excerpt_link: more
    # 文章卡片右下角常驻链接，不需要请设置为false
    show_all_link: '展开全文'
    # 数学公式
    mathjax: false
    # 是否在新窗口打开链接
    open_in_new: false
    
    # 打赏
    # 打赏type设定：0-关闭打赏； 1-文章对应的md文件里有reward:true属性，才有打赏； 2-所有文章均有打赏
    reward_type: 2
    # 打赏wording
    reward_wording: '谢谢你请我吃糖果'
    # 支付宝二维码图片地址，跟你设置头像的方式一样。比如：/assets/img/alipay.jpg
    alipay: /img/alipay.jpg
    # 微信二维码图片地址
    weixin: /img/wechatpay.jpg
    
    # 目录
    # 目录设定：0-不显示目录； 1-文章对应的md文件里有toc:true属性，才有目录； 2-所有文章均显示目录
    toc: 1
    # 根据自己的习惯来设置，如果你的目录标题习惯有标号，置为true即可隐藏hexo重复的序号；否则置为false
    toc_hide_index: true
    # 目录为空时的提示
    toc_empty_wording: '目录，不存在的…'
    
    # 是否有快速回到顶部的按钮
    top: true
    
    # Miscellaneous
    baidu_analytics: ''
    google_analytics: ''
    favicon: /img/favicon.ico
    
    #你的头像url
    avatar: /img/my.jpg
    
    #是否开启分享
    share_jia: true
    
    #评论：1、多说；2、网易云跟帖；3、畅言；4、Disqus；5、Gitment
    #不需要使用某项，直接设置值为false，或注释掉
    #具体请参考wiki：https://github.com/litten/hexo-theme-yilia/wiki/
    
    #1、多说
    duoshuo: false
    
    #2、网易云跟帖
    wangyiyun: false
    
    #3、畅言
    changyan_appid: false
    changyan_conf: false
    
    #4、Disqus 在hexo根目录的config里也有disqus_shortname字段，优先使用yilia的
    disqus: false
    
    #5、Gitment
    gitment_owner: false      #你的 GitHub ID
    gitment_repo: ''          #存储评论的 repo
    gitment_oauth:
      client_id: ''           #client ID
      client_secret: ''       #client secret
    
    # 样式定制 - 一般不需要修改，除非有很强的定制欲望…
    style:
      # 头像上面的背景颜色
      header: '#4d4d4d'
      # 右滑板块背景
      slider: 'linear-gradient(200deg,#a0cfe4,#e8c37e)'
    
    # slider的设置
    slider:
      # 是否默认展开tags板块
      showTags: false
    
    # 智能菜单
    # 如不需要，将该对应项置为false
    # 比如
    #smart_menu:
    #  friends: false
    smart_menu:
      innerArchive: '所有文章'
      friends: '友链'
      aboutme: '关于我'
    
    friends:
      CSDN: https://www.csdn.net/
      博客园: https://www.cnblogs.com/
    
    aboutme: 很惭愧<br><br>只做了一点微小的工作<br>谢谢大家

### 部署到github

部署之前先修改 `blog/_config.yml` 文件

    deploy:
      type: git
      repository: https://github.com/zhaojun0193/zhaojun0193.github.com.git
      branch: master
`注：在hexo3.x版本下，这里的type应该填git，不是github；另外冒号后面都有一个英文的空格，不然会报错的。`

然后使用以下命令进行部署

    hexo deploy
`注：如果执行上述命令报错，你可以试试下面这个命令再试`

    npm install hexo-deployer-git--save

部署成功后输入[https://zhaojun0193.github.io](https://zhaojun0193.github.io)进行访问
    
### 更改头像

在 `\blog\themes\yilia\source\img` 下放入你的头像图片(如:head.jpg),然后修改 `themes/yilia/_config.yml` 文件`:

    #你的头像url
    avatar: /img/my.jpg

### 新建文章  

文章都保存在 `\blog\source\_posts`目录下  
可以使用一下命令来新建文章

    hexo new '文章标题'


然后就是利用markdown编辑器修改文章,然后发布就好啦~


* * *
本文地址是：[利用Hexo搭建个人博客](https://zhaojun0193.github.io/2017/12/21/%E5%88%A9%E7%94%A8Hexo%E6%90%AD%E5%BB%BA%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/) 转载请注明原创地址。
  
参考博文:   
[http://visugar.com/2017/05/04/20170504SetUpHexoBlog/](http://visugar.com/2017/05/04/20170504SetUpHexoBlog/)  
[http://blog.csdn.net/scythe666/article/details/51956821](http://blog.csdn.net/scythe666/article/details/51956821)



