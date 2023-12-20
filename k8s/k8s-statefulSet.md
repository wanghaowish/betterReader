# StatefulSet

# StatefulSet的基本介绍

## 作用

StatefulSet 对于需要满足以下一个或多个需求的应用程序很有价值：

- 稳定的、唯一的网络标识符。
- 稳定的、持久的存储。
- 有序的、优雅的部署和扩缩。
- 有序的、自动的滚动更新。

statefulSet主要用于创建有状态的pod。保证pod在重新调度后保留它们的名称、标识和状态。使用场景：在有状态分部署存储应用中，Pod 的多副本有各自独立的 PVC 和 PV，pod 在新节点重建后需保证状态一致。

## 创建

statefulSet.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia
  replicas: 2
  selector:
    matchLabels:
      app: kubia # has to match .spec.template.metadata.labels
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
      storageClassName: "my-storage-class"
```

对于一个拥有 N 个副本的 statefulset，pod 是按照 {0..N-1}的序号顺序创建的，并且会等待前一个 pod 变为 Running & Ready 后才会启动下一个 pod。

## 扩缩容

statefulset 扩容时 pod 也是顺序创建的，编号与前面的 pod 相接。

缩容时控制器会按照与 pod 序号索引相反的顺序删除pod，在删除下一个 pod 前会等待上一个被完全删除。

 `kubectl scale statefulset statefulset --replicas=X` or ` kubectl apply -f statefulset.yaml`

## 更新

k8s资源三大重要部分：

- metadata包括名称、命名控件、标签及关于该容器的其他信息
- spec包含pod内容的实际说明
- status包含运行中的pod的当前信息

kubectl explain可用来发现可能的API对象字段。可查看每个API对象支持哪些属性

更新策略由 statefulset 中的 spec.updateStrategy.type 字段决定，可以指定为 OnDelete 或者 RollingUpdate , 默认的更新策略为 RollingUpdate。

```bash
$ kubectl.exe explain sts.spec.updateStrategy
KIND:     StatefulSet
VERSION:  apps/v1

RESOURCE: updateStrategy <Object>

DESCRIPTION:
     updateStrategy indicates the StatefulSetUpdateStrategy that will be
     employed to update Pods in the StatefulSet when a revision is made to
     Template.

     StatefulSetUpdateStrategy indicates the strategy that the StatefulSet
     controller will use to perform updates. It includes any additional
     parameters necessary to perform the update for the indicated strategy.

FIELDS:
   rollingUpdate        <Object>
     RollingUpdate is used to communicate parameters when Type is
     RollingUpdateStatefulSetStrategyType.

   type <string>
     Type indicates the type of the StatefulSetUpdateStrategy. Default is
     RollingUpdate.
