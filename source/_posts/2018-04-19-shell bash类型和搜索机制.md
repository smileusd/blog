---
title: shell bash 类型和搜索机制
update: 2018-04-08 20:00:11
tags:
- linux
categories:
- system
---

### 问题描述

今天在执行一个命令是遇到一个which路径与实际执行不同步的问题, 如下:

```bash
shentao@shentao-ThinkPad-T450:~/blog$ which ss-local
/usr/local/bin/ss-local
shentao@shentao-ThinkPad-T450:~/blog$ /usr/local/bin/ss-local -v | grep shadowsocks-libev
shadowsocks-libev 2.5.6 with OpenSSL 1.0.1f 6 Jan 2014
shentao@shentao-ThinkPad-T450:~/blog$ ss-local -v | grep shadowsocks-libev
shadowsocks-libev 3.1.3
shentao@shentao-ThinkPad-T450:~/blog$ /usr/bin/ss-local -v | grep shadowsocks-libev
shadowsocks-libev 3.1.3

```

which 现实路径是/usr/local/bin/ss-local, 但实际执行的路径却是/usr/bin/ss-local. 查看which的文档, 发现which是按顺序搜索PATH环境变量来查找的:

```bash
shentao@shentao-ThinkPad-T450:~/blog$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/go/bin:/usr/local/go/bin:/home/shentao/gopath//bin:/opt/idea-IC-145.1617.8/bin:/usr/local/go/bin:/home/shentao/gopath//bin:/opt/idea-IC-145.1617.8/bin:/usr/local/go/bin
```

所以最先在/usr/local/bin中找到.

### 问题解决

于是去调查了下bash的执行过程和顺序. 得到了答案. https://crashingdaily.wordpress.com/2008/04/21/hashing-the-executables-a-look-at-hash-and-type/

原来在每一次执行过程中shell都会有一个hash table作为cache将调用过的可执行命令存入hash table, 这样再下次调用的时候就可以直接从缓存中读取, 而不用每次去搜索PATH. 我们通过type命令可以查看:

```bash
shentao@shentao-ThinkPad-T450:~/blog$ type ss-local
ss-local is hashed (/usr/bin/ss-local)
```

可以通过hash命令删除:

```bash
shentao@shentao-ThinkPad-T450:~/blog$ hash -d ss-local
shentao@shentao-ThinkPad-T450:~/blog$ ss-local -v | grep shadowsocks-libev
shadowsocks-libev 2.5.6 with OpenSSL 1.0.1f 6 Jan 2014
```

### 知识回顾

我们再来回顾下关于bash 的一些知识:

#### 关于bash可以执行的类型

- Aliases: An alias is a word that is mapped to a certain string, 命令的别名.
```bash
$ alias nmapp="nmap -Pn -A --osscan-limit"
$ nmapp 192.168.0.1
```

- Functions:  A function contains shell commands, and acts very much like a small script. 用function定义一个函数:

  ```bash
  function gcc { echo “just a test for gcc”; }
  ```

- Builtins: 内置函数, cd之类的

- Keywords: Keywords are like builtins, with the main difference being that special parsing rules apply to them. For example, [ is a Bash builtin, while [[ is a Bash keyword. They are both used for testing stuff. [[ is a keyword rather than a builtin and is therefore able to offer an extended test: 
```bash
$ [ a < b ]
-bash: b: No such file or directory
$ [[ a < b ]]
```
第一个< 是重定向, 第二个加了[[的关键词之后 < 成了小于号.

- Executables: (Executables may also be called *external commands* or *applications*.) Executables are commonly invoked by typing only their name. This can be done because a pre-defined variable makes known to Bash a list of common, executable, file paths. This variable is called `PATH`. 外部命令, 也就是PATH中找到的那些命令
- Script: 脚本. 比如test.sh

#### 关于bash的搜索顺序总结

bash搜索的顺序是: 当前路径和绝对路径的目录->alias->keyword->function->built-in->Executables, Script(hash)->Executables, Script($PATH)

#### 相关命令

bash, hash, type, which, alias, function