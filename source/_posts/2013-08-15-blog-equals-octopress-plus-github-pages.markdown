---
layout: post
title: "blog = octopress + github pages"
date: 2013-08-15 10:44
comments: true
categories:
- github pages
- octopress
- blog
keywords: blog,octopress,github,pages
description: blog=octopress + github pages
---

Thanks to github , we have another choice to setup a blog beyond the popluar wordpress, [github pages][1] +  [octopress][2] (based on [jekyll][3], simply we can say : octopress is jekyll + themes).

Using octopress we can setup a blog very easy , the most import is that the blog is host at [github][4]:

*   it is free!
*   version control !
*   without bandwith limit !
*   without storage space limit!
*   keep as long as you wish !

*Let's go ...*
<!--more-->
### Step1: setup your project on [github][4].

first , you need a [github][4] account , if you do not have one , please apply one on [github][4]

second ,you need create a new repository  with your name , assume your github account name is **yourname** , you need create your project with the name **yourname.github.io** or **yourname.github.com** .


### Step2: [Setup Octopress][5]

assume you are working on uinx or linux, you alredy have [git][6] installed.

then , [install ruby with rvm][7] or [install ruby with Rbenv][8] , use  rvm for example :

run the following command from terminal to get rvm:
``` bash
    $ curl -L https://get.rvm.io | bash -s stable --ruby
```
install latest ruby (1.9.3 currently):
```ruby
    $ rvm install 1.9.3 
    $ rvm use 1.9.3 
    $ rvm rubygems latest
```
get octopress from github:
```bash
    $ git clone git://github.com/imathis/octopress.git octopress
    $ cd octopress    # If you use RVM, You'll be asked if you trust the .rvmrc file (say yes)
    $ ruby --version  # Should report Ruby 1.9.3
```
install dependencies:
``` ruby
    $ gem install bundler
    $ bundle install
```
install the default theme:
``` ruby
    $ rake install
```
### Step3: [deploy to github pages][9]

```ruby
    $ rake setup_github_pages #input your repo name as git@github.com://yourname/yourname.github.io(com).git
```
this command will do a couple things for you:

* Ask you for your github pages repository url.
* Rename the remote pointing to imathis/octopress from ‘origin’ to ‘octopress’
* Add your Github Pages repository as the default origin remote.
* Switch the active branch from master to source.
* Configure your blog’s url according to your repository. 
* Set up a master branch for your project in the _deploy directory, ready for deployment.

Next run:
``` ruby
    $rake generate
    $rake deploy
```
this will generate the content of your blog under _deploy directory and push it to master(gh-pages) branch ,
a few minites later, you can view your blog at **yourname.github.io(com)**

### Step4: [Configure Octopress][10]

open _config.yml, change the config as you like:
```
    url: http://stevevallay.github.io
    title: Zhibin's blog
    subtitle: alway's smile :-)
    author: zhibin
    ...
```

### Step5: [Blogging][11]
``` ruby
    $ rake new_post['blog title']
```

then, it will generate a file *source/_post/YYYY-MM-DD-XXXX.markdown* , this is the blog source , you can edit it with [markdown][12] , if you havn't familiar with *markdown* , refer [markdown wiki][14] or  [markdown in Chinese][13]

### Step6: Generate & Preview

after finish one post , you can preview it:

``` ruby
    $ rake generate  #Generates posts and pages into the public directory
    $ rake watch      # Watches source/ and sass/ for changes and regenerates
    $ rake preview    # Watches, and mounts a webserver at http://localhost:4000
```

open browser [http://localhost:4000](http://localhost:4000) you can see your blog.

*Do not* forget two things :

*  Commit sources of your blog to github , from your local source branch to source branch in github.
``` bash
    $ git add .
    $ git commit -m "blog = github pages + octopress"
    $ git push origin source
```
*  Deploy your blog, this will generate all files under_deploy and commit it to github master(gh-pages) branch.

``` ruby
    $ rake deploy
```

Finally , you will get a similar blog as current [stevevallay.github.io][15]

### [Octopress Plugin][16]

the first octopress plugin i recommanded is [backtick code block][17] , it can help 
add line number and syntax hightlight, octopress default installed it under plugin directory.
Simple you can use it as following:

**Syntax**

    ``` [language] [title] [url] [line text]
        code snipt
    ```

**Example**

    ``` bash
        $ sudo make me a sandwich
    ```

``` bash
    $ sudo make me a sandwich
```






[1]:http://page.github.com
[2]:http://octopress.org
[3]:http://jekyllrb.com
[4]:http://github.com
[5]:http://octopress.org/docs/setup
[6]:http://git-scm.com
[7]:http://octopress.org/docs/setup/rvm
[8]:http://octopress.org/docs/setup/rbenv
[9]:http://octopress.org/docs/deploying/githug
[10]:http://octopress.org/docs/configuring
[11]:http://octopress.org/odcs/blogging
[12]:http://http://daringfireball.net/projects/markdown/
[13]:http://wowubuntu.com/markdown
[14]:http://en.wikipedia.org/wiki/Markdown
[15]:http://stevevallay.github.io
[16]:http://http://octopress.org/docs/plugins/
[17]:http://octopress.org/docs/plugins/backtick-codeblock/
