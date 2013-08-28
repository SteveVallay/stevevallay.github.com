---
layout: post
title: "部署 Rails 程序到 Heroku"
date: 2013-08-27 17:26
comments: true
categories:
- Rails
- Ruby
- Heroku
---


##什么是 Heroku ?

[Heroku][1] 是一个 [Saas][2] (云应用平台)，用户可以将自己的 web 程序部署到 [Heroku][1] 云主机上。使用简单的命令就可以部署你的程序到 [Heroku][1]。 [Heroku][1] 使用 [Git][5] 作为版本控制工具。[Heroku][1] 目前支持 [Ruby][3]， [Node.js][4]，[Clojure][6]，Java，[Python][7]，[Scala][8]。默认数据库是 [PostgreSQL][9]。

>Build. Deploy. Scale. Heroku brings them together
>in an experience built and designed for developers.
> – Larry Marburger, CloudApp

想要部署你的程序到 [Heroku][1]？

Let's Go !

<!-- more -->

##Step 1：创建账户

你需要先创建一个 Heroku的账户，如果你已经有了，直接看 Step 2.

[点此创建 Heroku 账户][10]

##Step 2： 安装 Heroku Toolbelt

`Linux` 用户请查看 [安装 Heroku toolbelt][11] , 或直接执行下面命令：

```
    wget -qO- https://toolbelt.heroku.com/install-ubuntu.sh | sh
```

我在执行的时候发现非常的慢，就在浏览器打开了上面这个[URL][12] 然后，保存下来，手动执行了一下。

我还修改了一行,
将:

```
    apt-get install -y heroku-toolbelt
```
改为:

```
    apt-get install -y --force-yes heroku-toolbelt
```
##Step 3: 登录
安装好之后，就可用命令行登录 heroku:

``` bash
$ heroku login
Enter your Heroku credentials.
Email: adam@example.com
Password:
Could not find an existing public key.
Would you like to generate one? [Yn]
Generating new SSH public key.
Uploading ssh public key /Users/adam/.ssh/id_rsa.pub
```

## Step 4: 准备好自己的程序

一切就绪，你需要准备好自己的程序。

由于 Heroku上 使用的是 PostgreSQL ， 所以我们需要修改下Gemfile。

将:

``` ruby
gem 'sqlite3'
```

修改为:

``` ruby
group :development do
  gem 'sqlite3'
end
group :production do
  gem 'pg'
end
```

确保你的 app 已经纳入 Git 版本控制之中，如果没有请执行:

``` bash
git init
git add -A
git commit -m "initial commit"
```

## Step 5： 部署程序到 Heroku

下面该上传你的 app 到 Heroku 了:

创建 Heroku App:

``` bash
$ heroku create
Creating stark-fog-398... done, stack is cedar
http://stark-fog-398.herokuapp.com/ | git@heroku.com:stark-fog-398.git
Git remote heroku added

-----> Heroku receiving push
-----> Rails app detected
-----> Installing dependencies using Bundler version 1.1
       Checking for unresolved dependencies.
       Unresolved dependencies detected.
       Running: bundle install --without development:test --path vendor/bundle --deployment
       Fetching source index for https://rubygems.org/
       Installing rake (0.8.7)
       ...
       Installing rails (3.0.5)
       Your bundle is complete! It was installed into ./vendor/bundle
-----> Rails plugin injection
       Injecting rails_log_stdout
       Injecting rails3_serve_static_assets
-----> Discovering process types
       Procfile declares types -> (none)
       Default types for Rails -> console, rake, web, worker
-----> Compiled slug size is 8.3MB
-----> Launching... done, v5
       http://severe-mountain-793.herokuapp.com deployed to Heroku

To git@heroku.com:severe-mountain-793.git
 * [new branch]      master -> master
```


[1]:https://www.heroku.com/
[2]:http://en.wikipedia.org/wiki/Platform_as_a_service
[3]:http://www.ruby-lang.org/en/
[4]:http://nodejs.org/
[5]:http://git-scm.com/
[6]:http://clojure.org/
[7]:http://www.python.org/
[8]:http://www.scala-lang.org/
[9]:http://www.postgresql.org/
[10]:https://api.heroku.com/signup/devcenter "创建 Heroku 账户"
[11]:https://toolbelt.heroku.com/debian "install heroku toolbet linux"
[12]:https://toolbelt.heroku.com/install-ubuntu.sh "https://toolbelt.heroku.com/install-ubuntu.sh"