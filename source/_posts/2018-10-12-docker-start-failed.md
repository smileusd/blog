---
title: docker-start-failed
update: 2018-10-12 14:44:39
tags: docker
categories: cloud
---

make build过程中忽然出现错误:

```
Sending build context to Docker daemon   220 MB
Step 1 : FROM warpdrive:tos-release-1-5
 ---> 769306738d96
Step 2 : COPY . /go/src/github.com/transwarp/warpdrive/
 ---> 07c99697b16e
Removing intermediate container 127c0e71a84b
Successfully built 07c99697b16e
/usr/bin/docker: Error response from daemon: invalid header field value "oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:245: running exec setns process for init caused \\\"exit status 6\\\"\"\n".
FATA[0301] exit status 125                              
make: *** [build] Error 1
```

尝试run 07c99697b16e报同样的错, 查看docker日志, 出现`"oci runtime error: container_linux.go:247: starting container process caused \"process_linux .go:245: running exec setns process for init caused \\\"exit status 6\\\"\"\n"`

```shell
8月 21 15:29:01 zhenghc-tos dockerd[2517]: time="2018-08-21T15:29:01.778212288+08:00" level=error msg="Handler for POST /v1.24/containers/79a0906242e0180dfd7aeff30e9fb179f7b7c37f0ba20533f83b4cd40b409d2a/start returned error: invali
d header field value \"oci runtime error: container_linux.go:247: starting container process caused \\\"process_linux.go:245: running exec setns process for init caused \\\\\\\"exit status 6\\\\\\\"\\\"\\n\""
8月 21 15:51:08 zhenghc-tos dockerd[2517]: time="2018-08-21T15:51:08.075631868+08:00" level=error msg="containerd: start container" error="oci runtime error: container_linux.go:247: starting container process caused \"process_linux
.go:245: running exec setns process for init caused \\\"exit status 6\\\"\"\n" id=b93539a02f27cc1ad0114552643d8e0c942fba053dc8cd3980cb8f24cf61ff25
8月 21 15:51:08 zhenghc-tos dockerd[2517]: time="2018-08-21T15:51:08.092002257+08:00" level=error msg="Create container failed with error: invalid header field value \"oci runtime error: container_linux.go:247: starting container p
rocess caused \\\"process_linux.go:245: running exec setns process for init caused \\\\\\\"exit status 6\\\\\\\"\\\"\\n\""
```

发现Docker比较卡, 于是清理了大量的images, 删掉了所有未运行的container, 再清理了系统缓存, 问题解决了, 但深层次的问题没弄清楚, 尽管清了缓存, 但是清完其实缓存差距不大, docker 也没有显示资源问题, 另外一个点就是, 只有这个image是进不去的, 其他的Images是可以run -it的, 而且在做`COPY . /go/src/github.com/transwarp/warpdrive/`这一步时间等的比较长.

<https://github.com/opencontainers/runc/issues/1130> 记录了相关的问题, 说runc的问题.