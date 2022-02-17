# k8s副本机制和其他控制器

## 副本机制及其他控制器

在实际用例中，几乎不会直接去创建pod，而是通过创建ReplicationController或者Deployment这样的资源，再由它们去创建pod。

## 容器存活探针（Liveness Probe）

k8s可以通过存活探针来检查容器是否还在运行。可以为每个容器单独指定存活探针，如果探测失败，k8s将定期执行探针并重启容器。

k8s有三种探测容器的机制：

- HTTP GET探针：对容器的IP地址执行HTTP GET 请求，探测器收到响应且状态码不为错误码，则探测成功。
- TCP套接字探针：尝试和容器的指定端口建立TCP连接，连接成功建立，则探测成功。
- Exec探针：在容器内执行任意命令并检查命令的退出状态码。如果是0，则探测成功。

### HTTP GET探针示例：

pod yaml文件：

```bash
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```

### 使用存活探针

通过describe命令查看pod

```bash
$ kubectl.exe describe po kubia-liveness
Name:         kubia-liveness
...
Containers:
  kubia:
    Container ID:   docker://0813304798d8665026dad59e2294202063f96f5a30b1632ace555df5881b68ee
    Image:          luksa/kubia-unhealthy
    Image ID:       docker-pullable://luksa/kubia-unhealthy@sha256:5c746a42612be61209417d913030d97555cff0b8225092908c57634ad7c235f7
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 14 Jul 2021 10:22:45 +0800
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Wed, 14 Jul 2021 10:20:49 +0800
      Finished:     Wed, 14 Jul 2021 10:22:35 +0800
    Ready:          True
    Restart Count:  2
    Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
...
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  10m                    default-scheduler  Successfully assigned default/kubia-liveness to 172.18.100.163
  Normal   Pulled     8m35s                  kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 2m12.263447546s
  Normal   Pulled     6m48s                  kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 3.892932848s
  Normal   Created    4m52s (x3 over 8m34s)  kubelet            Created container kubia
  Normal   Started    4m52s (x3 over 8m34s)  kubelet            Started container kubia
  Normal   Pulled     4m52s                  kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 9.818354951s
  Warning  Unhealthy  3m32s (x9 over 7m42s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    3m32s (x3 over 7m22s)  kubelet            Container kubia failed liveness probe, will be restarted
  Normal   Pulling    3m2s (x4 over 10m)     kubelet            Pulling image "luksa/kubia-unhealthy"
```

可以看到该pod已被重启2次。之前由于错误被终止，退出码137(128+x)，这个x是终止进程的信号编号-9，意味着这个容器被强行终止。k8s发现容器不健康，所以终止容器并重新创建一个容器。

```bash
Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
```

delay 延迟，timeout 超时，period 周期，success 连续探测成功次数，failure 失败次数重启容器。

可以通过explain命令查看存活探针存在哪些属性：

```bash
$ kubectl.exe explain pod.spec.containers.livenessProbe
KIND:     Pod
VERSION:  v1

RESOURCE: livenessProbe <Object>

DESCRIPTION:
     Periodic probe of container liveness. Container will be restarted if the
     probe fails. Cannot be updated. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes

     Probe describes a health check to be performed against a container to
     determine whether it is alive or ready to receive traffic.

FIELDS:
   exec <Object>
     One and only one of the following should be specified. Exec specifies the
     action to take.

   failureThreshold     <integer>
     Minimum consecutive failures for the probe to be considered failed after
     having succeeded. Defaults to 3. Minimum value is 1.

   httpGet      <Object>
     HTTPGet specifies the http request to perform.

   initialDelaySeconds  <integer>
     Number of seconds after the container has started before liveness probes
     are initiated. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes

   periodSeconds        <integer>
     How often (in seconds) to perform the probe. Default to 10 seconds. Minimum
     value is 1.

   successThreshold     <integer>
     Minimum consecutive successes for the probe to be considered successful
     after having failed. Defaults to 1. Must be 1 for liveness and startup.
     Minimum value is 1.

   tcpSocket    <Object>
     TCPSocket specifies an action involving a TCP port. TCP hooks not yet
     supported

   timeoutSeconds       <integer>
     Number of seconds after which the probe times out. Defaults to 1 second.
     Minimum value is 1. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes
```

### 存活探针总结

我们可以将探针配置为请求特定的URL路径，并从应用内部对组件进行检查，确保它们没有终止或者停止响应。探针不应消耗太多的计算资源且运行不应该花太长时间。存活探针失败时k8s通过承载pod的节点上的kubelet执行重启容器来保持运行。但是如果节点本身崩溃了，那它将无法执行任何操作。

## ReplicationController(RC)

ReplicationController是一种k8s的资源，可确保 它的pod始终保持运行状态，保证相应标签的pod的数目与期望相符。

ReplicationController的三个主要部分：

- label Selector 标签选择器，用于确定RC作用域中有哪些pod
- replica count 副本个数，指应运行的pod数量
- pod template pod模板，用于创建新的pod副本

RC 的两个功能：

- 监控：确保符合标签选择器的 Pod 以指定的副本数量运行，多了则删除，少了则按 Pod 模板创建。
- 扩缩容：能对监控的某组 Pod 进行动态修改副本数量进行扩缩容。

示例：

kubia-rc.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 2
  #selector:
    #app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```

列出pod

```bash
$ kubectl.exe get pods -l app=kubia
NAME          READY   STATUS             RESTARTS   AGE
kubia-k698x   0/1     ImagePullBackOff   0          7m51s
kubia-ttz4d   1/1     Running            0          7m51s
```

