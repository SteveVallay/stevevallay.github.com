---
layout: post
title: "learn rails i18n"
date: 2013-11-05 17:56
comments: true
categories: [Rails，i18n]
keywords: Rails,Ruby, i18n
description: Learning i18n of Rails
---

今天，学习了一下 Rails 的国际化，为 app 添加了中文翻译，非常简单。

<!--more -->

参考这个 [Railscasts 视频][1] 和 [Rails Guide][2].下面结合添加 中文翻译，简单介绍一下。

每个平台都包含自己国际化的方式，比如 Android 平台，在 res 文件夹下， 会包含各种语言的字符串，放在特定名字的文件夹下，如 string, string-zh, string-en, string-fr 等等，应用开发者只要在 string-zh 下添加所需要的中文翻译，系统在中文环境下会自动加载 string-zh 下的字符串。

Rails 也有类似的机制，在 config/locales 下，默认只有 en.yml， 可以在这里添加多国语言的字符串来让 Rails app 支持多国语言版本。


###添加中文翻译文件

首先在 config/locales 下 copy en.yml 成  zh.yml ，然后在 zh.yml 中添加字符 id 和 对应的字符内容。

如下：

```
zh:
  user:
    name: 用户名
    email: 邮箱
    password: 密码
```

如果内容比较多，你可以分成不同的 namespace , 上面例子中，user 就是 zh 下面的一个 namespace， 你可以创建更多的层次， 引用的时候以 `.` 来引用就好了， 比如 user.name。



### 在你的 code 中引用字符串 id 

支持多国的话，就要把 code 中 引用的字符串，改为字符 id 来引用。I18n 的 api  translate (t) 可以根据字符 id 来找到对应的字符串。

将直接引用字符串，改为使用 api 函数 t 来查找对应语言的字符串:

```ruby
<h2><%= Create New User %></h2>
```

改为:

```ruby
<h2><%= t('user.new_user_reg') %></h2>

```


### 程序中配置你要使用的语言

application_controller.rb 中添加 set_language 方法：

```ruby
before_action :set_language
     ...
     def set_language
      I18n.locale = :zh
     end
```













[1]:http://railscasts.com/episodes/138-i18n?autoplay=true
[2]:http://guides.rubyonrails.org/i18n.html