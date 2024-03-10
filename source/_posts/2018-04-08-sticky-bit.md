---
title: sticky-bit
update: 2018-04-08 20:00:11
tags:
- linux
categories: system
---

在linux的权限中, 发现不仅仅有rwx, 还有一个t, 如下所示:
```bash
drwxrwxr-t   2 shentao shentao       4096  8月 24  2016 test/
```
t只能加在最后一个组, 也就是其他组的权限中, 他的意思是sticky bit, 表示其他组成员不能对该文件进行删除和重命名操作, 这是为了防止某个文件无意被其他组的成员删除和重命名后导致owner找不到文件.

添加的方式是:


```bash
chmod o+t test
```

或者

```bash
chmod +t test
```
或者使用数字的方式, 在首位设置1

```bash
chmod 1775 test
```

结果如下:

```bash
ll test
drwxrwxr-t   2 shentao shentao       4096  8月 24  2016 test/
```

移除方式如下

```bash
chmod o-t test
```

查看结果

```bash
ll test

drwxrwxr-x   2 shentao shentao       4096  8月 24  2016 test/
```

从这里也能看出, t是比x更高一个级别的权限, 即t包含了可执行权限, 只是不能delete和rename. 其次它与rw也不冲突.