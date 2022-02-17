# pod与客户端之间的通信(Service)

## Service(服务)

pod的IP地址是多变的，服务就是为了解决不断变化的podIP地址的问题。k8s的服务是一种为一组功能相同的pod提供单一不变的接入点的资源。服务的地址是静态的，客户端通过固定IP连接到服务，而不是直连pod。当应用有多个pod在运行时，请求会随机切换到不同的pod，服务作为负载均衡挡在多个pod前面。

### 创建服务

可以使用expose命令或者通过配置yaml文件手动创建服务

yaml文件创建服务示例：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  sessionAffinity: ClientIP
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https # defined in pod template.spec.containers.ports array
  selector:
    app: kubia
```

会话亲和性：sessionAffinity：none or ClientIP。ClientIP会把来自同一个clientIP的所有请求转发到同一个pod。

### 服务发现

客户端和 Pod 都需知道服务本身的 IP 和 Port，才能与其背后的 Pod 进行交互。

- 环境变量：`kubectl exec servicename env` 会看到 Pod 的环境变量列出了 Pod 创建时的所有服务地址和端口，如 SVCNAME_SERVICE_HOST 和 SVCNAME_SERVICE_PORT 指向服务。
- DNS 发现：Pod 上通过全限定域名 FQDN 访问服务：`<service_name>.<namespace>.svc.cluster.local`
    
    ```bash
    > kubectl exec kubia-qgtmw cat /etc/resolv.conf
    nameserver 10.96.0.10
    search default.svc.cluster.local svc.cluster.local cluster.local # 会
    options ndots:5
    ```
    
    在 Pod kubia-qgtmw 中可通过访问 `kubia.default.svc.cluster.local` 来访问 kubia 服务，在 `/etc/resolv.conf` 中指明了域名解析服务器地址，以及主机名补全规则，是在 Pod 创建时候，根据 namespace 手动导入的。
    

## Service 对内部解析外部

服务并不是和pod直接相连的。Endpoint资源就是暴露一个服务的IP地址和端口的列表。

## Service 对外部解析内部

### NodePort

使用：外部客户端直连宿主机端口访问服务。

原理：在集群所有节点暴露指定的端口给外部客户端，该端口会将请求转发 Service 进一步转发给节点上符合 label 的 Pod，即 Service 从所有节点收集指定端口的请求并分发给能处理的 Pod

缺点：高可用性需由外部客户端保证，若节点下线需及时切换。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30001
  selector:
    app: kubia
```

三个端口号，节点转发 30001，Service 转发 80：

### LoadBalancer

场景：外部客户端直连 LB 访问服务。其是 K8S 集群端高可用的 NameNode 扩展。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: kubia
```

k8s `app:kubia` 所在的所有节点打开随机端口，进一步转发给 Pod 的 8080 端口。

## Ingress

顶级转发代理资源，仅通过一个 IP 即可代理转发后端多个 Service 的请求。需要开启 nginx controller 功能

### 创建Ingress

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    - host: "kubia.example.com"
      http:
        paths:
          - path: /kubia # 将 /kubia 子路径请求转发到 kubia-nodeport 服务的 80 端口
            backend:
              serviceName: kubia-nodeport
              servicePort: 80
          - path: /user # 可配置多个 path 对应到 service
            backend:
              serviceName: user-svc
              servicePort: 90
    - host: "new.example.com" # 可配置多个 host
      http:
        paths:
          - path: /
            backend:
              serviceName: gooele
              servicePort: 8080
```

### 访问服务

可以直接通过rules中的host来访问服务

### 暴露服务

rules和paths都是数组，同一个Ingress可以暴露多个服务。

### 处理TLS传输

通过配置秘钥和证书以便Ingress接收HTTPS请求。

## 就绪探针

### 探针简介

就绪探测器会定期调用，并确定特定的pod是否接收客户端请求。当容器的准备就绪探测返回成功，表示容器已准备好接收请求。有Exec探针、HTTP GET探针、TCP socket探针三种探针。

### 给pod添加探针

```yaml
spec:
      containers:
        - name: kubia-container
          image: yinzige/kubia
          readinessProbe:
            exec:
              command:
                - ls
                - /var/ready
```

### 探针作用

pod 启动后并非立刻就绪，需延迟接收来自 service 的请求。若不定义就绪探针，pod 启动就会暴露给 service 使用，所以需像存活探针一样添加指定类型的就绪探针。应用程序是否可以接收客户端请求。决定了就绪探测应该返回成功或失败。

## Headless

通过创建headless服务可以让DNS发现podIP。