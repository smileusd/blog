---
title: 忽略掉一些奇怪的goroutine信息
tags:
  - golang
categories:
  - coding
date: 2018-03-19 18:55:19
---


在跑自己的golang程序时遇到死锁, 查看stack, 发现一些奇怪的goroutinue:

```
goroutine 17 [syscall, 12 minutes, locked to thread]:
runtime.goexit()
        /usr/local/go/src/runtime/asm_amd64.s:2086 +0x1
        
goroutine 20 [select, 12 minutes, locked to thread]:
runtime.gopark(0xd067b8, 0x0, 0xc738c8, 0x6, 0x18, 0x2)
        /usr/local/go/src/runtime/proc.go:259 +0xfd
runtime.selectgoImpl(0xc420022f30, 0xc420022f1c, 0x0)
        /usr/local/go/src/runtime/select.go:423 +0x1303
runtime.selectgo(0xc420022f30)
        /usr/local/go/src/runtime/select.go:238 +0x1c
runtime.ensureSigM.func1()
        /usr/local/go/src/runtime/signal1_unix.go:304 +0x28c
runtime.goexit()
        /usr/local/go/src/runtime/asm_amd64.s:2086 +0x1
```

这些goroutine显示: `locked to thread`, 但上网查找有回复说到, 这并不是一个真正的死锁, 而是正常的goroutine拿线程锁的过程. 可能调用了cgo或者runtime.LockOSThread. 这个问题可以忽略掉, 以防绕进死胡同.
```
That is not a deadlock, at least not a deadlock in the Go program.  It 
is normal for a goroutine to be locked to a thread.  It can happen 
because of cgo, or because the code called runtime.LockOSThread.  It 
is not a problem.  The problem is that the thread has not made any 
progress for 214 minutes; what that is depends on what it is doing. 
```

