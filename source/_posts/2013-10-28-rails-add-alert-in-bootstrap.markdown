---
layout: post
title: "Rails add alert in bootstrap"
date: 2013-10-28 22:12
comments: true
categories: [Rails,alert,Bootstrap]
keywords: Rails,Bootstrap,Alert
description: add alert in Rails using Bootstrap
---

今天,又给自己的小程序的 Notification 进行了下美化。让 Rails 应用的 Notification 使用 Bootstrap 样式的 [Alert][1] (如下图)：

![alert][101]

<!-- more -->

### Bootstrap 的 `alert` 
首先,在 `Bootstrap` 中 ,只要为你要显示的内容指定 class 为 alert 即可显示成上面的样式（当然不包含 `x` 号)。如下：

``` html
<div class="alert alert-success">...</div>
<div class="alert alert-info">...</div>
<div class="alert alert-warning">...</div>
<div class="alert alert-danger">...</div>
```

`alert-success` `alert-info` `alert-warning` `alert-danger` 会显示为绿色、蓝色、橙色和红色。效果如下：

![alerts][102]

### Rails 中 flash 的 key 映射到 Bootstrap alert 的 class

在 Rails 中，通常是使用 flash 来设置给用户的消息。flash 是一个 map ， 它的 key 常有 :error, :alert ,:notice, :success 用来区分给用户的消息类型。在 view 中遍历 flash 这个 map 即可取到相应的消息。

如：

```ruby
 <% flash.each do |key, msg| -%>
      <%= content_tag :div, msg, class: key %>
    <% end -%>
```

通常将 flash 这个 map 中的 key 设置为 html 元素的 class , 在对这个 class 进行 CSS 的定制化，即可显示处不同样式的 alert。但是 flash 中的 key 值似乎和 Bootstrap  alert 的 class 并不是一一对应，拿来就能用的。我们可以加一个方法来实现这种转化。在 application_helper 里面添加如下代码即可（在 Github gits 中看到的，一时要找不到了 ;-)：

```ruby
  def bootstrap_class_for flash_type
     case flash_type
       when :success
         "alert-success"
       when :error
         "alert-error"
       when :alert
         "alert-block"
       when :notice
         "alert-info"
       else
         flash_type.to_s
     end
   end
```
view 中的代码改成如下即可 :

```
      <%= content_tag :div, msg, class: bootstrap_class_for(key) %>
```

### Bootstrap alert 的 dismiss 

dismiss 的那个 `x` 需要结合 bootstrap 的 javascript 来完成。

在 alert 前面添加如下代码：

```
<button type="button" class="close" data-dismiss="alert">&times;</button>
```

application.js 加入 bootstrap的 js 和 dismiss 功能的 js 

```
//= require bootstrap
...
$(".alert").alert('close')
```

好了 ， OK 鸟 ！ 

效果如图：

![myalert][103]

[1]:http://getbootstrap.com/javascript/#alerts
[101]:/images/blog/alert.png
[102]:/images/blog/alerts.png
[103]:/images/blog/myalert.png