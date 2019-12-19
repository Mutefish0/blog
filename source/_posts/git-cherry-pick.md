---
title: 再谈git-cherry-pick
date: 2019-12-19 18:08:56
tags:
---

## cherry-pick一个普通的commit

`git cherry-pick <c1>`可以把`c1`这个commit的改动应用到当前分支来，并产生一个新的commit。具体做法：1.先计算c1的父节点`c1.parent`与`c1`的diff，2.把diff应用到当前分支并产生一个新的commit 3.移动当前分支指针到新产生的commit

看下面例子（我本地给2个git命令起了别名：`alias ci=commit`，`alias co=co`，以下所有例子均使用别名）：
```bash
mkdir trygit
cd trygit
git init

touch init.txt
git add .
git ci -m 'init'
ls # init.txt

git co -b c1
touch c1.txt
git add . 
git ci -m 'c1'
ls # c1.txt init.txt

git co master -b c2
touch c2.txt
git add . 
git ci -m 'c2'
ls # c2.txt init.txt

git co c2
```
图示如下
```
+------+          +--+
|master| <------- |c1|
+------+          +--+
   ^                  
   |              +--+
   +------------- |c2| <--- HEAD
                  +--+
```
然后进行cherry-pick 
```bash
git co -b c3
git cherry-pick c1
ls # c1.txt c2.txt init.txt
```
图示如下
```
+------+          +--+          
|master| <------- |c1|           
+------+          +--+          
   ^                             
   |              +--+      +--+
   +------------- |c2| <--- |c3| <--- HEAD
                  +--+      +--+
```
`c1`的父节点是`master`，diff结果是新增一个文件c1.txt。然后应用到`c2`，产生新的commit `c3`

## cherry-pick一个merge commit
看下面的例子
```bash
mkdir trygit
cd trygit
git init

touch init.txt
git add .
git ci -m 'init'
ls # init.txt

git co -b c1
touch c1.txt
git add . 
git ci -m 'c1'
ls # c1.txt init.txt

git co master -b c2
touch c2.txt
git add . 
git ci -m 'c2'
ls # c2.txt init.txt

git co master -b c3
touch c3.txt
git add . 
git ci -m 'c3'
ls # c3.txt init.txt

git co c2 -b c4
git merge c1
ls # c1.txt c2.txt init.txt

git co c3
```
图示如下
```
                  +--+                                 
   +------------- |c1| <-----+                         
   |              +--+       |                         
   v                         |                         
+------+          +--+      +--+                       
|master| <------- |c2| <--- |c4|                       
+------+          +--+      +--+                       
   ^                                                   
   |              +--+                                 
   +------------- |c3| <--- HEAD                          
                  +--+ 
```
现在我们想把c4 cheery-pick到`c3`，从图中我们可以看到`c4`有两个父节点，
如果直接像这样`git cherry-pick c4`，会直接报错
这时需要指定一个参数，来告诉git，我们想要选择哪个父节点来做diff，像这样：
`$ git cherry-pick c4 -m <parent number>`
`parent number`是从1开始的数字，表明所选择的父节点
我们可以看一下c4的commit对象
`$ git cat-file -p c4`
显示如下
```
tree 323e489dd752302902e6e805d4b6ef56b950da76
parent 9ae6a421ae63908d3bd649bb25bc4bd79b3614a0
parent 6eff213dc985084b87d3abc99e60319773aee6ab
author Mutefish0 <648262030@qq.com> 1576756313 +0800
committer Mutefish0 <648262030@qq.com> 1576756313 +0800
```
我们看parent字段，从上到下
`parent number`为1则对应`9ae6a421ae6`
`parent number`为2则对应`6eff213dc98`
我们可以分别看一下git log
`$ git log 9ae6a421ae6`
`$ git log 6eff213dc98`
可以发现`9ae6a421ae6`就是`c2`，`6eff213dc98`就是`c1`

所以结论是，如果使用`-m 1`，则以`c2`为父节点做diff，`-m 2`则以`c1`为父节点来做diff

