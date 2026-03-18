---
title: etcd实战操作指南 - 从部署到监控的完整实践
date: 2026-03-17 15:00:00
tags:
  - etcd
  - hands-on
  - deployment
  - monitoring
  - backup-restore
  - troubleshooting
  - kubernetes
categories:
  - cloud
  - kubernetes-learning
  - tutorial
description: etcd集群部署、配置、监控、备份恢复和故障排查的完整实战指南
---

# etcd实战操作指南

## 集群部署配置

### 三节点etcd集群部署

```yaml
# etcd-cluster.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: etcd-cluster-config
data:
  etcd1.conf: |
    name: etcd-1
    data-dir: /var/lib/etcd
    listen-client-urls: http://0.0.0.0:2379
    advertise-client-urls: http://10.0.0.1:2379
    listen-peer-urls: http://0.0.0.0:2380
    initial-advertise-peer-urls: http://10.0.0.1:2380
    initial-cluster: etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380
    initial-cluster-state: new
    initial-cluster-token: etcd-cluster-1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
spec:
  serviceName: etcd
  replicas: 3
  selector:
    matchLabels:
      app: etcd
  template:
    metadata:
      labels:
        app: etcd
    spec:
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.0
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
        volumeMounts:
        - name: data
          mountPath: /var/lib/etcd
        - name: config
          mountPath: /etc/etcd
        env:
        - name: ETCD_CLUSTER_SIZE
          value: "3"
        command:
        - /usr/local/bin/etcd
        - --config-file=/etc/etcd/etcd.conf
      volumes:
      - name: config
        configMap:
          name: etcd-cluster-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 生产级安全配置

```bash
#!/bin/bash
# etcd-secure-setup.sh

# 1. 生成CA证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# 2. 生成服务器证书
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=server \
  server-csr.json | cfssljson -bare server

# 3. 生成客户端证书
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=client \
  client-csr.json | cfssljson -bare client

# 4. 启动安全etcd
etcd \
  --name=etcd-1 \
  --data-dir=/var/lib/etcd \
  --cert-file=/etc/etcd/ssl/server.pem \
  --key-file=/etc/etcd/ssl/server-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --client-cert-auth \
  --listen-client-urls=https://0.0.0.0:2379 \
  --advertise-client-urls=https://10.0.0.1:2379
```

## 基础操作实战

### 1. 安装etcd客户端工具

```bash
# 方式1: 直接下载
ETCD_VER=v3.5.0
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/{etcd,etcdctl} /usr/local/bin/

# 方式2: 使用包管理器
sudo apt-get install etcd-client  # Ubuntu
brew install etcd                 # macOS
```

### 2. 基本CRUD操作

```bash
# 设置API版本
export ETCDCTL_API=3

# 写入数据
etcdctl put name "kubernetes"
etcdctl put version "1.28"
etcdctl put /config/database/host "mysql.example.com"
etcdctl put /config/database/port "3306"

# 读取数据
etcdctl get name
etcdctl get /config/database/host
etcdctl get /config --prefix    # 前缀匹配

# 删除数据
etcdctl del name
etcdctl del /config --prefix    # 删除所有前缀匹配的键

# 原子操作
etcdctl txn <<< '
compare:
value("/config/lock") = ""

success:
put("/config/lock", "acquired")

failure:
get("/config/lock")
'
```

### 3. Watch机制实战

```bash
# 监听单个键
etcdctl watch mykey

# 监听前缀
etcdctl watch /config --prefix

# 从历史版本开始监听
etcdctl watch mykey --rev=10

# 使用程序监听变化
#!/bin/bash
etcdctl watch /config --prefix | while read line; do
    echo "配置变更: $line"
    # 触发配置重新加载
    systemctl reload my-service
done
```

## 集群管理操作

### 1. 集群状态检查

```bash
# 查看集群成员
etcdctl member list -w table

