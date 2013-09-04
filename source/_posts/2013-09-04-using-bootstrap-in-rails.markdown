---
layout: post
title: "using bootstrap in rails"
date: 2013-09-04 12:41
comments: true
categories:
- bootstrap
- rails
---


本人 `rails` 新手，使用 `rails 4.0` ， 在 `rails` `app ` 中添加 [Bootstrap][1] 支持, 参考 `Ruby on Rails tutorial 2nd edition En`。

<!-- more -->

##Step 1  添加 [bootstrap-sass][2] 到 __Gemfile__

打开 **Gemfile** 添加如下内容：

``` ruby
# Use bootstrap
  gem 'bootstrap-sass','~>2.3.2' #tutorial 上用的是2.0.0
```

## Step 2 安装

直接运行：

``` ruby
bundle install
```

## Step 3 创建 `custom.css.scss`

创建 `custom.css.scss` 文件：

__app/assets/stylesheets/custom.css.scss__

在 **app/assets/stylesheets** 下的文件会自动被 **application.css** `include`进来。

在这个文件中可以添加 bootstrap CSS 进来，在 `custom.css.scss` 中添加：

``` css
@import "bootstrap";
```

当然，还可以在`custom.css.sass`文件中添加自己的`custom` 如：

``` css
@import "bootstrap";

/*universal*/

html {
        overflow-y:scroll;
}

body {
        padding-top: 10px;
        padding-left: 10px;
}
```
[1]:http://getbootstrap.com/ 'home page'
[2]:https://github.com/thomas-mcdonald/bootstrap-sass 'github'