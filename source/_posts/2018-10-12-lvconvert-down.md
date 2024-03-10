---
title: lvconvert-down
update: 2018-10-12 14:42:45
tags: linux
categories: system
---

记录一个kernel bug.

在组建中调用了lvconvert生成存储池, 但是测试告诉我过程失败并且机器重启了, 通过抓kdump生成以下日志:

```shell
[  983.179047] BUG kmalloc-256(4:099fef32f6df55082271a9ab572acec8251aa754a14a60444d7ce4cb110be56f) (Tainted: G    B          ------------ T): Objects remaining in kmalloc-256(4:099fef32f6df55082271a9ab572acec8251aa754a14a60444d7ce4cb110be56f
[  983.179890] -----------------------------------------------------------------------------

[  983.180762] INFO: Slab 0xffffea000cae8840 objects=16 used=9 fp=0xffff88032ba21b00 flags=0x2fffff00000080
[  983.181181] CPU: 3 PID: 25070 Comm: lvconvert Tainted: G    B          ------------ T 3.10.0-327.el7.x86_64 #1
[  983.181182] Hardware name: Red Hat KVM, BIOS 0.5.1 01/01/2011
[  983.181183]  ffffea000cae8840 00000000c255adbf ffff88018d43baa8 ffffffff816351f1
[  983.181186]  ffff88018d43bb80 ffffffff811beac4 0000000000000020 ffff88018d43bb90
[  983.181188]  ffff88018d43bb40 656a624f00000000 616d657220737463 6e6920676e696e69
[  983.181190] Call Trace:
[  983.181195]  [<ffffffff816351f1>] dump_stack+0x19/0x1b
[  983.181203]  [<ffffffff811beac4>] slab_err+0xb4/0xe0
[  983.181206]  [<ffffffff811c1f33>] ? __kmalloc+0x1f3/0x230
[  983.181208]  [<ffffffff811c316b>] ? kmem_cache_close+0x12b/0x2f0
[  983.181210]  [<ffffffff811c318c>] kmem_cache_close+0x14c/0x2f0
[  983.181212]  [<ffffffff811c35c4>] __kmem_cache_shutdown+0x14/0x80
[  983.181215]  [<ffffffff8118ceaf>] kmem_cache_destroy+0x3f/0xe0
[  983.181217]  [<ffffffff811d4949>] kmem_cache_destroy_memcg_children+0x89/0xb0
[  983.181221]  [<ffffffff8118ce84>] kmem_cache_destroy+0x14/0xe0
[  983.181231]  [<ffffffff8121705e>] bioset_free+0xce/0x110
[  983.181250]  [<ffffffffa0002fa0>] __dm_destroy+0x1b0/0x340 [dm_mod]
[  983.181257]  [<ffffffffa00047e3>] dm_destroy+0x13/0x20 [dm_mod]
[  983.181263]  [<ffffffffa000a34e>] dev_remove+0x11e/0x180 [dm_mod]
[  983.181268]  [<ffffffffa000a230>] ? dev_suspend+0x250/0x250 [dm_mod]
[  983.181274]  [<ffffffffa000aa25>] ctl_ioctl+0x255/0x500 [dm_mod]
[  983.181278]  [<ffffffff81271b94>] ? SYSC_semtimedop+0x264/0xd10
[  983.181287]  [<ffffffffa000ace3>] dm_ctl_ioctl+0x13/0x20 [dm_mod]
[  983.181289]  [<ffffffff811f1ef5>] do_vfs_ioctl+0x2e5/0x4c0
[  983.181292]  [<ffffffff811e05ee>] ? ____fput+0xe/0x10
[  983.181294]  [<ffffffff811f2171>] SyS_ioctl+0xa1/0xc0
[  983.181298]  [<ffffffff81645909>] system_call_fastpath+0x16/0x1b
```

看起来是lvconvert过程中缓存分配出现错误, 查到这一步基本无从下手了, 只能google搜索. 在docker项目中发现有相似的情况<https://github.com/moby/moby/issues/29879>, 出问题的三个网址都是3.10的kernel, 回到说是kernel问题, 升到4.16.7-1.el7.elrepo.x86_64没有问题.