```

### RollingUpdate

当使用RollingUpdate 更新策略更新所有 pod 时采用与序号索引相反的顺序进行更新，即最先删除序号最大的 pod 并根据更新策略中的 partition 参数来进行分段更新，控制器会更新所有序号大于或等于 partition 的 pod，等该区间内的 pod 更新完成后需要再次设定 partition 的值以此来更新剩余的 pod，最终 partition 被设置为 0 时代表更新完成了所有的 pod。在更新过程中，如果一个序号小于 partition 的 pod 被删除或者终止，controller 依然会使用更新前的配置重新创建。

### OnDelete

如果 statefulset 的 .spec.updateStrategy.type 字段被设置为 OnDelete，在更新 statefulset 时，statefulset controller 将不会自动更新其 pod。你必须手动删除 pod，此时 statefulset controller 在重新创建 pod 时，使用修改过的 spec.template 的内容创建新 pod。

使用滚动更新策略时你必须以某种策略不断更新 partition 值来进行升级，类似于金丝雀部署方式，升级对于 pod 名称来说是逆序。使用非滚动更新方式时，需要手动删除对应的 pod，升级可以是无序的。

## 回滚

statefulSet的回滚操作其实也是进行了一次发布更新。和发布更新的策略一样，更新 statefulset 后需要按照对应的策略手动删除 pod 或者修改 partition 字段以达到回滚 pod 的目的。但是要注意的是statefulSet对象回滚很简单，它使用的pv中保存的数据无法回滚。

![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202312201050537.png)

## 删除

statefulset 同时支持级联和非级联删除。使用非级联方式删除 statefulset 时，statefulset 的 pod 不会被删除。使用级联删除时，statefulset 和它关联的 pod 都会被删除。对于级联与非级联删除，在删除时需要指定删除选项(orphan、background 或者 foreground)进行区分。

kubernetes 中有三种删除策略：`Orphan`、`Foreground` 和 `Background`，三种删除策略的意义分别为：

- `Orphan` 策略：非级联删除，删除对象时，不会自动删除它的依赖或者是子对象，这些依赖被称作是原对象的孤儿对象，例如当执行以下命令时会使用 `Orphan` 策略进行删除，此时 ds 的依赖对象 `controllerrevision` 不会被删除；

```bash
$ kubectl delete sts kubia --cascade=false
```

- `Background` 策略：在该模式下，kubernetes 会立即删除该对象，然后垃圾收集器会在后台删除这些该对象的依赖对象；
- `Foreground` 策略：在该模式下，对象首先进入“删除中”状态，即会设置对象的 `deletionTimestamp` 字段并且对象的 `metadata.finalizers` 字段包含了值 “foregroundDeletion”，此时该对象依然存在，然后垃圾收集器会删除该对象的所有依赖对象，垃圾收集器在删除了所有“Blocking” 状态的依赖对象（指其子对象中 `ownerReference.blockOwnerDeletion=true`的对象）之后，然后才会删除对象本身；

# StatefulSet源码分析

startStatefulSetController 是 statefulSetController 的启动方法，其中调用 NewStatefulSetController 进行初始化 controller 对象然后调用 Run 方法启动 controller。其中 ConcurrentStatefulSetSyncs 默认值为 5。

k8s.io/kubernetes/cmd/kube-controller-manager/app/apps.go:51

```go
func startStatefulSetController(ctx ControllerContext) (http.Handler, bool, error) {
	go statefulset.NewStatefulSetController(
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Apps().V1().StatefulSets(),
		ctx.InformerFactory.Core().V1().PersistentVolumeClaims(),
		ctx.InformerFactory.Apps().V1().ControllerRevisions(),
		ctx.ClientBuilder.ClientOrDie("statefulset-controller"),
	).Run(int(ctx.ComponentConfig.StatefulSetController.ConcurrentStatefulSetSyncs), ctx.Stop)
	return nil, true, nil
}
```

当 controller 启动后会通过 informer 同步 cache 并监听 pod 和 statefulset 对象的变更事件，最后会执行 sync 方法.

## informer

Informer 是 Client-go 中的一个核心工具包。在 Kubernetes 源码中，如果 Kubernetes 的某个组件，需要 List/Get Kubernetes 中的Object，在绝大多数情况下，会直接使用 Informer 实例中的 Lister()方法（该方法包含 了 Get 和 List 方法）。Informer 最基本 的功能就是 List/Get Kubernetes 中的 Object。除 List/Get Object 外，Informer 还可以监听事件并触发回调函数等，以实现更加复杂的业务逻辑。

### 更快地返回 List/Get 请求，减少对 Kubenetes API 的直接调用

使用 Informer 实例的 Lister() 方法， List/Get Kubernetes 中的 Object 时，Informer 不会去请求 Kubernetes API，而是直接查找缓存在本地内存中的数据(这份数据由 Informer 自己维护)。通过这种方式，Informer 既可以更快地返回结果，又能减少对 Kubernetes API 的直接调用。

### 依赖 Kubernetes List/Watch API

Informer 在初始化的时，先调用 Kubernetes List API 获得某种 resource 的全部 Object，缓存在内存中; 然后，调用 Watch API 去 watch 这种 resource，去维护这份缓存; 最后，Informer 就不再调用 Kubernetes 的任何 API。注意：Informer 只在初始化时调用一次 List API，之后完全依赖 Watch API 去维护缓存。

### 可监听事件并触发回调函数

Informer 通过 Kubernetes Watch API 监听某种 resource 下的所有事件。而且，Informer 可以添加自定义的回调函数，这个回调函数实例(即 ResourceEventHandler 实例)只需实现 OnAdd(obj interface{}) OnUpdate(oldObj, newObj interface{}) 和 OnDelete(obj interface{}) 三个方法，这三个方法分别对应 informer 监听到创建、更新和删除这三种事件类型。

### **二级缓存**

二级缓存属于 Informer 的底层缓存机制，这两级缓存分别是 DeltaFIFO 和 LocalStore。

这两级缓存的用途各不相同。DeltaFIFO 用来存储 Watch API 返回的各种事件 ，LocalStore 只会被 Lister 的 List/Get 方法访问 。

### 关键逻辑解析：pod示例

1. Informer 在初始化时，Reflector 会先 List API 获得所有的 Pod

2. Reflect 拿到全部 Pod后，会将全部 Pod 放到 Store 中

3. 如果有人调用 Lister 的 List/Get 方法获取 Pod， 那么 Lister 会直接从 Store 中拿数据

   ![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/v2-61b6a728a7c18b608aa77b40d02f3042_720w.jpg)

4. Informer 初始化完成之后，Reflector 开始 Watch Pod，监听 Pod 相关 的所有事件;如果此时 pod_1 被删除，那么 Reflector 会监听到这个事件

5. Reflector 将 pod_1 被删除 的这个事件发送到 DeltaFIFO

6. DeltaFIFO 首先会将这个事件存储在自己的数据结构中(实际上是一个 queue)，然后会直接操作 Store 中的数据，删除 Store 中的 pod_1

7. DeltaFIFO 再 Pop 这个事件到 Controller 中

   ![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/v2-a1a23b69f7fae636ca0cf72c80ad64f7_720w.jpg)

8. Controller 收到这个事件，会触发 Processor 的回调函数

   ![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/v2-b37f5ccf2fb67dfa58f6f04ce76b53a7_720w.jpg)

9. LocalStore 会周期性地把所有的 Pod 信息重新放到 DeltaFIFO 中

## sync方法

1. 根据 ns/name 获取 sts 对象；
2. 获取 sts 的 selector；
3. 调用 `ssc.adoptOrphanRevisions` 检查是否有孤儿 `controllerrevisions` 对象，若有且能匹配 selector 的对象则添加 ownerReferences 进行关联，已关联但 label 不匹配的则进行释放；
4. 调用 `ssc.getPodsForStatefulSet` 通过 selector 获取 sts 关联的 pod，若有孤儿 pod 的 label 与 sts 的能匹配则进行关联，若已关联的 pod label 有变化则解除与 sts 的关联关系；
5. 最后调用 `ssc.syncStatefulSet` 执行真正的 sync 操作

k8s.io/kubernetes/pkg/controller/statefulset/stateful_set.go:428

```go
// sync syncs the given statefulset.
func (ssc *StatefulSetController) sync(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing statefulset %q (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	// 获取 sts 对象
	set, err := ssc.setLister.StatefulSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.Infof("StatefulSet has been deleted %v", key)
		return nil
	}
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to retrieve StatefulSet %v from store: %v", key, err))
		return err
	}

	selector, err := metav1.LabelSelectorAsSelector(set.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("error converting StatefulSet %v selector: %v", key, err))
		// This is a non-transient error, so don't retry.
		return nil
	}
	// 关联以及释放 sts 的 controllerrevisions
	if err := ssc.adoptOrphanRevisions(set); err != nil {
		return err
	}
	// 获取 sts 所关联的 pod
	pods, err := ssc.getPodsForStatefulSet(set, selector)
	if err != nil {
		return err
	}

	return ssc.syncStatefulSet(set, pods)
}
```

k8s.io/kubernetes/pkg/controller/statefulset/stateful_set.go:323

```go
//关联以及释放 sts 的 controllerrevisions
// adoptOrphanRevisions adopts any orphaned ControllerRevisions matched by set's Selector.
func (ssc *StatefulSetController) adoptOrphanRevisions(set *apps.StatefulSet) error {
	revisions, err := ssc.control.ListRevisions(set)
	if err != nil {
		return err
	}
	orphanRevisions := make([]*apps.ControllerRevision, 0)
	for i := range revisions {
		if metav1.GetControllerOf(revisions[i]) == nil {
			orphanRevisions = append(orphanRevisions, revisions[i])
		}
	}
	if len(orphanRevisions) > 0 {
		canAdoptErr := ssc.canAdoptFunc(set)()
		if canAdoptErr != nil {
			return fmt.Errorf("can't adopt ControllerRevisions: %v", canAdoptErr)
		}
		return ssc.control.AdoptOrphanRevisions(set, orphanRevisions)
	}
	return nil
}
```

k8s.io/kubernetes/pkg/controller/statefulset/stateful_set.go:290

```go
// 获取 sts 所关联的 pod
// getPodsForStatefulSet returns the Pods that a given StatefulSet should manage.
// It also reconciles ControllerRef by adopting/orphaning.
//
// NOTE: Returned Pods are pointers to objects from the cache.
//       If you need to modify one, you need to copy it first.
func (ssc *StatefulSetController) getPodsForStatefulSet(set *apps.StatefulSet, selector labels.Selector) ([]*v1.Pod, error) {
	// List all pods to include the pods that don't match the selector anymore but
	// has a ControllerRef pointing to this StatefulSet.
	pods, err := ssc.podLister.Pods(set.Namespace).List(labels.Everything())
	if err != nil {
		return nil, err
	}

	filter := func(pod *v1.Pod) bool {
		// Only claim if it matches our StatefulSet name. Otherwise release/ignore.
		return isMemberOf(set, pod)
	}

	cm := controller.NewPodControllerRefManager(ssc.podControl, set, selector, controllerKind, ssc.canAdoptFunc(set))
	return cm.ClaimPods(pods, filter)
}
```

## syncStatefulSet方法

在 syncStatefulSet 中仅仅是调用了 ssc.control.UpdateStatefulSet 方法进行处理。ssc.control.UpdateStatefulSet 会调用 defaultStatefulSetControl 的 UpdateStatefulSet 方法，defaultStatefulSetControl 是 statefulset controller 中另外一个对象，主要负责处理 statefulset 的更新。

k8s.io/kubernetes/pkg/controller/statefulset/stateful_set.go:468

```go
// syncStatefulSet syncs a tuple of (statefulset, []*v1.Pod).
func (ssc *StatefulSetController) syncStatefulSet(set *apps.StatefulSet, pods []*v1.Pod) error {
	klog.V(4).Infof("Syncing StatefulSet %v/%v with %d pods", set.Namespace, set.Name, len(pods))
	var status *apps.StatefulSetStatus
	var err error
	// TODO: investigate where we mutate the set during the update as it is not obvious.
	status, err = ssc.control.UpdateStatefulSet(set.DeepCopy(), pods)
	if err != nil {
		return err
	}
	klog.V(4).Infof("Successfully synced StatefulSet %s/%s successful", set.Namespace, set.Name)
	// One more sync to handle the clock skew. This is also helping in requeuing right after status update
	if utilfeature.DefaultFeatureGate.Enabled(features.StatefulSetMinReadySeconds) && set.Spec.MinReadySeconds > 0 && status != nil && status.AvailableReplicas != *set.Spec.Replicas {
		ssc.enqueueSSAfter(set, time.Duration(set.Spec.MinReadySeconds)*time.Second)
	}

	return nil
}

