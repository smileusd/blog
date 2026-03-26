---
title: Kubernetes API Server实战示例 - 认证授权与性能测试完整指南
date: 2026-03-18 14:00:00
tags:
  - kubernetes-hands-on
  - apiserver-lab
  - rbac-testing
  - authentication-demo
  - version-conversion
  - watch-mechanism
  - admission-control
  - stress-testing
categories:
  - cloud
  - kubernetes-learning
  - hands-on-labs
description: 全面的Kubernetes API Server实战教程，包含认证授权测试、版本转换验证、Watch机制演示和性能压力测试
---

# Kubernetes API Server实战示例

## 实验环境准备

### 1. 本地集群搭建

```bash
# 使用kind创建本地集群
kind create cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: apiserver-lab
networking:
  apiServerAddress: "127.0.0.1"
  apiServerPort: 6443
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
        extraArgs:
          audit-log-path: /var/log/audit.log
          audit-policy-file: /etc/kubernetes/audit-policy.yaml
          v: "2"
- role: worker
- role: worker
EOF
```

### 2. API Server配置验证

```bash
# 检查API Server状态
kubectl get componentstatuses
kubectl cluster-info

# 查看API Server配置
kubectl get pod kube-apiserver-apiserver-lab-control-plane -n kube-system -o yaml
```

## 实验1: 认证机制测试

### ServiceAccount认证

```bash
# 1. 创建ServiceAccount
kubectl create serviceaccount api-test-sa

# 2. 创建Token
kubectl create token api-test-sa

# 3. 获取Token并测试API调用
TOKEN=$(kubectl create token api-test-sa)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# 4. 使用Token调用API
curl -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -k "$APISERVER/api/v1/namespaces/default/pods" | jq .
```

### 客户端证书认证

```bash
# 1. 生成私钥
openssl genrsa -out user1.key 2048

# 2. 创建证书签名请求
openssl req -new -key user1.key -out user1.csr \
    -subj "/CN=user1/O=developers"

# 3. 获取集群CA证书
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | \
    base64 -d > ca.crt

kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority}' > ca.crt

# 4. 签发用户证书
openssl x509 -req -in user1.csr \
    -CA ca.crt -CAkey ca.key \
    -CAcreateserial -out user1.crt -days 365
```

## 实验2: 授权机制测试

### RBAC权限管理

```yaml
# rbac-test.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: default
roleRef:
  kind: Role
  name: pod-reader-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
# 应用RBAC配置
kubectl apply -f rbac-test.yaml

# 测试权限
kubectl auth can-i get pods --as=system:serviceaccount:default:pod-reader
kubectl auth can-i create pods --as=system:serviceaccount:default:pod-reader
kubectl auth can-i get secrets --as=system:serviceaccount:default:pod-reader
```

### 权限测试脚本

```bash
#!/bin/bash
# test-rbac.sh

SA_NAME="pod-reader"
NAMESPACE="default"

echo "🔐 测试RBAC权限"
echo "================="

# 获取ServiceAccount Token
TOKEN=$(kubectl create token $SA_NAME -n $NAMESPACE)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

echo "1. 测试允许的操作 (应该成功)"
echo "----------------------------------------"

echo "获取Pod列表:"
curl -s -H "Authorization: Bearer $TOKEN" \
     -k "$APISERVER/api/v1/namespaces/$NAMESPACE/pods" | \
     jq '.items[] | {name: .metadata.name, phase: .status.phase}'

echo ""
echo "2. 测试禁止的操作 (应该失败)"
echo "----------------------------------------"

echo "创建Pod:"
curl -s -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -k -X POST "$APISERVER/api/v1/namespaces/$NAMESPACE/pods" \
     -d '{"apiVersion":"v1","kind":"Pod","metadata":{"name":"test"}}' | \
     jq '.code, .message'

echo "获取Secret:"
curl -s -H "Authorization: Bearer $TOKEN" \
     -k "$APISERVER/api/v1/namespaces/$NAMESPACE/secrets" | \
     jq '.code, .message'
```

## 实验3: API版本转换测试

### 版本转换验证

```yaml
# deployment-v1beta1.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: version-test
  labels:
    app: version-demo
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
        version: demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

```bash
# 版本转换测试脚本
#!/bin/bash
echo "🔄 API版本转换测试"
echo "=================="

# 1. 创建v1beta1版本资源
echo "1. 使用v1beta1创建Deployment"
kubectl apply -f deployment-v1beta1.yaml

echo ""
echo "2. 查看不同版本的返回格式"
echo "-------------------------"

echo "默认版本 (kubectl默认使用apps/v1):"
kubectl get deployment version-test -o yaml | head -10

echo ""
echo "明确请求apps/v1版本:"
kubectl get deployment.v1.apps version-test -o jsonpath='{.apiVersion}'

