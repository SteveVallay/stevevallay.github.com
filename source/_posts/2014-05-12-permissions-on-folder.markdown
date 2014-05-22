---
layout: post
title: "permissions on folder"
date: 2014-05-12 21:46
comments: true
categories: 
- linux
---

Never foget your dream, never ~ 

<!--more-->

对于一个文件来说，我们都知道，RWX 三个permission 分别对应读、写 和执行权限。但是对于目录来说，它又不能被执行，它的 X 权限有何用呢？如果没有 X 权限，是否可以修改目录中的文件呢？

让我们来实验一下：

```bash
$ mkdir fz
$ ls -ld fz
drwxrwxr-x 2 goodluck goodluck 4096 2014-05-12 21:57 fz
```
创建成功，默认权限775 

添加几个文件进来

```bash
$ cd fz/
$ touch a b c
$ ls -al fz
drwxrwxr-x 2 goodluck goodluck 4096 2014-05-12 21:59 .
drwxr-xr-x 8 goodluck goodluck 4096 2014-05-12 21:59 ..
-rw-rw-r-- 1 goodluck goodluck    0 2014-05-12 21:59 a
-rw-rw-r-- 1 goodluck goodluck    0 2014-05-12 21:59 b
-rw-rw-r-- 1 goodluck goodluck    0 2014-05-12 21:59 c
```
让我们对fz单独设置 RWX 权限来看看是什么效果。

-R

```bash
$ sudo chmod 444 fz #修改 folder 的 permission 需要 root 
$ ls -ld fz
dr--r--r-- 2 goodluck goodluck 4096 2014-05-12 21:59 fz

$ ls -al fz
ls: cannot access fz/.: Permission denied
ls: cannot access fz/..: Permission denied
ls: cannot access fz/a: Permission denied
ls: cannot access fz/c: Permission denied
ls: cannot access fz/b: Permission denied
total 0
d????????? ? ? ? ?                ? .
d????????? ? ? ? ?                ? ..
-????????? ? ? ? ?                ? a
-????????? ? ? ? ?                ? b
-????????? ? ? ? ?                ? c
$ cat fz/a
cat: fz/a: Permission denied
$ cd fz
bash: cd: fz: Permission denied
```

只有 R 权限的话，只能知道 fz 里面有 . .. 两个目录和 a b c 三个文件。文件属性和文件的内容都不能读取，当然也不能写。

-W
```bash
$ sudo chmod 222 fz
$ ls -ld fz
d-w--w--w- 2 goodluck goodluck 4096 2014-05-12 21:59 fz

$ ls fz
ls: cannot open directory fz: Permission denied
$ cat fz/a
cat: fz/a: Permission denied
$ echo "aaa" >fz/a
bash: fz/a: Permission denied
mv fz/a fz/d
mv: accessing `fz/d': Permission denied
$ cd fz
bash: cd: fz: Permission denied
```
只有 W 权限,啥也干不了

-X
```bash
$ sudo chmod 111 fz
$ ls -ld fz
d--x--x--x 2 goodluck goodluck 4096 2014-05-12 21:59 fz

$ ls -al fz
ls: cannot open directory fz: Permission denied
$ echo "aaa" > fz/a  #success
$ cat fz/a
aaa
chmod +x fz/a #success
$ ls -l fz/a
-rw-rw-r-- 1 goodluck goodluck 4 2014-05-12 22:16 fz/a
$ rm fz/a
rm: cannot remove `fz/a': Permission denied
$ cd fz
$ cd fz
$ ls
ls: cannot open directory .: Permission denied
$ touch ff
touch: cannot touch `ff': Permission denied

$ chmod 775 fz/a #fail but not prompt error
$ echo $?
0 #seems success but not changed
-rw-rw-r-- 1 goodluck goodluck 11 2014-05-12 22:26 fz/a
```
只有X权限可以读，写,修改内部文件内容，修改内部文件属性，不能rename/delete/create 文件,chmod 没有报错，却也没有生效


-RX
```
$ sudo chmod 555 fz
$ ls -ld fz
dr-xr-xr-x 2 goodluck goodluck 4096 2014-05-12 22:19 fz
$ ls -al fz
total 12
dr-xr-xr-x 2 goodluck goodluck 4096 2014-05-12 22:19 .
drwxr-xr-x 8 goodluck goodluck 4096 2014-05-12 21:59 ..
-rw-rw-r-- 1 goodluck goodluck   11 2014-05-12 22:26 a
-rw-rw-r-- 1 goodluck goodluck    0 2014-05-12 21:59 b
-rw-rw-r-- 1 goodluck goodluck    0 2014-05-12 21:59 c
```
可以正常读、写、执行文件，cd 到目录，查看文件属性。不能 rename/delete/create 文件

奇怪的一点是在 cd 到 fz 可以使用 chmod 修改 a 文件的permisson, 但是在 fz 外面目录则不生效，奇怪!

-RW
```
$ sudo chmod 666 fz
$ ls -al fz
ls: cannot access fz/.: Permission denied
ls: cannot access fz/..: Permission denied
ls: cannot access fz/a: Permission denied
ls: cannot access fz/c: Permission denied
ls: cannot access fz/b: Permission denied
total 0
d????????? ? ? ? ?                ? .
d????????? ? ? ? ?                ? ..
-????????? ? ? ? ?                ? a
-????????? ? ? ? ?                ? b
-????????? ? ? ? ?                ? c

$ rm fz/a
rm: cannot remove `fz/a': Permission denied
$ touch fz/d
touch: cannot touch `fz/d': Permission denied
```

基本上和只有 R 权限一样，只能获取目录里面文件的文件名，不能读、不能写、不能执行、不能获取文件属性、不能 delete/rename/create 文件。

看得出来，没有 X 的 W 没有什么作用

-WX
```
$ sudo chmod 333 fz
$ ls -ld fz
d-wx-wx-wx 2 goodluck goodluck 4096 2014-05-12 22:19 fz
$ cat fz/a
echo aaa
$ ls -al fz/a
-rwxrwxrwx 1 goodluck goodluck 9 2014-05-12 22:38 fz/a
$ echo "bbb"> fz/b
$ cat fz/b
bbb
$ rm fz/b #success
$ touch fz/e  #success
$ cd fz #success
```

只是不能获取目录中的文件名列表，如果你已经知道某个文件的名字，那么则可以读、写、执行、获取属性、delete/rename/create 都没问题。


总结一下，linux 下，目录的权限属性：

1. R 读取目录中包含的文件名和文件类型
2. W create/rename/delete 目录中的文件(仅在同时有 X 权限的时候生效)
3. X access/search 目录中的文件