```

### UpdateStatefulSet方法

`UpdateStatefulSet` 方法的主要逻辑如下所示：

1. 获取历史 revisions；
2. 计算 `currentRevision` 和 `updateRevision`，若 sts 处于更新过程中则 `currentRevision` 和 `updateRevision` 值不同；
3. 调用 `ssc.updateStatefulSet` 执行实际的 sync 操作；
4. 调用 `ssc.updateStatefulSetStatus` 更新 status subResource；
5. 根据 sts 的 `spec.revisionHistoryLimit`字段清理过期的 `controllerrevision`；

在基本操作的回滚阶段提到过sts 通过 `controllerrevision` 保存历史版本，在 sts 的滚动升级过程中是通过 `currentRevision` 和 `updateRevision`来进行控制并不会用到 `controllerrevision`。

k8s.io/kubernetes/pkg/controller/statefulset/stateful_set_control.go:76

```go
func (ssc *defaultStatefulSetControl) UpdateStatefulSet(set *apps.StatefulSet, pods []*v1.Pod) (*apps.StatefulSetStatus, error) {
	// 获取历史 revisions
	// list all revisions and sort them
	revisions, err := ssc.ListRevisions(set)
	if err != nil {
		return nil, err
	}
	history.SortControllerRevisions(revisions)

	
	currentRevision, updateRevision, status, err := ssc.performUpdate(set, pods, revisions)
	if err != nil {
		return nil, utilerrors.NewAggregate([]error{err, ssc.truncateHistory(set, pods, revisions, currentRevision, updateRevision)})
	}
	
// 清理过期的历史版本
	// maintain the set's revision history limit
	return status, ssc.truncateHistory(set, pods, revisions, currentRevision, updateRevision)
}