echo ""
echo "3. 检查自动生成的字段"
echo "---------------------"
echo "Selector字段 (v1beta1没有，系统自动生成):"
kubectl get deployment version-test -o jsonpath='{.spec.selector}' | jq .

echo ""
echo "Strategy字段 (系统自动添加默认策略):"
kubectl get deployment version-test -o jsonpath='{.spec.strategy}' | jq .

# 清理
kubectl delete deployment version-test
```

## 实验4: Watch机制测试

### 实时事件监控

```python
#!/usr/bin/env python3
# watch-events.py
import json
import requests
import time
from kubernetes import client, config, watch

def watch_pods():
    # 加载kubeconfig
    config.load_kube_config()
    v1 = client.CoreV1Api()

    print("🔍 开始监控Pod事件...")
    print("==================")

    # 创建Watch对象
    w = watch.Watch()

    try:
        # 监控Pod事件
        for event in w.stream(v1.list_namespaced_pod, namespace="default", timeout_seconds=60):
            event_type = event['type']
            pod = event['object']
            pod_name = pod.metadata.name
            pod_phase = pod.status.phase

            timestamp = time.strftime("%H:%M:%S")
            print(f"[{timestamp}] {event_type}: {pod_name} ({pod_phase})")

            # 详细信息
            if event_type in ['ADDED', 'MODIFIED']:
                if hasattr(pod.status, 'container_statuses') and pod.status.container_statuses:
                    for container in pod.status.container_statuses:
                        print(f"  Container: {container.name}, Ready: {container.ready}")

    except KeyboardInterrupt:
        print("\n监控结束")
    finally:
        w.stop()

if __name__ == "__main__":
    watch_pods()
```

### 使用curl监控事件

```bash
#!/bin/bash
# watch-with-curl.sh

TOKEN=$(kubectl create token default)
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

echo "📡 使用curl监控Pod事件"
echo "===================="

# 在后台启动事件监控
curl -s -H "Authorization: Bearer $TOKEN" \
     -k "$APISERVER/api/v1/namespaces/default/pods?watch=true" | \
while read -r event; do
    if [ -n "$event" ]; then
        echo "[$(date '+%H:%M:%S')] $event" | jq -r '{type: .type, name: .object.metadata.name, phase: .object.status.phase}'
    fi
done &

WATCH_PID=$!

# 创建测试Pod来生成事件
sleep 2
echo "创建测试Pod..."
kubectl run watch-test --image=nginx --rm --restart=Never -- sleep 30

# 等待一段时间
sleep 35

# 停止监控
kill $WATCH_PID 2>/dev/null
echo "监控结束"
```

## 实验5: 准入控制器测试

### Validating Webhook示例

```yaml
# validation-webhook.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webhook-config
data:
  webhook.py: |
    from flask import Flask, request, jsonify
    import base64
    import json

    app = Flask(__name__)

    @app.route('/validate', methods=['POST'])
    def validate():
        admission_request = request.get_json()

        # 获取要创建的对象
        obj = admission_request.get('request', {}).get('object', {})

        # 验证逻辑：禁止创建名称包含"bad"的Pod
        if obj.get('kind') == 'Pod':
            name = obj.get('metadata', {}).get('name', '')
            if 'bad' in name:
                return jsonify({
                    'apiVersion': 'admission.k8s.io/v1',
                    'kind': 'AdmissionResponse',
                    'response': {
                        'uid': admission_request['request']['uid'],
                        'allowed': False,
                        'status': {
                            'code': 403,
                            'message': 'Pod名称不能包含"bad"'
                        }
                    }
                })

        # 默认允许
        return jsonify({
            'apiVersion': 'admission.k8s.io/v1',
            'kind': 'AdmissionResponse',
            'response': {
                'uid': admission_request['request']['uid'],
                'allowed': True
            }
        })

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=8443, ssl_context='adhoc')

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: validation-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: validation-webhook
  template:
    metadata:
      labels:
        app: validation-webhook
    spec:
      containers:
      - name: webhook
        image: python:3.9-slim
        command: ["python", "-c"]
        args:
        - |
          import subprocess
          subprocess.run(["pip", "install", "flask", "pyopenssl"])
          exec(open('/config/webhook.py').read())
        ports:
        - containerPort: 8443
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: webhook-config
```

### 测试准入控制

```bash
# 测试准入控制脚本
#!/bin/bash
echo "🛡️ 测试准入控制器"
echo "================"

# 1. 部署webhook
kubectl apply -f validation-webhook.yaml

# 等待Pod就绪
kubectl wait --for=condition=Ready pod -l app=validation-webhook --timeout=60s

# 2. 创建ValidatingAdmissionWebhook配置
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: pod-name-validator
webhooks:
- name: validate-pod-name
  clientConfig:
    service:
      name: validation-webhook
      namespace: default
      path: /validate
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
EOF

