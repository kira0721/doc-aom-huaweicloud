# ElasticSearch 安装

## 操作场景

安装 ElasticSearch。

> 为了方便安装管理 ElasticSearch，推荐使用华为云 容器服务CCE 进行统一管理。

## 前提条件

- [CCE服务：已有CCE集群](https://console.huaweicloud.com/cce2.0)

## 操作步骤

### 步骤1：pv和pvc配置
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-evs-es
  namespace: aom-middleware-demo
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: es-host
  hostPath:
    path: /data/es-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: es-pvc
  namespace: aom-middleware-demo
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: es-host
```

### 步骤1：elasticsearch.yml配置

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: es-configmap
  namespace: aom-middleware-demo
data:
  elasticsearch.yml: |
    cluster.name: my-cluster
    node.name: node-1
    node.max_local_storage_nodes: 3
    network.host: 0.0.0.0
    http.port: 9200
    discovery.seed_hosts: ["127.0.0.1", "[::1]"]
    cluster.initial_master_nodes: ["node-1"]
    http.cors.enabled: true
    http.cors.allow-origin: /.*/
```

### 步骤1：ElasticSearch 部署

1. 登录 [容器服务](https://console.huaweicloud.com/cce2.0)。
2. 在左侧菜单栏中单击*集群*。
3. 选择某一个集群，进入该集群的管理页面。
4. 选择yml。
    ```yml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: elasticsearch
    spec:
      selector:
        matchLabels:
          name: elasticsearch
      replicas: 1
      template:
        metadata:
          labels:
            name: elasticsearch
        spec:
          initContainers:
          - name: init-sysctl
            image: busybox
            command:
            - sysctl
            - -w
            - vm.max_map_count=262144
            securityContext:
              privileged: true
          containers:
          - name: elasticsearch
            image: elasticsearch:6.8.0
            imagePullPolicy: IfNotPresent
            resources:
              limits:
                cpu: 1000m
                memory: 2Gi
              requests:
                cpu: 100m
                memory: 1Gi
            env:
            - name: ES_JAVA_OPTS
              value: -Xms512m -Xmx512m
            ports:
            - containerPort: 9200
            - containerPort: 9300
            volumeMounts:
            - name: elasticsearch-data
              mountPath: /usr/share/elasticsearch/data/
            - name: es-config
              mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
              subPath: elasticsearch.yml
          volumes:
          - name: elasticsearch-data
            persistentVolumeClaim:
              claimName: es-pvc
            # hostPath:
              # path: /data/es
          - name: es-config
            configMap:
              name: es
    ```
### 步骤2：ElasticSearch 访问