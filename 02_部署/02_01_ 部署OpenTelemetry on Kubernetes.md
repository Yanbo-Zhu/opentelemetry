

在 Kubernetes 上使用 OpenTelemetry，主要就是部署 OpenTelemetry 收集器。我们建议使用 OpenTelemetry Operator 来部署，因为它可以帮助我们轻松部署和管理 OpenTelemetry 收集器，还可以自动检测应用程序。

# 1 **部署**

这里我们使用 Helm Chart 来部署 OpenTelemetry Operator，首先添加 Helm Chart 仓库：
```javascript
$ helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
$ helm repo update
```

----

默认情况下会部署一个准入控制器，用于验证 OpenTelemetry Operator 的配置是否正确，为了使 APIServer 能够与 Webhook 组件进行通信，Webhook 需要一个由 APIServer 配置为可信任的 TLS 证书。

为了简单我们这里直接使用自动生成签名证书的方式，使用下面的命令一键安装 OpenTelemetry Operator：

```javascript
$ helm upgrade --install --set admissionWebhooks.certManager.enabled=false --set admissionWebhooks.certManager.autoGenerateCert=true opentelemetry-operator open-telemetry/opentelemetry-operator --namespace kube-otel --create-namespace
```

- **默认情况下会部署一个准入控制器**
    - 在 Kubernetes 里，**Admission Controller（准入控制器）** 是在对象被持久化到 etcd 前，对 API 请求进行**拦截和检查/修改**的插件。
    - OpenTelemetry Operator 安装时，会默认部署一个 Admission Webhook（属于准入控制器的一种实现）。
- **用于验证 OpenTelemetry Operator 的配置是否正确**
    - 当你创建/修改 OpenTelemetry 相关的自定义资源（CR，例如 `OpenTelemetryCollector`），这个 Admission Webhook 会拦截请求并检查：
        - 配置是否合法（比如字段拼写、格式是否正确）
        - 是否符合 OpenTelemetry Operator 的要求
    - 这样避免错误的配置直接进入集群，引发 Collector 部署失败。
- **为了使 APIServer 能够与 Webhook 组件进行通信**
    - Admission Webhook 是一个在集群中运行的服务（通常是一个 Pod + Service）。
    - 当 API Server 收到用户请求时，会调用这个 Webhook 服务去做验证。
- **Webhook 需要一个由 APIServer 配置为可信任的 TLS 证书**
    - 因为 API Server 和 Webhook 之间是 **HTTPS 通信**。
    - 所以 Webhook 服务必须有一个 **TLS 证书**，而且这个证书必须被 API Server 信任。
    - 否则，API Server 会拒绝和 Webhook 通信，Admission Controller 就无法工作。
    - 通常这个证书由 `cert-manager` 或 Operator 自己生成，并自动注入到 Webhook 配置中。

![](image/Pasted%20image%2020250821174243.png)


在 **Kubernetes** 中，**APIServer**（全称 _kube-apiserver_）是整个集群的 **核心组件**之一，可以理解为 **Kubernetes 的大脑和中枢**。


---------

正常部署完成后可以看到对应的 Pod 已经正常运行：

```
$ kubectl get pods -n kube-otel -l app.kubernetes.io/name=opentelemetry-operator
NAME                                      READY   STATUS    RESTARTS   AGE
opentelemetry-operator-6f77dc895c-4wn8z   2/2     Running   0          33s

```