func (ssc *defaultStatefulSetControl) performUpdate(
	set *apps.StatefulSet, pods []*v1.Pod, revisions []*apps.ControllerRevision) (*apps.ControllerRevision, *apps.ControllerRevision, *apps.StatefulSetStatus, error) {
	var currentStatus *apps.StatefulSetStatus
	// get the current, and update revisions
	// 计算 currentRevision 和 updateRevision
	currentRevision, updateRevision, collisionCount, err := ssc.getStatefulSetRevisions(set, revisions)
	if err != nil {
		return currentRevision, updateRevision, currentStatus, err
	}
	
	// 执行实际的 sync 操作
	// perform the main update function and get the status
	currentStatus, err = ssc.updateStatefulSet(set, currentRevision, updateRevision, collisionCount, pods)
	if err != nil {
		return currentRevision, updateRevision, currentStatus, err
	}

	// 更新 sts 状态
	// update the set's status
	err = ssc.updateStatefulSetStatus(set, currentStatus)
	if err != nil {
		return currentRevision, updateRevision, currentStatus, err
	}
	klog.V(4).Infof("StatefulSet %s/%s pod status replicas=%d ready=%d current=%d updated=%d",
		set.Namespace,
		set.Name,
		currentStatus.Replicas,
		currentStatus.ReadyReplicas,
		currentStatus.CurrentReplicas,
		currentStatus.UpdatedReplicas)

	klog.V(4).Infof("StatefulSet %s/%s revisions current=%s update=%s",
		set.Namespace,
		set.Name,
		currentStatus.CurrentRevision,
		currentStatus.UpdateRevision)

	return currentRevision, updateRevision, currentStatus, nil
}
```

## updateStatefulSet方法

`updateStatefulSet` 是 sync 操作中的核心方法，对于 statefulset 的创建、扩缩容、更新、删除等操作都会在这个方法中完成。

1. 分别获取 `currentRevision` 和 `updateRevision` 对应的的 statefulset object；
2. 构建 status 对象；
3. 将 statefulset 的 pods 按 ord(ord 为 pod name 中的序号)的值分到 replicas 和 condemned 两个数组中，0 <= ord < Spec.Replicas 的放到 replicas 组，ord >= Spec.Replicas 的放到 condemned 组，replicas 组代表可用的 pod，condemned 组是需要删除的 pod；
4. 找出 replicas 和 condemned 组中的 unhealthy pod，healthy pod 指 `running & ready` 并且不处于删除状态；
5. 判断 sts 是否处于删除状态；
6. 遍历 replicas 数组，确保 replicas 数组中的容器处于 `running & ready`状态，其中处于 `failed` 状态的容器删除重建，未创建的容器则直接创建，最后检查 pod 的信息是否与 statefulset 的匹配，若不匹配则更新 pod 的状态。在此过程中每一步操作都会检查 `monotonic` 的值，即 sts 是否设置了 `Parallel` 参数，若设置了则循环处理 replicas 中的所有 pod，否则每次处理一个 pod，剩余 pod 则在下一个 syncLoop 继续进行处理；
7. 按 pod 名称逆序删除 `condemned` 数组中的 pod，删除前也要确保 pod 处于 `running & ready`状态，在此过程中也会检查 `monotonic` 的值，以此来判断是顺序删除还是在下一个 syncLoop 中继续进行处理；
8. 判断 sts 的更新策略 `.Spec.UpdateStrategy.Type`，若为 `OnDelete` 则直接返回；
9. 此时更新策略为 `RollingUpdate`，更新序号大于等于 `.Spec.UpdateStrategy.RollingUpdate.Partition` 的 pod，在 `RollingUpdate` 时，并不会关注 `monotonic` 的值，都是顺序进行处理且等待当前 pod 删除成功后才继续删除小于上一个 pod 序号的 pod，所以 `Parallel` 的策略在滚动更新时无法使用。

`updateStatefulSet` 这个方法中包含了 statefulset 的创建、删除、扩缩容、更新等操作。

- 创建：在创建 sts 后，sts 对象已被保存至 etcd 中，此时 sync 操作仅仅是创建出需要的 pod，即执行到第 6 步就会结束；
- 扩缩容：对于扩若容操作仅仅是创建或者删除对应的 pod，在操作前也会判断所有 pod 是否处于 `running & ready`状态，然后进行对应的创建/删除操作，在上面的步骤中也会执行到第 6 步就结束了；
- 更新：第6步之后的所有操作就是与更新相关的了，所以更新操作会执行完整个方法，在更新过程中通过 pod 的 `currentRevision` 和 `updateRevision` 来计算 `currentReplicas`、`updatedReplicas` 的值，最终完成所有 pod 的更新；
- 删除：删除操作会止于第五步，但是在此之前检查 pod 状态以及分组的操作确实是多余的；

k8s.io/kubernetes/pkg/controller/statefulset/stateful_set_control.go:270

```go
package main

