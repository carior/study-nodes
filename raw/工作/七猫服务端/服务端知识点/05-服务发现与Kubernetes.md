# 服务发现、Kubernetes 与云平台

## 1. 什么是服务注册与发现

业务实例启动后，可以向注册中心登记自己的信息：

```text
服务名称：order-service
实例地址：10.0.1.11:8080
健康状态：正常
```

网关或其他服务向注册中心查询：

```text
order-service 目前有哪些可用实例？
```

这样实例扩容、缩容或重启后，不必人工修改所有调用方的配置。

## 2. Nacos、Consul 和 etcd

### Nacos

Nacos 常用于 Java、Spring Cloud Alibaba 系统，主要提供：

- 服务注册与发现
- 配置管理
- 服务健康状态
- 命名空间和环境隔离

Nacos 通常提供 Web 管理页面，可以看到服务及其实例列表。常见地址形式为：

```text
http://nacos地址:8848/nacos
```

实际地址和账号通常由运维或平台团队管理。

### Consul

Consul 是 HashiCorp 提供的服务发现和健康检查工具，也支持键值配置和服务间网络管理，并提供 Web UI。

### etcd

etcd 是分布式键值存储，适合保存集群配置和状态。Kubernetes 使用 etcd 保存自身的集群数据。

etcd 更偏向底层存储，业务开发人员通常不会像使用 Nacos 那样直接把业务服务注册到 Kubernetes 的 etcd 中。

三者不需要同时使用：

- Spring Cloud Alibaba 项目经常使用 Nacos。
- 一些微服务系统使用 Consul。
- Kubernetes 底层使用 etcd，并通过自己的资源体系管理服务和实例。

## 3. Kubernetes 是什么

Kubernetes，简称 K8s，是用于管理大量容器化服务实例的平台。

当系统有几十个服务和大量实例时，需要持续处理以下问题：

```text
实例应该运行在哪台机器？
实例崩溃后怎么办？
流量增加时如何扩容？
新版本如何逐步替换旧版本？
服务之间如何发现彼此？
```

Kubernetes 可以负责：

- 启动和停止实例
- 实例异常后自动重启
- 根据负载扩容和缩容
- 滚动发布
- 服务发现和负载均衡
- 配置和密钥管理
- 将实例调度到合适的服务器

例如声明订单服务始终运行三个实例，其中一个崩溃后，Kubernetes 会自动创建新实例，恢复到三个。

## 4. Pod、Deployment、Service 和 Ingress

### Pod

Pod 是 Kubernetes 中运行应用容器的基本单位，可以把它理解为一个服务实例的运行载体。

```text
Pod A：10.244.1.10:8080
Pod B：10.244.2.15:8080
Pod C：10.244.3.20:8080
```

Pod 重启后 IP 可能变化，不适合作为调用方长期保存的地址。

### Deployment

Deployment 描述一个服务应该使用哪个镜像、运行几个 Pod，以及如何更新版本。

例如：

```text
订单服务镜像：order-service:v2
期望实例数量：3
```

### Service

Kubernetes Service 为一组 Pod 提供稳定地址：

```text
http://order-service:8080
```

请求关系：

```text
网关
  |
  v
order-service（稳定地址）
  |
  +-> Pod A
  +-> Pod B
  +-> Pod C
```

Service 根据标签选择 Pod：

```yaml
selector:
  app: order-service
```

满足标签的 Pod 会成为 Service 的后端实例。

因此，服务和实例都有地址：

- Pod 地址指向一个具体实例，可能变化。
- Service 地址代表一组实例，是相对稳定的访问入口。

银行类比：Pod 地址是“3 号窗口”，Service 地址是“对公业务取号入口”。

### Ingress

Ingress 定义外部域名和路径如何进入集群，例如：

```text
test-1.example.com/api/orders -> order-service
```

它通常需要配合 Ingress Controller，例如 Nginx Ingress。

## 5. Kubernetes 环境中的请求链路

```text
客户端
  |
  v
域名 / 云负载均衡
  |
  v
Ingress / API 网关
  |
  v
Kubernetes Service
  |
  +-> Pod A
  +-> Pod B
  +-> Pod C
```

在这种环境中，Nginx 通常不需要手工记录每个 Pod IP，而是访问稳定的 Service：

```nginx
upstream order_service {
    server order-service:8080;
}
```

Pod 增加、减少或地址变化由 Kubernetes 自动管理。

## 6. 云平台是什么

云平台提供可以按需使用的计算资源和基础设施，例如：

- 云服务器
- 云数据库
- Redis
- Kafka
- 对象存储
- 云负载均衡
- Kubernetes 集群
- 日志、监控和告警

常见平台包括阿里云、腾讯云、华为云、AWS 和 Google Cloud。

两者的关系可以简单理解为：

> 云平台提供机器和基础设施，Kubernetes 负责在这些机器上组织和管理应用实例。

## 7. 阿里云 Kubernetes 管理界面

阿里云的托管 Kubernetes 产品叫“容器服务 Kubernetes 版（ACK）”。可以登录[阿里云控制台](https://home.console.aliyun.com/)，搜索“容器服务 Kubernetes 版”或“ACK”，然后选择目标集群。

常见资源结构：

```text
集群
├── 工作负载
│   ├── Deployment
│   └── Pod
├── 网络
│   ├── Service
│   └── Ingress
├── 配置
│   ├── ConfigMap
│   └── Secret
├── 存储
└── 日志与监控
```

排查一个服务时，可以按照以下链路查看：

```text
Ingress -> Service -> Deployment -> Pod
```

- `Ingress`：域名和路径如何进入集群。
- `Service`：请求会转发给哪组 Pod。
- `Deployment`：期望运行几个实例、使用哪个镜像版本。
- `Pod`：真正运行的实例。
- `Endpoints/EndpointSlice`：Service 当前关联的具体实例地址。

能否查看和操作集群，取决于阿里云 RAM 权限以及 Kubernetes RBAC 权限。

## 8. 一个完整示例

假设有三台服务器组成 Kubernetes 集群，订单服务运行三个 Pod：

```text
阿里云负载均衡
  |
  v
Ingress
  |
  v
Service：order-service:8080
  |
  +-> Pod A：运行在服务器 1
  +-> Pod B：运行在服务器 2
  +-> Pod C：运行在服务器 3
```

如果 Pod B 崩溃：

1. Service 暂时停止向 Pod B 转发请求。
2. Kubernetes 创建新的 Pod。
3. 新 Pod 健康检查通过后加入 Service。
4. 调用方始终访问 `order-service:8080`，不需要感知 Pod IP 的变化。

