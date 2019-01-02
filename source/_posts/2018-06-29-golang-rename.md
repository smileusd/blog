---
title: 记一次疑云重重的kubelet延迟很高的问题
update: 2018-06-29 11:19:32
tags: kubernetes
categories: cloud
---
最近在调查一个kubernetes中发现Kubelet的pods目录:
```
/var/lib/kubelet/pods/xxx/volumes/
```
下出现了大量的包含"deleting~" 的目录:
```bash
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~859156558
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~912994645
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~096627888
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~361944655
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~827756898
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~850958169
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~435144420
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~573873907
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~817019830
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~300298653
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~414447192
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~453118423
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~634999626
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~329196065
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~705907980
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~060876539
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~371568670
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~473777381
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~852926720
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~911951455
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~221614642
/var/lib/kubelet/pods/612f2e76-3ce1-11e8-b2c9-0cc47ae2b22c/volumes/transwarp.io~tosdisk/pvc-f39e86b9-019b-11e8-b2c9-0cc47ae2b22c.deleting~643761641
```
导致每次reconciler将这些多余的"deleting~"加入到"ActualOfWorld"中, 然后触发大量的Umount操作, 使得reconciler很久才Loop一次, 现象就是pod create和delete都变得非常得慢. 
一. 一开始, 我发现自己写的plugin中使用了pkg/volume/volume.go中的RenameDirectory函数, 函数如下:
```go
func RenameDirectory(oldPath, newName string) (string, error) {
	newPath, err := ioutil.TempDir(filepath.Dir(oldPath), newName)
	if err != nil {
		return "", err
	}

	// os.Rename call fails on windows (https://github.com/golang/go/issues/14527)
	// Replacing with copyFolder to the newPath and deleting the oldPath directory
	if runtime.GOOS == "windows" {
		err = copyFolder(oldPath, newPath)
		if err != nil {
			glog.Errorf("Error copying folder from: %s to: %s with error: %v", oldPath, newPath, err)
			return "", err
		}
		os.RemoveAll(oldPath)
		return newPath, nil
	}

	err = os.Rename(oldPath, newPath)
	if err != nil {
		return "", err
	}
	return newPath, nil
}

```
在每次删除目录时, 并不是直接删除, 而是先创建一个随机的空目录, 然后将原目录rename到随机目录, 最后再将这个随机目录删除掉.
看似没有什么问题, 但不巧的是, 在golang1.8之后, os.Rename的实现发生了变化
```go
func rename(oldname, newname string) error {
	fi, err := Lstat(newname)
	if err == nil && fi.IsDir() {
		// There are two independent errors this function can return:
		// one for a bad oldname, and one for a bad newname.
		// At this point we've determined the newname is bad.
		// But just in case oldname is also bad, prioritize returning
		// the oldname error because that's what we did historically.
		if _, err := Lstat(oldname); err != nil {
			if pe, ok := err.(*PathError); ok {
				err = pe.Err
			}
			return &LinkError{"rename", oldname, newname, err}
		}
		return &LinkError{"rename", oldname, newname, syscall.EEXIST}
	}
	//------------------------版本分割线1.8
	err = syscall.Rename(oldname, newname)
	if err != nil {
		return &LinkError{"rename", oldname, newname, err}
	}
	return nil
}
```
在虚线以上是go1.8之后新加的内容, 如果rename之后的目录存在, 就会打印"File Exits"错误, 这样就会创建大量的"deleting~"目录. 相关修改和讨论在[Bugs in EmptyDir Teardown path](https://github.com/kubernetes/kubernetes/issues/43534).

以为问题就这样解决了, 然而不是, 我检查了版本, 使用的是1.7.4, 同时查看了日志, 并没有打印"FILE Exist"的log, 而是打印了"device or resource busy". 通过多种测试, 发现rename操作无论是进程占用还是进程对文件的读写, 都不会导致device busy问题. 这说明我找到了出问题的地方, 却没找到背后的原因. 

二. 如果rename没有返回错误, 那么只能是之后的remove操作返回的错误. 然而却有几个疑点:
1. 查看rename出来的deleting目录中都是空目录, 里面并没有数据
2. 手动可以remove掉这些目录
3. 原目录并没有消失, 而是存在且有大量的正在更新的数据.

​    显然问题比我想象的更加复杂. 有一个可能的解释是在我操作delete之后由于太长时间没有删掉导致Kubelet直接暴力删除了这个pod, statefulset又重启了新的pod, 原来的container虽然删掉但相应的namespaces中依然有应用程序在读写, 这种情况导致remove busy, 而且数据在更新.

这些问题要调查起来都很费力. 显然直接删掉rename逻辑可以解决这个问题, 于是不再追究.