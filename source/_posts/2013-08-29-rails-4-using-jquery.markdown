---
layout: post
title: "rails 4 using jquery"
date: 2013-08-29 12:40
comments: true
categories:
- rails
- jquery
- javascript
- ruby
---


本人 rails 新手，在学习 `rails` (version 4.0.0） 程序的时候，需要用到 `javascript`，想使用现成的 `java script` 库，就想着把 [JQuery][1] 添加进来，并且要使用一个 `jquery` 的 `plugin ` -- [jquery validation][2]。

<!-- more -->
## rails 怎么加入 jQuery ?

[Google][3] 了很久，有很多 `rails 3` 如何添加使用 `jQuery` 的， `rails4` 的就没找到。后来，终于发现原来__自 `rails 3.1` 之后，`rails` 已经包含了对 `jQuery`的支持__。:-) [jquery-rails][4] 这个 `gem` 就是 `rails` 对 `jQuery` 的支持， 查看 Gemfile , 如果已经包含了 [jquery-rails][4] , 而且 `rails` 版本等于或高于 `rails3.1` 那就说明已经支持 jQuery 了。

具体请看 [jquery-rails 的 github][4]。

查看 `Gemfile` :

如果有下面这一行，那么 `rails new app` 的时候会自动用 `bundle install` 安装 `jquery-rails`。

    gem "jquery-rails"

在 `app/assets/javascripts/application.js` 里面应该有下面这两句，这样 `application.js` 就自动包含了 `jquery.js` 和 `jquery_ujs.js` :

    //= require jquery
    //= require jquery_ujs


## 怎么测试我的 `rails` 已经包含了 `jQuery` ?

有个简单的方法可以测试你的 rails 是否已经包含了对 jQuery的支持，首先，你要确保上面所说的 `Gemfile` 和 `application.js` 都如上所示。

新建一个 `rails app` (如果已经有了就不用新建了),新建个 `welcome#index`

``` ruby
    $ rails new my-app
    $ rails generate controller welcome index
```

启动你的 `rails app` ：

```
    cd my-app
    rails s
```

打开 <http://localhost:3000/welcome/index>, 右击页面，查看源代码,如果有了下面两行，应该是 `Ok` 了。

```html
    <script data-turbolinks-track="true" src="/assets/jquery.js?body=1"></script>
    <script data-turbolinks-track="true" src="/assets/jquery_ujs.js?body=1"></script>
```


## 添加一段 `js` 代码测试 `jQuery`

当然，我们也可以加一段 `js` 代码来测试 `jQuery` 是否可以被正确的调用。

在 `app/assets/javascripts/application.js` 文件末尾添加如下代码:

``` javascript
$(document).ready(function(){
        alert("success!");
});
```

你再次打开 <http://localhost:3000/welcome/index> 的时候，会弹出一个  `success!` 的 `dialog`，那就说明成功了!

## 添加 `jQuery` `Validation ` `plugin`

下载 [jQuery Validation][5]，把 `jquery.validate.min.js` 放在 `app/assets/javascripts` 下面就可以用了。

使用方法可以参考 [这个tutorials][6]，或者[官方文档][7]。



[1]:http://jquery.com/
[2]:http://jqueryvalidation.org/
[3]:http://google.com
[4]:https://github.com/rails/jquery-rails
[5]:http://jqueryvalidation.org/
[6]:http://sleekd.com/tutorials/jquery-validation-in-ruby-on-rails/
[7]:http://jqueryvalidation.org/documentation