echo "3. 测试准入控制效果"
echo "-------------------"

echo "创建正常Pod (应该成功):"
kubectl run good-pod --image=nginx --dry-run=server -o name

echo ""
echo "创建包含'bad'的Pod (应该被拒绝):"
kubectl run bad-pod --image=nginx --dry-run=server -o name || echo "❌ 被准入控制器拒绝"

# 清理
kubectl delete validatingadmissionwebhook pod-name-validator
kubectl delete deployment validation-webhook
kubectl delete configmap webhook-config
```

## 实验6: 性能监控和调优

### API Server性能指标监控

```bash
#!/bin/bash
# monitor-apiserver.sh

echo "📊 API Server性能监控"
echo "==================="

# 获取API Server Pod名称
API_POD=$(kubectl get pod -n kube-system -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')

echo "API Server Pod: $API_POD"
echo ""

echo "1. 基础资源使用情况"
echo "-------------------"
kubectl top pod $API_POD -n kube-system

echo ""
echo "2. API Server内置监控指标"
echo "-------------------------"

# 通过proxy访问metrics端点
kubectl proxy --port=8080 &
PROXY_PID=$!
sleep 2

echo "请求延迟指标:"
curl -s http://localhost:8080/api/v1/namespaces/kube-system/pods/$API_POD:8080/proxy/metrics | \
    grep "apiserver_request_duration_seconds_bucket" | head -5

echo ""
echo "请求总数:"
curl -s http://localhost:8080/api/v1/namespaces/kube-system/pods/$API_POD:8080/proxy/metrics | \
    grep "apiserver_request_total" | head -5

echo ""
echo "当前飞行中请求:"
curl -s http://localhost:8080/api/v1/namespaces/kube-system/pods/$API_POD:8080/proxy/metrics | \
    grep "apiserver_current_inflight_requests"

# 停止proxy
kill $PROXY_PID 2>/dev/null

echo ""
echo "3. etcd连接状态"
echo "---------------"
# 检查etcd健康状态
kubectl get --raw="/api/v1/namespaces/kube-system/pods/$API_POD:2379/proxy/health" || \
echo "无法直接访问etcd指标，请检查集群配置"
```

### 压力测试

```bash
#!/bin/bash
# stress-test-api.sh

echo "⚡ API Server压力测试"
echo "=================="

# 创建测试命名空间
kubectl create namespace stress-test || true

echo "1. 并发创建Pod测试"
echo "------------------"

# 函数：创建Pod
create_pod() {
    local i=$1
    kubectl run stress-pod-$i --image=nginx:alpine --namespace=stress-test \
        --requests=cpu=10m,memory=20Mi --limits=cpu=50m,memory=100Mi 2>/dev/null
}

# 并发创建50个Pod
echo "并发创建50个Pod..."
for i in {1..50}; do
    create_pod $i &
done

# 等待所有后台任务完成
wait

echo "创建完成，检查结果:"
kubectl get pods -n stress-test --no-headers | wc -l

echo ""
echo "2. 并发查询测试"
echo "---------------"

# 并发查询测试
start_time=$(date +%s)

for i in {1..20}; do
    kubectl get pods -n stress-test >/dev/null 2>&1 &
done

wait
end_time=$(date +%s)
duration=$((end_time - start_time))

echo "20次并发查询耗时: ${duration}秒"

echo ""
echo "3. Watch连接压力测试"
echo "-------------------"

# 创建多个watch连接
for i in {1..5}; do
    kubectl get pods -n stress-test -w >/dev/null 2>&1 &
    WATCH_PIDS+=($!)
done

echo "创建了5个并发watch连接"
sleep 10

# 在watch期间修改资源
echo "在watch期间执行资源修改..."
kubectl scale deployment --all --replicas=3 -n stress-test 2>/dev/null || true

sleep 5

# 停止所有watch
for pid in "${WATCH_PIDS[@]}"; do
    kill $pid 2>/dev/null
done

echo ""
echo "4. 清理测试资源"
echo "---------------"
kubectl delete namespace stress-test
echo "✅ 压力测试完成"
```

## 实验7: 故障排查实践

### 常见故障模拟

```bash
#!/bin/bash
# troubleshoot-lab.sh

echo "🔧 API Server故障排查实验"
echo "========================"

echo "1. 模拟认证失败"
echo "---------------"

# 使用错误token访问API
echo "使用无效token访问:"
curl -H "Authorization: Bearer invalid-token" \
     -k "$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')/api/v1/pods" 2>/dev/null | \
     jq '.code, .message'

echo ""
echo "2. 模拟授权失败"
echo "---------------"

# 创建受限ServiceAccount
kubectl create serviceaccount limited-sa 2>/dev/null || true

# 使用受限账户尝试访问
LIMITED_TOKEN=$(kubectl create token limited-sa)
echo "使用无权限账户访问secrets:"
curl -H "Authorization: Bearer $LIMITED_TOKEN" \
     -k "$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')/api/v1/namespaces/default/secrets" | \
     jq '.code, .message'

echo ""
echo "3. 模拟API版本错误"
echo "------------------"

echo "访问不存在的API版本:"
curl -k "$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')/api/v999/pods" | \
     jq '.code, .message'

echo ""
echo "4. 模拟资源不存在"
echo "-----------------"

echo "访问不存在的Pod:"
kubectl get pod non-existent-pod 2>&1 | tail -1

echo ""
echo "5. 检查API Server日志"
echo "-------------------"

API_POD=$(kubectl get pod -n kube-system -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')
echo "API Server Pod: $API_POD"
echo "最近的日志 (最后10行):"
kubectl logs $API_POD -n kube-system --tail=10

# 清理
kubectl delete serviceaccount limited-sa 2>/dev/null || true
```

### 诊断工具集

```bash
#!/bin/bash
# diagnostic-tools.sh

echo "🩺 API Server诊断工具集"
echo "====================="

echo "1. 集群基本信息"
echo "---------------"
kubectl cluster-info
kubectl version --short

echo ""
echo "2. API Server健康检查"
echo "-------------------"
kubectl get --raw="/livez" && echo " - API Server存活检查: ✅"
kubectl get --raw="/readyz" && echo " - API Server就绪检查: ✅"

echo ""
echo "3. 可用API版本"
echo "-------------"
kubectl api-versions | head -10
echo "... (总共 $(kubectl api-versions | wc -l) 个API版本)"

echo ""
echo "4. 支持的资源类型"
echo "----------------"
kubectl api-resources | grep -E "(pods|deployments|services)" | \
    awk '{printf "%-30s %-15s %s\n", $1, $3, $4}'

echo ""
echo "5. 控制平面组件状态"
echo "------------------"
kubectl get pods -n kube-system -l tier=control-plane

echo ""
echo "6. API Server配置检查"
echo "-------------------"
API_POD=$(kubectl get pod -n kube-system -l component=kube-apiserver -o jsonpath='{.items[0].metadata.name}')

if [ -n "$API_POD" ]; then
    echo "API Server启动参数关键配置:"
    kubectl get pod $API_POD -n kube-system -o yaml | \
        grep -A 20 "command:" | \
        grep -E "(audit-log|authorization-mode|enable-admission-plugins|etcd-servers)" | \
        head -10
else
    echo "⚠️  无法找到API Server Pod"
fi

echo ""
echo "7. etcd连接测试"
echo "---------------"
ETCD_POD=$(kubectl get pod -n kube-system -l component=etcd -o jsonpath='{.items[0].metadata.name}')
if [ -n "$ETCD_POD" ]; then
    echo "etcd Pod: $ETCD_POD"
    kubectl exec $ETCD_POD -n kube-system -- etcdctl endpoint health 2>/dev/null || \
        echo "⚠️  etcd健康检查失败或访问受限"
else
    echo "⚠️  无法找到etcd Pod (可能是托管集群)"
fi
```

## 总结和最佳实践

### 🎯 关键学习点

1. **认证授权**：理解多层安全机制的配置和测试
2. **版本转换**：掌握API版本兼容性和转换过程
3. **性能监控**：熟练使用内置指标和监控工具
4. **故障排查**：具备系统性的问题诊断能力

### 📋 实验检查清单

- [ ] 完成认证机制测试 (ServiceAccount + 证书)
- [ ] 验证RBAC权限控制
- [ ] 理解API版本转换过程
- [ ] 掌握Watch机制使用
- [ ] 配置准入控制器
- [ ] 进行性能压力测试
- [ ] 练习故障排查技能

### 🚀 进阶实践建议

1. **源码阅读**：结合实验深入理解API Server源码
2. **生产环境**：在真实环境中应用学到的技能
3. **自动化**：编写脚本自动化常见运维任务
4. **监控告警**：搭建完整的监控和告警体系

---

**这是Kubernetes API Server的完整实战指南，通过动手实验深入理解API Server的核心机制和最佳实践。建议按照实验顺序逐步完成，并在真实环境中应用所学技能。**

**系列文章导航：**
- [Kubernetes API Server深度解析](./kubernetes-apiserver-deep-dive) ← 基础概述
- [Kubernetes API Server架构设计深度剖析](./kubernetes-apiserver-architecture-detailed) ← 架构详解
- [Kubernetes API Server源码分析](./kubernetes-apiserver-source-code-analysis) ← 源码实现
- [Kubernetes API Server故障排查完整指南](./kubernetes-apiserver-troubleshooting-guide) ← 故障排查
- [Kubernetes核心组件学习系列概览](./kubernetes-learning-series-overview)