手动删除其中一个pod

```bash
$ kubectl.exe delete pod kubia-k698x
pod "kubia-k698x" deleted

$ kubectl.exe get pods -l app=kubia
NAME          READY   STATUS        RESTARTS   AGE
kubia-k698x   0/1     Terminating   0          10m
kubia-ttz4d   1/1     Running       0          10m
kubia-zxq2b   1/1     Running       0          37s
```

RC通过标签匹配来管理pod，如果你修改了pod的标签，该pod也就不再受RC的管理，RC会启动一个新的pod来替换它。当你更改RC的标签选择器时，所有的pod都会脱离RC的管理，RC会重新创建若干个新的pod。但是一般不会去直接修改RC的标签选择器。

RC中的pod模板可以随时修改，但是修改pod模板只会影响修改后创建的pod，并不会对现有的pod有所修改。

放大或缩小pod数量示例：

```bash
$ kubectl.exe edit rc kubia
....
apiVersion: v1
kind: ReplicationController
...
spec:
  replicas: 4 #修改为4
...
replicationcontroller/kubia edited

$ kubectl.exe get pods -l app=kubia
NAME          READY   STATUS              RESTARTS   AGE
kubia-ttz4d   1/1     Running             0          24m
kubia-vvprg   0/1     ContainerCreating   0          16s
kubia-wcn2z   1/1     Running             0          16s
kubia-zxq2b   1/1     Running             0          14m
```

所以在k8s中放大或缩小pod是陈述式的，你不是告诉k8s做什么或如何去做，你只需要指定你期望的状态即可。

删除一个RC的同时默认pod也会被删除。使用—cascade=false命令可以只删除RC，该RC管理的pod将不再被管理，你也可以再次创建相关标签选择器的RC来再次管理这些pod。 

```bash
$ kubectl.exe delete rc kubia
replicationcontroller "kubia" deleted
$ kubectl.exe get pods -l app=kubia
NAME          READY   STATUS        RESTARTS   AGE
kubia-ttz4d   1/1     Terminating   0          29m
kubia-vvprg   1/1     Terminating   0          5m15s
kubia-wcn2z   1/1     Terminating   0          5m15s
kubia-zxq2b   1/1     Terminating   0          19m
```

## ReplicaSet(RS)

ReplicaSet = ReplicationController + 扩展的 label selector ，即对 pod 的 label selector 表达力更强 。

RS 能通过 `selector.matchLabels` 和 `selector.matchExpressio` 来扩展对 pod label 的筛选：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels: # 与 RC 一样必须完整匹配
      app: kubia
    matchExpressions: # 更富表达力的标签选择器
      - key: app
        operator: In # 必须有 KEY 且 VALUE 在列表中
        values:
          - kubia
          - kubia-v2
      - key: app
        operator: NotIn # 有 KEY 则不能在如下列表中
        values:
          - KUBIA
      - key: env # 必须存在的 KEY，不能有 VALUE
        operator: Exists
      - key: ENV
        operator: DoesNotExist # 必须不能存在的 KEY，也不能有 VALUE
  template:
    metadata:
      labels: # Pod 模板的 label 必须能和 RS 的 selector 匹配上
        app: kubia
        env: env_exists
    spec:
      containers:
        - name: kubia
          image: luksa/kubia
          ports:
            - protocol: TCP
              containerPort: 8080
```

operator运算符：

- In 标签的值必须和其中一个指定的values匹配
- NotIn 标签的值和任何指定的values不匹配
- Exists pod必须包含一个指定名称的标签
- DoesNotExist pod必须不包含指定名称的标签

存在多个表达式时，它们取得是&&，所有表达式必须都为true，该pod才能和选择器匹配。

## DaemonSet

- 功能：保证标签匹配的 Pod 在符合nodeSelector 的节点上运行一个pod，没有目标 pod 数量的概念，无法 scale
- 场景：部署系统级组件，如 Node 监控，如 kube-proxy 处理各节点网络代理等。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor # 指定要控制运行的一组 Pod
  template:
    metadata:
      labels:
        app: ssd-monitor # 被控制的 Pod 的标签
    spec:
      nodeSelector: # 选择 Pod 要运行的节点标签
        disk: ssd # 注意 YAML 文件的 true 类型是布尔型，如 ssd: true 是无法被解析为 String 的
      containers:
        - name: main
          image: luksa/ssd-monitor
```

## Job

- 功能：保证任务以指定的并发数执行指定次数，任务执行失败后按配置策略处理。
- 场景：执行一次性任务。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  completions: 5 # 总任务数量
  parallelism: 2 # 并发执行任务数
  template:
    metadata:
      labels:
        app: batch-job # 要执行的 pod job label
    spec:
      restartPolicy: OnFailure # 任务异常结束或节点异常时处理方式："Always", "OnFailure", "Never"
      containers:
        - name: main
          image: luksa/batch-job
```

## CronJob

对标 Linux 的 crontab 的定时任务。在特定的时间运行或者在指定时间间隔内重复运行的任务。schedule使用cron时间表格式。

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cron-batch-job
spec:
  schedule: "*/1 * * * *" # 每分钟运行一次
	startingDeadlineSeconds: 15 # pod最迟在预定时间后15S开始运行
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: cron-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cron-batch-job
              image: luksa/batch-job
```

在计划的时间内，CronJob会创建Job资源，然后Job创建pod。