import (
	"math"
	"sort"
)

func (ssc *defaultStatefulSetControl) updateStatefulSet(......) (*apps.StatefulSetStatus, error) {
	// 分别获取 currentRevision 和 updateRevision 对应的的 statefulset object
	currentSet, err := ApplyRevision(set, currentRevision)
	if err != nil {
		return nil, err
	}
	updateSet, err := ApplyRevision(set, updateRevision)
	if err != nil {
		return nil, err
	}

	// 计算 status
	status := apps.StatefulSetStatus{}
	status.ObservedGeneration = set.Generation
	status.CurrentRevision = currentRevision.Name
	status.UpdateRevision = updateRevision.Name
	status.CollisionCount = new(int32)
	*status.CollisionCount = collisionCount

	// 将 statefulset 的 pods 按 ord(ord 为 pod name 中的序数)的值
	// 分到 replicas 和 condemned 两个数组中
	replicaCount := int(*set.Spec.Replicas)
	replicas := make([]*v1.Pod, replicaCount)
	condemned := make([]*v1.Pod, 0, len(pods))
	unhealthy := 0
	firstUnhealthyOrdinal := math.MaxInt32

	var firstUnhealthyPod *v1.Pod

	// 计算 status 字段中的值，将 pod 分配到 replicas和condemned两个数组中
	for i := range pods {
		status.Replicas++

		if isRunningAndReady(pods[i]) {
			status.ReadyReplicas++
		}

		if isCreated(pods[i]) && !isTerminating(pods[i]) {
			if getPodRevision(pods[i]) == currentRevision.Name {
				status.CurrentReplicas++
			}
			if getPodRevision(pods[i]) == updateRevision.Name {
				status.UpdatedReplicas++
			}
		}

		if ord := getOrdinal(pods[i]); 0 <= ord && ord < replicaCount {
			replicas[ord] = pods[i]
		} else if ord >= replicaCount {
			condemned = append(condemned, pods[i])
		}
	}

	// 检查 replicas数组中 [0,set.Spec.Replicas) 下标是否有缺失的 pod，若有缺失的则创建对应的 pod object 
	// 在 newVersionedStatefulSetPod 中会判断是使用 currentSet 还是 updateSet 来创建
	for ord := 0; ord < replicaCount; ord++ {
		if replicas[ord] == nil {
			replicas[ord] = newVersionedStatefulSetPod(
				currentSet,
				updateSet,
				currentRevision.Name,
				updateRevision.Name, ord)
		}
	}

	// 对 condemned 数组进行排序
	sort.Sort(ascendingOrdinal(condemned))

	// 根据 ord 在 replicas 和 condemned 数组中找出 first unhealthy Pod 
	for i := range replicas {
		if !isHealthy(replicas[i]) {
			unhealthy++
			if ord := getOrdinal(replicas[i]); ord < firstUnhealthyOrdinal {
				firstUnhealthyOrdinal = ord
				firstUnhealthyPod = replicas[i]
			}
		}
	}

	for i := range condemned {
		if !isHealthy(condemned[i]) {
			unhealthy++
			if ord := getOrdinal(condemned[i]); ord < firstUnhealthyOrdinal {
				firstUnhealthyOrdinal = ord
				firstUnhealthyPod = condemned[i]
			}
		}
	}

	......
	
	// 判断是否处于删除中
	if set.DeletionTimestamp != nil {
		return &status, nil
	}

	// 默认设置为非并行模式
	monotonic := !allowsBurst(set)

	// 确保 replicas 数组中所有的 pod 是 running 的
	for i := range replicas {
		// 11、对于 failed 的 pod 删除并重新构建 pod object
		if isFailed(replicas[i]) {
			......
			if err := ssc.podControl.DeleteStatefulPod(set, replicas[i]); err != nil {
				return &status, err
			}
			if getPodRevision(replicas[i]) == currentRevision.Name {
				status.CurrentReplicas--
			}
			if getPodRevision(replicas[i]) == updateRevision.Name {
				status.UpdatedReplicas--
			}
			status.Replicas--
			replicas[i] = newVersionedStatefulSetPod(
				currentSet,
				updateSet,
				currentRevision.Name,
				updateRevision.Name,
				i)
		}

		// 如果 pod.Status.Phase 不为“” 说明该 pod 未创建，则直接重新创建该 pod
		if !isCreated(replicas[i]) {
			if err := ssc.podControl.CreateStatefulPod(set, replicas[i]); err != nil {
				return &status, err
			}
			status.Replicas++
			if getPodRevision(replicas[i]) == currentRevision.Name {
				status.CurrentReplicas++
			}
			if getPodRevision(replicas[i]) == updateRevision.Name {
				status.UpdatedReplicas++
			}

			// 如果为Parallel，直接return status结束；如果为OrderedReady，循环处理下一个pod。
			if monotonic {
				return &status, nil
			}
			continue
		}

		// 如果pod正在删除(pod.DeletionTimestamp不为nil)，且Spec.PodManagementPolicy不
		// 为Parallel，直接return status结束，结束后会在下一个 syncLoop 继续进行处理，
		// pod 状态的改变会触发下一次 syncLoop
		if isTerminating(replicas[i]) && monotonic {
			......
			return &status, nil
		}

		// 如果pod状态不是Running & Ready，且Spec.PodManagementPolicy不为Parallel，
		// 直接return status结束
		if !isRunningAndReady(replicas[i]) && monotonic {
			......
			return &status, nil
		}
		
		// if we are in monotonic mode and the condemned target is not the first unhealthy Pod, block.
		// TODO: Since available is superset of Ready, once we have this featuregate enabled by default, we can remove the
		// isRunningAndReady block as only Available pods should be brought down.
		if utilfeature.DefaultFeatureGate.Enabled(features.StatefulSetMinReadySeconds) && !isRunningAndAvailable(condemned[target], set.Spec.MinReadySeconds) && monotonic && condemned[target] != firstUnhealthyPod {
			......
			return &status, nil
		}

		// 检查 pod 的信息是否与 statefulset 的匹配，若不匹配则更新 pod 的状态
		if identityMatches(set, replicas[i]) && storageMatches(set, replicas[i]) {
			continue
		}

		replica := replicas[i].DeepCopy()
		if err := ssc.podControl.UpdateStatefulPod(updateSet, replica); err != nil {
			return &status, err
		}
	}

	// 逆序处理 condemned 中的 pod
	for target := len(condemned) - 1; target >= 0; target-- {

		// 如果pod正在删除，检查 Spec.PodManagementPolicy 的值，如果为Parallel，
		// 循环处理下一个pod 否则直接退出
		if isTerminating(condemned[target]) {
			......
			if monotonic {
				return &status, nil
			}
			continue
		}

		// 不满足以下条件说明该 pod 是更新前创建的，正处于创建中
		if !isRunningAndReady(condemned[target]) && monotonic && condemned[target] != firstUnhealthyPod {
			......
			return &status, nil
		}

		// if we are in monotonic mode and the condemned target is not the first unhealthy Pod, block.
		// TODO: Since available is superset of Ready, once we have this featuregate enabled by default, we can remove the
		// isRunningAndReady block as only Available pods should be brought down.
		if utilfeature.DefaultFeatureGate.Enabled(features.StatefulSetMinReadySeconds) && !isRunningAndAvailable(condemned[target], set.Spec.MinReadySeconds) && monotonic && condemned[target] != firstUnhealthyPod {
			......
			return &status, nil
		}

		// 否则直接删除该 pod
		if err := ssc.podControl.DeleteStatefulPod(set, condemned[target]); err != nil {
			return &status, err
		}
		if getPodRevision(condemned[target]) == currentRevision.Name {
			status.CurrentReplicas--
		}
		if getPodRevision(condemned[target]) == updateRevision.Name {
			status.UpdatedReplicas--
		}

		// 如果为 OrderedReady 方式则返回否则继续处理下一个 pod
		if monotonic {
			return &status, nil
		}
	}

	// 对于 OnDelete 策略直接返回
	if set.Spec.UpdateStrategy.Type == apps.OnDeleteStatefulSetStrategyType {
		return &status, nil
	}

	// 若为 RollingUpdate 策略，则倒序处理 replicas数组中下标大于等于        
	//        Spec.UpdateStrategy.RollingUpdate.Partition 的 pod
	updateMin := 0
	if set.Spec.UpdateStrategy.RollingUpdate != nil {
		updateMin = int(*set.Spec.UpdateStrategy.RollingUpdate.Partition)
	}

	for target := len(replicas) - 1; target >= updateMin; target-- {
		// 如果Pod的Revision 不等于 updateRevision，且 pod 没有处于删除状态则直接删除 pod
		if getPodRevision(replicas[target]) != updateRevision.Name && !isTerminating(replicas[target]) {
			......
			err := ssc.podControl.DeleteStatefulPod(set, replicas[target])
			status.CurrentReplicas--
			return &status, err
		}

		// 如果 pod 非 healthy 状态直接返回
		if !isHealthy(replicas[target]) {
			return &status, nil
		}
	}
	return &status, nil
}
```

# 总结

statefulSet主要是用来部署有状态应用的，statefulset 中的 pod 名称存在顺序性和唯一性，同时每个 pod 都使用了 pv 和 pvc 来存储状态，在创建、删除、更新操作中都会按照 pod 的顺序进行。