### -m 1
我们试一下选择`c2`为父节点
```
git co -b c5
git cherry-pick c4 -m 1
ls # c1.txt   c3.txt   init.txt
```
图示如下
```
   +------------- |c1| <-----+                         
   |              +--+       |                         
   v                         |                         
+------+          +--+      +--+                       
|master| <------- |c2| <--- |c4|            
+------+          +--+      +--+                       
   ^                                                   
   |              +--+      +--+                       
   +------------- |c3| <--- |c5| <--- HEAD  c5 = c3 + (c4 - c2)
                  +--+      +--+   
```
`c4`与`c2`做diff，结果是新增文件c1.txt，和结果一致

### -m 2
我们再试一下以`c1`为父节点
```
git co c3 -b c6
git cherry-pick c4 -m 2
ls # c2.txt   c3.txt   init.txt
```
图示如下
```
                  +--+                                 
   +------------- |c1| <-----+                         
   |              +--+       |                         
   v                         |                         
+------+          +--+      +--+                       
|master| <------- |c2| <--- |c4|                       
+------+          +--+      +--+                       
   ^                                                   
   |              +--+      +--+                       
   +------------- |c3| <--- |c5|  c5 = c3 + (c4 - c2)                     
                  +--+      +--+                       
                    ^                                  
                    |       +--+                       
                    +------ |c6|  <--- HEAD  c6 = c3 + (c4 - c1)                    
                            +--+   
```
`c4`与`c1`做diff，结果是新增文件c2.txt，和结果一致

## parent number可以大于2吗？
答案是可以的。这样意味着，一个节点可以有2个以上的父节点。什么情况下会出现这种情况？
之前一直没注意，`git merge`是后面是可以跟多个commit/branch的，这样产生的commit就有2个以上的父节点
我们动手试验一下：
```
mkdir trygit
cd trygit
git init

touch init.txt
git add .
git ci -m 'init'
ls # init.txt

git co -b c1
touch c1.txt
git add . 
git ci -m 'c1'
ls # c1.txt init.txt

git co master -b c2
touch c2.txt
git add . 
git ci -m 'c2'
ls # c2.txt init.txt

git co master -b c3
touch c3.txt
git add . 
git ci -m 'c3'
```
图示如下:
```
                  +--+                                 
   +------------- |c1|                                 
   |              +--+                                 
   v                                                   
+------+          +--+                                 
|master| <------- |c2|                                 
+------+          +--+                                 
   ^                                                   
   |              +--+                                 
   +------------- |c3| <--- HEAD                              
                  +--+   
```
我们把`c1`、`c2`都合到`c3`：
```
git co -b c4
git merge c1 c2
ls # c1.txt   c2.txt   c3.txt   init.txt
```
图示如下:
```
                  +--+                                 
   +------------- |c1| <----+                          
   |              +--+      |                          
   v                        |                          
+------+          +--+      |                          
|master| <------- |c2| <----+                          
+------+          +--+      |                          
   ^                        |                          
   |              +--+     +--+                        
   +------------- |c3| <-- |c4|  <--- HEAD  
                  +--+     +--+   
```
我们看一下`c4`的commit对象的内容
`$ git cat-file c4`
```
tree b0432c46b02d7097bd11759efc109b4856ab3928
parent f90dedc08aafd53b8a287f89621757f89930124e
parent ba229d2f4a45601d8d39e2a01c86181fed13ef7c
parent 32bc711cb25b4c2eac269369ef686b1a93a14b3c
author Mutefish0 <648262030@qq.com> 1576760401 +0800
committer Mutefish0 <648262030@qq.com> 1576760401 +0800
```
确实有三个父节点，顺序为：`c3`、`c1`、`c2`
可以总结出，parent number为1则对应被merge入的commit，然后根据merge后的参数顺序，依次递增

我们来检验一下
### -m 3
parent number传3，则对应的父节点是`c2`
```
git co master -b c5
git cherry-pick c4 -m 3
ls #  c1.txt c3.txt init.txt
```
图示如下:
```
                  +--+                                 
   +------------- |c1| <----+                          
   |              +--+      |                          
   v                        |                          
+------+          +--+      |                          
|master| <------- |c2| <----+                          
+------+          +--+      |                          
 ^ ^                        |                          
 | |              +--+     +--+                        
 | +------------- |c3| <-- |c4|                        
 |                +--+     +--+                                             
 |                                   +--+                  
 +---------------------------------- |c5| <--- HEAD  c5 = master + (c4 - c2)              
                                     +--+  
```
`c4`与`c2`的diff结果是新增两个文件c1.txt、c3.txt，与结果相符