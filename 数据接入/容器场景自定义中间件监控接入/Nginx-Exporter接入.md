# Nginx Exporter 接入

## 操作场景

Nginx 通过 stub_status 页面暴露了部分监控指标。Nginx Prometheus Exporter 会采集单个 Nginx 实例指标，并将其转化为 Prometheus 可用的监控数据， 最终通过 HTTP 协议暴露给 Prometheus 服务进行采集。我们可以通过 Exporter 上报重点关注的监控指标，用于异常报警和大盘展示。

> 为了方便安装管理 Exporter，推荐使用华为云 容器服务CCE 进行统一管理。

## 前提条件

- [CCE服务：已有CCE集群](https://console.huaweicloud.com/cce2.0)
- [华为云镜像服务SWR](https://console.huaweicloud.com/swr)
- [AOM服务：创建Prometheus For CCE实例，实例已关联集群](https://console.huaweicloud.com/aom2)
- 在集群中 安装exporter命名空间 和对应的 安装中间件命名空间 要相同

## 操作步骤

### Nginx Exporter 部署

用 [nginx-prometheus-exporter](https://hub.docker.com/r/nginx/nginx-prometheus-exporter) 镜像将服务部署在腾讯云 容器服务 TKE 中，通过托管 Prometheus 的监控自发现 CRD PodMonitor 或者 ServiceMonitor 来采集监控数据。

### 使用二进制程序运行 Exporter

#### 下载安装

1. 根据实际运行环境在社区中下载相应的 [Nginx Prometheus Exporter](https://github.com/nginxinc/nginx-prometheus-exporter/releases) 。
2. 安装 Nginx Prometheus Exporter。

#### 开启 NGINX stub_status 功能
1. 开源 Nginx 提供一个简单页面用于展示状态数据，该页面由 stub_status 模块提供。执行以下命令检查 Nginx 是否已经开启了该模块：
```bash
nginx -V 2>&1 | grep -o with-http_stub_status_module
```
   + 如果在终端中输出 with-http_stub_status_module ，则说明 Nginx 已启用 tub_status 模块。
   + 如果未输出任何结果，则可以使用 --with-http_stub_status_module 参数从源码重新配置编译一个 Nginx。示例如下：
    ```bash
    ./configure \
    … \
    --with-http_stub_status_module
    make
    sudo make install
    ```
2. 确认 stub_status 模块启用之后，修改 Nginx 的配置文件指定 status 页面的 URL。示例如下：
```bash
server {
 location = /nginx_status {
     stub_status;
 }
}
```
3. 检查并重新加载 Nginx的配置使其生效。
```bash
nginx -t
nginx -s reload
```
4. 完成上述步骤之后，可以通过配置的 URL 查看 Nginx 的指标：
```bash
curl http://192.168.10.10:8080/stub_status

Active connections: 45
server accepts handled requests
1056958 1156958 4491319
Reading: 0 Writing: 25 Waiting : 7
```
#### 运行 NGINX Prometheus Exporter
执行以下命令启动 NGINX Prometheus Exporter：
```bash
$ ./nginx-prometheus-exporter -nginx.scrape-uri=http://192.168.10.10:8080/nginx_status
```

#### 上报指标
+ nginxexporter_build_info -- exporter 编译信息。
+ 所有的 stub_status 指标。
+ nginx_up -- 展示上次抓取的状态：1表示抓取成功， 0表示抓取失败。

5. 登录 [容器服务](https://console.huaweicloud.com/cce2.0)。
6. 在左侧菜单栏中单击*集群*。
7. 选择某一个集群，进入该集群的管理页面。
8. 执行以下3个步骤完成 Exporter 部署。
   4.1 配置项与密钥 > YAML创建,输入以下yml文件，密码是按照Opaque加密过的。
   使用 Secret 管理 Memcached 连接串，详见以下说明
   >  Memcached 连接串的格式为 http://10.247.43.50:11211
    【建议】也可以使用界面化操作：
    ![Alt text](images/image12.png)
    ```yml
    apiVersion: v1
    kind: Secret
    metadata:
      name: memcached-exporter-secret
      namespace: aom-middleware-demo
    type: Opaque
    data:
      memcachedURI: 10.247.167.43:11211  # 对应中间件监控地址,这里要转成密文才能通过yml文件创建
    ```
    4.2 部署 Memcached Exporter
    在 Deployment 管理页面，单击新建，选择对应的命名空间来进行部署服务。可以通过控制台的方式创建，如下以 YAML 的方式部署 Exporter，YAML 配置示例如下：
    > 更多 Exporter 详细参数介绍请参见 [memcached_exporter](https://github.com/prometheus/memcached_exporter)。

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    labels:
        k8s-app: memcached-exporter # 根据业务需要调整成对应的名称，建议加上 Redis 实例的信息
    name: memcached-exporter # 根据业务需要调整成对应的名称，建议加上 Redis 实例的信息
    namespace: aom-middleware-demo
    spec:
    replicas: 1
    selector:
        matchLabels:
        k8s-app: memcached-exporter # 根据业务需要调整成对应的名称，建议加上 Redis 实例的信息
    template:
        metadata:
        labels:
            k8s-app: memcached-exporter # 根据业务需要调整成对应的名称，建议加上 Redis 实例的信息
        spec:
        containers:
            - name: memcached-exporter
            env:
                - name: Memcached_Url
                valueFrom:
                    secretKeyRef:
                    name: memcached-exporter-secret
                    key: memcachedURI
            command: ["memcached_exporter", "--memcached.address=$(Memcached_Url)"]
            image: swr.cn-east-3.myhuaweicloud.com/aom-org/bitnami/memcached-exporter:0.13.0
            imagePullPolicy: IfNotPresent
            ports:
                - containerPort: 9150
                name: metric-port  # 这个名称在配置抓取任务的时候需要
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
            securityContext:
                privileged: false
        dnsPolicy: ClusterFirst
        imagePullSecrets:
            - name: default-secret
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30
    ```
    4.3 验证
    1. Deployment列表>Deployment详情>Pod实例>更多>日志，查看，可以查看到Exporter成功启动并暴露对应的访问地址，如下图所示：
    ![Alt text](images/image13.png)
    2. 单击 Pod 管理页签，进入 Pod 页面。
    3. 也可以通过创建一个外网的service，验证暴露的地址是否可以正常得到对应的memcached指标。
    ![Alt text](images/image15.png)
    4. 访问地址：http://123.60.5.226:9114/metrics,如发现未能得到对应的数据，验证完之后，可以把service删除，示例如下：
    ![Alt text](images/image14.png)

### 步骤2：添加采集任务

- 登录 [AOM](https://console.huaweicloud.com/aom2)
- 在左侧菜单栏中单击Prometheus监控，选择对应的Prometheus实例（For CCE实例类型）进入管理面。
- 点击服务发现
- 通过服务发现添加PodMonitor

```yml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: memcached-exporter
  namespace: aom-middleware-demo
spec:
  namespaceSelector:
    matchNames:
      - aom-middleware-demo
  podMetricsEndpoints:
  - interval: 30s
    path: /metrics
    port: metric-port
  selector:
    matchLabels:
      k8s-app: memcached-exporter
```

## 在AOM上配置仪表盘和告警
![Alt text](images/image16.png)
![Alt text](images/image17.png)