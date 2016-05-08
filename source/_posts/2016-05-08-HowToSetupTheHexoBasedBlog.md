---
title: 怎么用GithubPages+Hexo+NexT搭建博客
date: 2016-05-08 04:15:18
categories:
- Engineering
tags:
- Web
- GitHub
---
我想自己搭博客很久了，之前是不会 Web，等到终于会做网站的时候，已经嫌麻烦不太想弄了，所以转而寻求现成的博客生成、托管方案。
<!-- more -->
考虑到，博客托管和文章备份一定要靠谱，一番考察，还是 [GitHub Pages](https://pages.github.com/) 最省事；而基于 GitHub Pages 的静态网站方案中，对比[Jekyll](https://jekyllrb.com/) 和 [Hexo](https://hexo.io/docs/writing.html) ，后者诞生得晚、活跃的主题/插件多，安装和部署都更简单。在 Hexo 主题中，[NexT](http://theme-next.iissnan.com/tag-plugins.html) ([Demo](http://notes.iissnan.com/))又比绝大部分竞争对手更成熟，于是最终方案就是 GitHub Pages + Hexo + NexT. 


在考察各种教程和主题后，这是我的实施过程：

1. 先看[GitHub Pages](https://pages.github.com/)文档，用它托管一个只包含 index.html 的静态网站。成功后，你能通过`http://username.github.io`访问这个静态博客。
2. 注册域名，添加 [CNAME record](https://en.wikipedia.org/wiki/CNAME_record) 重定向到你刚创建的 GitHub Pages 上的静态博客。另外，GitHub Pages 那边也需要配置一下： [Using a custom domain with GitHub Pages](https://help.github.com/articles/using-a-custom-domain-with-github-pages/)。
3. 了解 [Hexo](https://hexo.io/docs/) 这个静态博客生成器，按照 Doc 在本机创建一个博客模板。此时运行`hexo generate`命令可以根据初始模板生成对应的静态网站，再 push 到前面创建的 GitHub Pages 的 Repo 的话，其实一个基本的博客已经完工。这里有篇很详细的教程：[史上最详细的Hexo博客搭建图文教程](https://xuanwo.org/2015/03/26/hexo-intor/)。
4. 上面的博客还太简单，所以我们要用 Hexo 主题。Github 上 Star 最多的是[NextT](http://theme-next.iissnan.com/getting-started.html)，质量很高，也挺好看 ([Demo](http://notes.iissnan.com/))，改几行配置文件就能完成很多事情：
    + 多说/Disqus 评论
    + 百度/腾讯/Google 统计
    + 微信打赏/公众号


至此全部完工，以后想添加新文章，在 Hexo 的模板 Repo 里`hexo new "post_name"`，再运行`hexo generate`，最后 push 到`https://github.com/username/username.github.io` ，一篇新的博文就发布了。