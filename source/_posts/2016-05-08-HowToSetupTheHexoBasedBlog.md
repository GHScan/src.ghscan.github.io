---
title: 讲下怎么用Hexo+NexT搭这个博客
date: 2016-05-08 04:15:18
categories:
- Engineering
tags:
- Web
- GitHub
---
我自己想搭博客很久了，之前是不会 Web 编程；后来终于学会做网站，但仍然嫌麻烦不想自己做博客。
<!-- more -->
考虑到，文章的托管和备份一定要靠谱，一番考察，还是觉得用 [GitHub Pages](https://pages.github.com/) 最省事；而基于 GitHub Pages 的静态网站方案中，对比[Jekyll](https://jekyllrb.com/) 和 [Hexo](https://hexo.io/docs/writing.html) ，后者诞生得晚、活跃的主题多，安装和部署都更简单。在 Hexo 的主题中，[NexT](http://theme-next.iissnan.com/tag-plugins.html) ([Demo](http://notes.iissnan.com/))又比所有的竞争对手的成熟度都要高一个档次，于是最终方案就是 GitHub Pages + Hexo + NexT. 


我自己看了多少乱七八糟的教程和主题按下不表，这是推荐路线：

1. 先看[GitHub Pages](https://pages.github.com/)的文档，至少用它托管一个空的 index.html  。成功过后，你就能通过`http://username.github.io`来访问这个只有空的 index.html 页面的静态网站了。
2.  注册一个域名，添加 CNAME record 重定向到你刚生成的由 GitHub 托管的网站。另外在 Repo 中也需要做点事： [Using a custom domain with GitHub Pages](https://help.github.com/articles/using-a-custom-domain-with-github-pages/)。
3.  在[Hexo](https://hexo.io/docs/)了解一下这个静态博客生成器，然后按照 Doc 在本机初始化一个博客。此时如果你运行 hexo 命令，在这个初始博客模板上生成静态网站，push 到前面创建的 GitHub Pages 的 Repo 的话，那其实一个简陋的博客已经完工了，以后每次发新文章，直接`hexo new "post_name"`就可以了。这篇教程写得很好：[史上最详细的Hexo博客搭建图文教程](https://xuanwo.org/2015/03/26/hexo-intor/)。
4.  上面生成的博客太简陋，所以我们需要用 Hexo 的模板。Github 上 Star 最多的是[NextT](http://theme-next.iissnan.com/getting-started.html)，确实很专业，外观够炫 ([Demo](http://notes.iissnan.com/))，功能也很丰富，改几行配置文件就能完成很多事情：
    + 多说/Disqus 评论
    + 百度/腾讯/Google 统计
    + 微信打赏
    + 微信公众号


到这里就全部完工了，以后每次添加新文章，在 Hexo 的模板 Repo 里面`hexo new "post_name"`，然后运行`hexo generate`得到静态网站，再 push 到`https://github.com/username/username.github.io` (或者`hexo deploy`)，博客内容就更新了。