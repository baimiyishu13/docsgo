+++
title = 'Kube State Metrics'
date = 2024-03-25T10:23:51+08:00
draft = true
+++

适用于集群版本，k8s v1.9

前提准备：

1. 准备镜像

```sh
docker pull bitnami/kube-state-metrics:1.6.0
```
上传到所有worker节点

🎉 完成上述步骤再继续

---

部署：

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
  - apiGroups: [""]
    resources:
      - configmaps
      - secrets
      - nodes
      - pods
      - services
      - resourcequotas
      - replicationcontrollers
      - limitranges
      - persistentvolumeclaims
      - persistentvolumes
      - namespaces
      - endpoints
    verbs: ["list", "watch"]
  - apiGroups: ["extensions"]
    resources:
      - daemonsets
      - deployments
      - replicasets
      - ingresses
    verbs: ["list", "watch"]
  - apiGroups: ["apps"]
    resources:
      - daemonsets
      - deployments
      - replicasets
      - statefulsets
    verbs: ["list", "watch"]
  - apiGroups: ["batch"]
    resources:
      - cronjobs
      - jobs
    verbs: ["list", "watch"]
  - apiGroups: ["autoscaling"]
    resources:
      - horizontalpodautoscalers
    verbs: ["list", "watch"]
  - apiGroups: ["policy"]
    resources:
      - poddisruptionbudgets
    verbs: ["list", "watch"]
  - apiGroups: ["certificates.k8s.io"]
    resources:
      - certificatesigningrequests
    verbs: ["list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources:
      - storageclasses
    verbs: ["list", "watch"]
  - apiGroups: ["autoscaling.k8s.io"]
    resources:
      - verticalpodautoscalers
    verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      containers:
        - image: bitnami/kube-state-metrics:1.6.0
          imagePullPolicy: IfNotPresent
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 30
          name: kube-state-metrics
          ports:
            - containerPort: 8080
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          resources:
            limits:
              cpu: 500m
              memory: 768Mi
            requests:
              cpu: 250m
              memory: 768Mi
      restartPolicy: Always
      serviceAccount: kube-state-metrics
      serviceAccountName: kube-state-metrics
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kube-state-metrics
  name: kube-state-metrics
  namespace: kube-system
spec:
  ports:
    - name: kube-state-metrics
      port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app: kube-state-metrics
  type: NodePort
```

执行：

```
kubectl create -f deployment.yaml
```

1.创建serviceaccounts

```sh
kubectl create sa prometheus -n kube-system
```

2.创建prometheus角色并对其绑定cluster-admin

```sh
kubectl create clusterrolebinding prometheus --clusterrole cluster-admin --serviceaccount=kube-system:prometheus
```

3.创建secret; k8s1.24之后默认不会为serveiceaccounts创建secret

```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: prometheus-token
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: "prometheus"
EOF
```

4.测试访问kube-apiserver

```sh
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl get secret  prometheus-token -n kube-system -o jsonpath='{.data.token}' | base64 --decode)
curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

5.保存token

```sh
echo $TOKEN  > /data/k8s-apiserver_token
```

6.测试访问指标,访问pod性能资源指标：（访问kubelet）

🔔 注意：tw-worker1为当前worker节点的hostname，需要修改

```sh
curl $APISERVER/api/v1/nodes/tw-worker1:10250/proxy/metrics --header "Authorization: Bearer $TOKEN" --insecure
```



配置项：

```yaml
  - job_name: kube-state-metrics
    static_configs:
      - targets: ['1.1.1.1:18566']
  - job_name: "k8s-cadvisor"
    honor_timestamps: true
    metrics_path: /metrics
    scheme: https
    kubernetes_sd_configs:
      - api_server: https://1.1.1.1:6443
        role: node
        bearer_token_file: /data/k8s-apiserver_token
        tls_config:
          insecure_skip_verify: true
    bearer_token_file: /data/k8s-apiserver_token
    tls_config:
      insecure_skip_verify: true
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - separator: ;
        regex: (.*)
        target_label: __address__
        replacement: 1.1.1.1:6443
        action: replace
      - source_labels: [__meta_kubernetes_node_name]
        separator: ;
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}:10250/proxy/metrics/cadvisor
        action: replace
    metric_relabel_configs:
      - source_labels: [instance]
        separator: ;
        regex: (.+)
        target_label: node
        replacement: $1
        action: replace
  - job_name: "kube-node-kubelet"
    scheme: https
    tls_config:
      insecure_skip_verify: true
    bearer_token_file: /data/k8s-apiserver_token
    kubernetes_sd_configs:
      - role: node
        api_server: "https://1.1.1.1:6443"
        tls_config:
          insecure_skip_verify: true
        bearer_token_file: /data/k8s-apiserver_token
    relabel_configs:
      - target_label: __address__
        replacement: 1.1.1.1:6443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}:10250/proxy/metrics
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: service_name
```

🔔 修改 `master1` 位 master节点IP

sed 修改信息后，复制到监控抓取配置

```sh
sed -i 's/1.1.1.1/master1/g' vmagent.yml
```

修改为Nodeport端口

```sh
  - job_name: kube-state-metrics
    static_configs:
      - targets: ['1.1.1.1:18566']
```



