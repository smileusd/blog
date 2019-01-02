---
title: "git出现fatal: Pathspec 'xxx' is in submodule 'xxx'的解决方法"
update: 2018-03-21 12:21:03
tags: 
- git
categories:
- coding
---

在hexo中, 在themes目录下git clone了一个子目录themes/next. git push 到远程仓库后没有看到themes/next中的内容, 也无法git add
```shell
shentao@shentao-ThinkPad-T450:~/blog$ git commit -m "fix" themes/next/
On branch master
Changes not staged for commit:
        modified:   themes/next (modified content)
shentao@shentao-ThinkPad-T450:~/blog$ git add themes/next/*
fatal: Pathspec 'themes/next/bower.json' is in submodule 'themes/next'
```
当创建一个子git项目是, 项目就叫做git submodule结构, 外部无法对子模块进行控制, 有两种方式(我知道的):
1.
```shell
cd themes/next
git add .
git commit
```
再到外部提交
``` shell
git add themes/next
git commit
git push
```

2.
```shell
git rm --cached themes/next  
git add themes/next
```