# 检查集群健康状态
etcdctl endpoint health --cluster
etcdctl endpoint status --cluster -w table

# 详细性能检查
etcdctl check perf

# 查看集群信息
etcdctl cluster-health        # v2 API
etcdctl endpoint health      # v3 API
```

### 2. 集群成员管理

```bash
# 添加新成员
etcdctl member add etcd-4 \
  --peer-urls=http://10.0.0.4:2380

# 移除成员
etcdctl member remove <member-id>

# 更新成员
etcdctl member update <member-id> \
  --peer-urls=http://10.0.0.1:2380
```

### 3. 数据压缩和碎片整理

```bash
# 获取当前revision
rev=$(etcdctl endpoint status --write-out="json" | \
  egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')

# 压缩历史版本
etcdctl compact $rev

# 碎片整理
etcdctl defrag --cluster

# 查看压缩效果
etcdctl endpoint status --cluster -w table
```

## 备份与恢复

### 1. 数据备份

```bash
#!/bin/bash
# etcd-backup.sh

BACKUP_DIR="/backup/etcd"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db"

# 创建备份目录
mkdir -p ${BACKUP_DIR}

# 创建快照
etcdctl snapshot save ${BACKUP_FILE} \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ssl/ca.pem \
  --cert=/etc/etcd/ssl/client.pem \
  --key=/etc/etcd/ssl/client-key.pem

# 验证快照
etcdctl snapshot status ${BACKUP_FILE} -w table

# 清理老备份（保留7天）
find ${BACKUP_DIR} -name "*.db" -mtime +7 -delete

echo "Backup completed: ${BACKUP_FILE}"
```

### 2. 数据恢复

```bash
#!/bin/bash
# etcd-restore.sh

BACKUP_FILE="/backup/etcd/etcd-snapshot-20240317-100000.db"
DATA_DIR="/var/lib/etcd"

# 停止etcd服务
systemctl stop etcd

# 清理现有数据
rm -rf ${DATA_DIR}

# 从快照恢复
etcdctl snapshot restore ${BACKUP_FILE} \
  --data-dir=${DATA_DIR} \
  --name=etcd-1 \
  --initial-cluster=etcd-1=http://10.0.0.1:2380,etcd-2=http://10.0.0.2:2380,etcd-3=http://10.0.0.3:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=http://10.0.0.1:2380

# 修改数据目录权限
chown -R etcd:etcd ${DATA_DIR}

# 启动etcd服务
systemctl start etcd
```

## 性能监控

### 1. 基础监控指标

```bash
# 实时性能监控
watch -n 1 'etcdctl endpoint status --cluster -w table'

# 延迟测试
etcdctl check perf --load="small"
etcdctl check perf --load="large"

# 存储使用情况
etcdctl endpoint status -w json | jq '.[] | {endpoint: .Endpoint, dbSize: .Status.dbSize}'
```

### 2. Prometheus监控配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'etcd'
    static_configs:
      - targets: ['10.0.0.1:2379', '10.0.0.2:2379', '10.0.0.3:2379']
    metrics_path: /metrics
    scheme: https
    tls_config:
      ca_file: /etc/ssl/etcd/ca.pem
      cert_file: /etc/ssl/etcd/client.pem
      key_file: /etc/ssl/etcd/client-key.pem
```

### 3. Grafana仪表板关键指标

```json
{
  "dashboard": {
    "title": "etcd Cluster Monitoring",
    "panels": [
      {
        "title": "Leader Changes",
        "targets": [
          {
            "expr": "rate(etcd_server_leader_changes_seen_total[5m])"
          }
        ]
      },
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(etcd_http_received_total[1m])"
          }
        ]
      },
      {
        "title": "Database Size",
        "targets": [
          {
            "expr": "etcd_mvcc_db_total_size_in_bytes"
          }
        ]
      }
    ]
  }
}
```

## 故障排查实战

### 1. 常见问题诊断

```bash
# 检查集群健康状态
etcdctl endpoint health --cluster

# 查看集群告警
etcdctl alarm list

# 检查磁盘空间
etcdctl endpoint status --cluster -w table

# 查看详细日志
journalctl -u etcd -f

# 网络连接测试
telnet 10.0.0.1 2379
telnet 10.0.0.1 2380
```

### 2. 性能问题排查

```bash
# 磁盘I/O性能测试
sudo fio --randrepeat=1 --ioengine=libaio --direct=1 --gtod_reduce=1 \
  --name=test --filename=/var/lib/etcd/test --bs=4k --iodepth=64 \
  --size=4G --readwrite=randrw --rwmixread=75

# 网络延迟测试
ping -c 10 10.0.0.2

# etcd性能压测
benchmark put --total=10000 --val-size=1024
benchmark range key --total=10000 --consistency=l
```

### 3. 数据一致性检查

```bash
# 检查数据版本一致性
etcdctl endpoint status --cluster -w table

# 对比不同节点的数据
etcdctl get "" --prefix --keys-only --endpoint=http://10.0.0.1:2379 | sort > node1.keys
etcdctl get "" --prefix --keys-only --endpoint=http://10.0.0.2:2379 | sort > node2.keys
diff node1.keys node2.keys

# 检查WAL和快照
ls -la /var/lib/etcd/member/wal/
ls -la /var/lib/etcd/member/snap/
```

## 生产环境最佳实践

### 1. 配置优化

```yaml
# etcd.conf 生产环境配置
name: 'etcd-1'
data-dir: '/var/lib/etcd'
wal-dir: '/var/lib/etcd/wal'
snapshot-count: 100000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 8589934592  # 8GB
max-snapshots: 5
max-wals: 5
cors: "*"
auto-compaction-retention: 1h
auto-compaction-mode: 'periodic'
```

### 2. 监控告警规则

```yaml
# alerting_rules.yml
groups:
- name: etcd
  rules:
  - alert: EtcdClusterDown
    expr: up{job="etcd"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "etcd cluster is down"

  - alert: EtcdHighNumberOfLeaderChanges
    expr: increase(etcd_server_leader_changes_seen_total[1h]) > 3
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High number of leader changes"

  - alert: EtcdDatabaseQuotaExceeded
    expr: etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes > 0.95
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "etcd database quota exceeded"
```

### 3. 自动化运维脚本

```bash
#!/bin/bash
# etcd-health-check.sh

ENDPOINTS="http://10.0.0.1:2379,http://10.0.0.2:2379,http://10.0.0.3:2379"
LOG_FILE="/var/log/etcd-health-check.log"

# 健康检查
check_health() {
    echo "$(date): Checking etcd cluster health..." >> $LOG_FILE

    if etcdctl endpoint health --endpoints=$ENDPOINTS; then
        echo "$(date): Cluster is healthy" >> $LOG_FILE
        return 0
    else
        echo "$(date): Cluster health check failed" >> $LOG_FILE
        return 1
    fi
}

# 自动压缩
auto_compact() {
    rev=$(etcdctl endpoint status --endpoints=$ENDPOINTS --write-out="json" | \
      jq -r '.[] | .Status.header.revision' | sort -n | tail -1)

    if [ $rev -gt 100000 ]; then
        echo "$(date): Auto compacting to revision $rev" >> $LOG_FILE
        etcdctl compact $rev --endpoints=$ENDPOINTS
    fi
}

# 执行检查
check_health
auto_compact
```

---

**这篇实战指南涵盖了etcd从部署到运维的完整操作流程。建议在测试环境中逐一实践这些操作，熟悉etcd的管理和故障处理。**

**相关阅读：**
- [etcd分布式存储原理与实践](./kubernetes-etcd-distributed-storage)
- [Kubernetes集群架构深度解析](./kubernetes-cluster-architecture-overview)
- [Kubernetes核心组件面试题精选](./kubernetes-interview-questions-comprehensive)