### 一、监控方案



- Cadvisor + node-exporter + prometheus + grafana



[数据采集](https://so.csdn.net/so/search?q=数据采集&spm=1001.2101.3001.7020) ——> 汇总 ——> 处理、存储 ——> 展示



### 二、监控流程



##### 容器监控：



Prometheus使用cadvisor采集容器监控指标，而cadvisor集成在K8S的kubelet中所以无需部署，通过Prometheus进程存储，使用grafana进行展示



##### node节点监控：



node端的监控通过node_exporter采集当前主机的资源，通过Prometheus进程存储，最后使用grafana进行展示



##### master节点监控：



master的监控通过kube-state-metrics插件从K8S获取到apiserver的相关数据并通过网页页面暴露出来，然后通过Prometheus进程存储，最后使用grafana进行展示



### 三、部署kube-state-metric



```plain
[root@k8s-master prometheus]# cat kube-state-metrics.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: cnych/kube-state-metrics:v2.3.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app: kube-state-metrics
spec:
  ports:
  - name: kube-state-metrics
    port: 8080
    protocol: TCP
  selector:
    app: kube-state-metrics
```



##### 创建kube-state-metrics



```plain
[root@k8s-master prometheus]# kubectl apply -f  kube-state-metrics.yaml
```



### 四、采用daemonset方式部署node-exporter



**node-export的yaml文件**



```plain
[root@k8s-master prometheus]# cat node-export.yaml 
apiVersion: apps/v1
kind: DaemonSet                         #可以保证 k8s 集群的每个节点都运行完全一样的 pod
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    k8s-app: node-exporter
spec:
  selector:
    matchLabels:
     k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:v0.16.0
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15           #这个容器运行至少需要0.15核cpu
        securityContext:
          privileged: true      #开启特权模式
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /

---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 9100
    nodePort: 31672
    protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
```



**创建node-export**



```plain
[root@master k8s-prometheus-grafana]# kubectl apply -f node-exporter.yaml
daemonset.apps/node-exporter created
service/node-exporter created
```



### 五、部署Prometheus



##### **rbac进行授权**



```plain
[root@master k8s-prometheus-grafana]# cat Prometheus-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  - pods/proxy
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-system
```



##### **Prometheus的配置文件**



```plain
[root@master k8s-prometheus-grafana]# cat Prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-system
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s   #15s抓取一次数据
      evaluation_interval: 15s
    scrape_configs:

    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt   #容器启动自动注入
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    - job_name: 'kubernetes-nodes'  #监控的job
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'     
        replacement: '${1}:9100'    #端口重写，将10250端口替换为9100，9011是定义的nodeport端口
        target_label: __address__ 
        action: replace
      - action: labelmap 
        regex: __meta_kubernetes_node_label_(.+)

    - job_name: 'kubernetes-cadvisor'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name


    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep     #keep保存策略，只有带上述标签的pod才会被保留
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```



##### Prometheus-deploy.yaml



```plain
[root@master k8s-prometheus-grafana]# cat prometheus-deploy.yaml.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      nodeName: k8s-node01
      containers:
      - image: prom/prometheus:v2.2.1
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=15d"  #数据保留策略，保留15d
        - "--web.enable-lifecycle"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      serviceAccountName: prometheus
      volumes:
      - name: data
        hostPath:
          path: /var/prometheus    #挂载到本地，前提是本地有这个目录
      - name: config-volume
        configMap:
          name: prometheus-config
```



##### Prometheus-svc.yaml



```plain
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus
  namespace: kube-system
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30010
  selector:
    app: prometheus
```



##### 依次创建上述文件



```plain
[root@master prometheus]# kubectl apply -f prometheus-rbac.yaml
clusterrole.rbac.authorization.k8s.io/prometheus configured
serviceaccount/prometheus configured
clusterrolebinding.rbac.authorization.k8s.io/prometheus configured
[root@master prometheus]# kubectl apply -f prometheus-configmap.yaml
configmap/prometheus-config configured
[root@master prometheus]# kubectl apply -f prometheus-deploy.yml
deployment.apps/prometheus created
[root@master prometheus]# kubectl apply -f prometheus-svc.yml
service/prometheus created
```







### 六、部署grafana



##### grafana.yaml



```plain
[root@k8s-master prometheus]# cat grafana.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-core
  namespace: kube-system
  labels:
    app: grafana
    component: core
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
        component: core
    spec:
      nodeName: k8s-node01
      containers:
      - image: grafana/grafana:5.0.4
        name: grafana-core
        imagePullPolicy: IfNotPresent
        # env:
        resources:
          limits:
            cpu: 100m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 100Mi
        env:
          # The following env variables set up basic auth twith the default admin user and admin password.
          - name: GF_AUTH_BASIC_ENABLED
            value: "true"
          - name: GF_AUTH_ANONYMOUS_ENABLED
            value: "false"
          # - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          #   value: Admin
          # does not really work, because of template variables in exported dashboards:
          # - name: GF_DASHBOARDS_JSON_ENABLED
          #   value: "true"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
          # initialDelaySeconds: 30
          # timeoutSeconds: 1
        volumeMounts: 
        - name: grafana-persistent-storage
          mountPath: /var/lib/grafana
      volumes:
      - name: grafana-persistent-storage
        hostPath:
          path: /var/grafana    #挂载到本地，前提是本地有这个目录

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-system
  labels:
    app: grafana
    component: core
spec:
  type: NodePort
  ports:
    - port: 3000
      nodePort: 30011
  selector:
    app: grafana
    component: core
```



##### 创建grafana的deploy和svc



```plain
[root@k8s-master prometheus]# kubectl apply -f grafana.yaml 
[root@k8s-master prometheus]# kubectl get svc -n kube-system
```







### 七、检查校验



##### 这是Prometheus的页面，依次点击`Status——Targets`可以看到已经成功连接到k8s的apiserver







-   

##### 访问`http://10.102.10.132/metrics`，这是node-exporter采集的数据







##### 这是grafana的页面，账户、密码都是`admin`







##### 添加数据源







##### 导入模板，根据企业需要进行公式编辑