---
title: docker-mountflags
update: 2018-06-07 12:31:15
tags: 
- container
categories: cloud
---

docker run 的时候如果出现类似:

```bash
/usr/bin/docker: Error response from daemon: linux mounts: path /tmp is mounted on / but it is not a shared mount.
```

之类的错误, 可以通过修改Docker启动参数解决, 注释掉mountFlags或者改为shared:

```
$vi /usr/lib/systemd/system/docker.service
MountFlags=slave
```

修改后发现新的错误:

```shell
/usr/bin/docker: Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:402: container init caused \"open /dev/console: input/output error\"": unknown.
FATA[0007] exit status 125
```

网上查有如下回复:

```tex
I met this problem while I suspend my computer, then I restart my computer, this error was solved. I guess it was because the docker daemon missed driver library path.
```

所以重启大法好...

但是真正原因没搞清楚, 待更新.