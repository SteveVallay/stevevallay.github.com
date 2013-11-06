---
layout: post
title: "使用 Disqus 评论"
date: 2013-11-06 16:59
comments: true
categories: [disqus,comments]
keywords: Disqus, comments
description: add Disqus api to your site
---

想给自己的 [Rails app][1] 添加评论功能，之前在使用 [Octopress][2] 的时候，接触到了 [Disqus][3] 这个专门做 Comments 的平台，许多的 blog 都使用 [Disqus][3] 作为它们的评论插件。所以决定使用 [Disqus][3] 的 api.

<!-- more -->


###帐号

首先，需要现有 [Disqus][1] 的帐号， 没有的话就 [注册][4] 一个吧。


### Add Disqus to your site

登录之后，找到 [Add Disqus to your site][5], 创建一个新的 site profile ,
简单的填写完 site name, admin url, category ，点击  Finish registration， 然后来到
__Choose your platform__ 界面选择你使用的平台，如果没有对应的，就选择 __Universal Code__(通用) 。

然后，Disqus 就为你自动生成了一段代码，将这段代码贴到你的 site 上去，你就拥有了自己的评论系统。

```
    <div id="disqus_thread"></div>
    <script type="text/javascript">
        /* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
        var disqus_shortname = 'yoursitename'; // required: replace example with your forum shortname

        /* * * DON'T EDIT BELOW THIS LINE * * */
        (function() {
            var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
            dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
            (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
        })();
    </script>
    <noscript>Please enable JavaScript to view the <a href="http://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
    <a href="http://disqus.com" class="dsq-brlink">comments powered by <span class="logo-disqus">Disqus</span></a>
```

这段代码主要使用 `js` 来加载评论。如果你觉得还没有满足你的需求，你也可以考虑下[定制化][6]



















[1]:https://github.com/SteveVallay/rshare
[2]:http://octopress.org/
[3]:http://disqus.com/
[4]:https://disqus.com/profile/signup/
[5]:http://disqus.com/admin/create/
[6]:http://help.disqus.com/customer/portal/articles/565624-tightening-your-disqus-integration