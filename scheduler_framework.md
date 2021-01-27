## Scheduler Framework 

​		前面我们聊了 kubernetes 的默认调度器 default scheduler，其简单的调度逻辑，在 kubernetes 多个版本的迭代中一直保持稳定性能。不过随着 Kubernetes 部署的任务类型越来越多，原生的调度器已经不能应对多样的调度需求：比如机器学习、深度学习训练任务中对于多个 pod 协同调度的需求；大数据作业有原来自己的生态，需要在调度层面做相应的适配和迁移；原来高性能计算作业中，对一些高性能组件像 GPU、infiniteBand 网络、存储卷的动态资源的绑定需求等。另外，越来越多的 feature 也一直在被引入到 scheduler 的主干中，也使得 kube-scheduler 的维护变得越来越困难。 

​		所以，kubernetes 社区在 v1.15 的版本提出中开始逐步引入 scheduler framework 为 kube-scheduler 带来更多的可扩展性，把之前很多的调度逻辑都通过 plugin 的形式重新改造，同时引入了更多位点方便定制 scheduler。本文会先讨论 scheduler 的原理，然后通过分析不同的plugin 的代码实现来更加具体的了解 scheduler framework。



## 原理

scheduler framework 最早是通过 kubernetes enhancements 的 [624-scheduling-framework](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/624-scheduling-framework) 提案引入的，主要是为了实现以下几个目标：

- 提供更多自定位位点和更多的可扩展性。
- 简化scheduler 的核心代码，把部分 features 的实现迁移到 plugin 中。
- 提供一种高效的机制，确认 plugins 的结果或者启用 plugins 的结果，并对发生的错误进行处理。
- 支持 out-of-tree 的扩展等。

为此 scheduler framework 定义了多个扩展点如下：

![scheduling-framework-extensions](scheduler_framework/scheduling-framework-extensions.png)

上图的调度周期（scheduling cycle）和绑定周期（binding cycle）具体的逻辑在我们之前的文章中已经有所讲述了，不过为了讲述方便我们也回顾一下，kube-scheduler 具体的调度过程。

![scheduler_framework](scheduler_framework/scheduler_framework.png)

- 当用户创建 pod 的时候，apiserver 接受请求把数据写入到 etcd
- scheduler 的 informer 会监听到 pod 创建的信息，然后把事件同步给 scheudulingQueue （如果是已经调度过的pod，如：被删除的deployment pod 会同步到 schedulerCache），进入 queue 的 pod 会通过 **Sort** 的 plugins 对pod 的优先级进行排序，优先级高的放在前面。
- scheduler 的主逻辑（scheduleOne）会不断地从 schedulingQueue 弹出未调度的 pod
- 然后调用 **PrefilterPlugins** 对 pod 进行预筛选
- 然后调用 **FilterPlugins** 对 nodes 的信息进行筛选，主要看一下资源是否足够等
- 然后会把前面过滤的 nodes 塞给 prioritizeNodes 函数主要会给前面过滤的 nodes 一个优先级。期间会调用 **PreScorePlugins** 和 **ScorePlugins** 用于计算每个plugins 对每个节点的分数，算分的时候会调用 **NormalizeScore** 用于对分数归一化。
- 通过selectHost进行汇总和选出合适的节点
- 如果没有选出合适的节点，会调用 **PostFilterPlugins** 主要是选主失败的时候调用，会运行抢占驱逐等逻辑
- 如果调度成功会通过 assume 告诉 cache pod 已经调度，之后会调用 **ReservePlugins** 也会告诉 cache 要预留 pod 调度需要的资源。通常设置了 reservePlugins 就需要设置 **UnReservePlugins** 保证如果后续的步骤失败了，可以释放前面预调度预留的资源。
- 然后触发 **PermitPlugins**，只要有一个 permit 不满足，就会在后面的绑定的时候停留在 waitOnPermit 中，直到 permit 的条件满足或者超时
- 然后进入异步调度逻辑，会调用 **PreBindPlugins** 主要是对所需的网络，存储等资源进行预绑定。
- 然后进入bind函数，会调用 **BindPlugins**， 不过目前主要还是默认的绑定方式。
- 如果绑定成功会调用 **PostBindPlugins** 对绑定现场进行清理。

上面就是整一个调度流程的步骤，也已经把所有 plugin 的位点给出，下面我们会看一下 plugin 具体的实现结构。

## 实现

​		为了复现和查找，我这里的主要以 [kubernetes](https://github.com/kubernetes/kubernetes) 的 v1.19.1 的版本进行分析。上面我们讲述了 scheduler framework 的一系列位点和起的作用，在源码中，scheduler-framework 主要是通过 pkg/scheduler/framework/runtime/framework.go 中的 frameworkImpl 来实现 pkg/scheduler/framework/v1alpha1/interface.go  中的 Framework 接口，然后在通过 scheduleOne 中调用相关逻辑来实现 plugin 的注入。

### framwork

​		Framework 的接口如下所示，使用上，可以认为一个 framework 就是一个调度器，里面可以加入各种 plugins，kube-scheduler 就是一个 framework，不过加入了各种 default 的 plugins 而已。Framework 提供的接口跟上面调度流程的每一个 plugin 基本上一一对应，同时加入了 FrameworkHandle 的接口，主要提供 ClientSet 和 Informer 等接口方便根据不同资源进行调度。

```go
type Framework interface {
   FrameworkHandle
  
   QueueSortFunc() LessFunc
  
   RunPreFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod) *Status
  
   RunFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) PluginToStatus
  
   RunPostFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
  
   RunPreFilterExtensionAddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
  
   RunPreFilterExtensionRemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
  
   RunPreScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
  
   RunScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) (PluginToNodeScores, *Status)
  
   RunPreBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
  
   RunPostBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string)
  
   RunReservePluginsReserve(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
  
   RunReservePluginsUnreserve(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string)
  
   RunPermitPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
  
   WaitOnPermit(ctx context.Context, pod *v1.Pod) *Status
  
   RunBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
  
   HasFilterPlugins() bool
  
   HasPostFilterPlugins() bool
  
   HasScorePlugins() bool
  
   ListPlugins() map[string][]config.Plugin
}

type FrameworkHandle interface {
	SnapshotSharedLister() SharedLister

	IterateOverWaitingPods(callback func(WaitingPod))

	GetWaitingPod(uid types.UID) WaitingPod

	RejectWaitingPod(uid types.UID)

	ClientSet() clientset.Interface

	EventRecorder() events.EventRecorder

	SharedInformerFactory() informers.SharedInformerFactory

	PreemptHandle() PreemptHandle
}
```

 在 pkg/scheduler/framework/runtime/framework.go 中 frameworkImpl 实现了 Framework 的接口，基本上如果没有特殊需求（改变不同 plugins 通过的逻辑等），可以直接使用 frameworkImpl 的实现，基本上所有的接口函数就是提供调用不同 plugins 的逻辑，有的是只要有一个plugins通过了，就会往下走，如RunPostFilterPlugins，只要有一个postFilterPlugin success 就会往下走，有的是要所有plugins通过了，才会往下走，如 RunFilterPlugins 会轮询所有的plugins都通过才往下走 ，否则返回错误示例如下：

RunPostFilterPlugins：

```go
func (f *frameworkImpl) RunPostFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, filteredNodeStatusMap framework.NodeToStatusMap) (_ *framework.PostFilterResult, status *framework.Status) {
   startTime := time.Now()
   defer func() {
      metrics.FrameworkExtensionPointDuration.WithLabelValues(postFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
   }()

   statuses := make(framework.PluginToStatus)
   for _, pl := range f.postFilterPlugins {
      r, s := f.runPostFilterPlugin(ctx, pl, state, pod, filteredNodeStatusMap)
      if s.IsSuccess() {
         return r, s
      } else if !s.IsUnschedulable() {
         // Any status other than Success or Unschedulable is Error.
         return nil, framework.NewStatus(framework.Error, s.Message())
      }
      statuses[pl.Name()] = s
   }

   return nil, statuses.Merge()
}
```

RunFilterPlugins：

```go
func (f *frameworkImpl) RunFilterPlugins(
   ctx context.Context,
   state *framework.CycleState,
   pod *v1.Pod,
   nodeInfo *framework.NodeInfo,
) framework.PluginToStatus {
   statuses := make(framework.PluginToStatus)
   for _, pl := range f.filterPlugins {
      pluginStatus := f.runFilterPlugin(ctx, pl, state, pod, nodeInfo)
      if !pluginStatus.IsSuccess() {
        ...
      }
   }

   return statuses
}
```

### Plugin

我们继续以 RunFilterPlugins 为例继续往下看，上面会分别为不同的 plugin 调用 runFilterPlugin(示例如下)：

```go
func (f *frameworkImpl) runFilterPlugin(ctx context.Context, pl framework.FilterPlugin, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
   if !state.ShouldRecordPluginMetrics() {
      return pl.Filter(ctx, state, pod, nodeInfo)
   }
   startTime := time.Now()
   status := pl.Filter(ctx, state, pod, nodeInfo)
   f.metricsRecorder.observePluginDurationAsync(Filter, pl.Name(), status, metrics.SinceInSeconds(startTime))
   return status
}
```

然后调用 FilterPlugin 的 Filter 方法，这步就是我们需要自己实现的步骤了，我们看一下，Plugin 的定义如下：

```go
type Plugin interface {
   Name() string
}
type FilterPlugin interface {
	Plugin
  
	Filter(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) *Status
}

```

如果需要添加自定义的过滤步骤，只需要实现Name, Filter 函数，然后在启动 custom scheduler 的时候加上这个 plugins 就可以了。具体我们看一个 filter 的实现，以节点亲和性（nodeaffinity）的 Plugins 为例进行讲述。

#### nodeAffinity

一般来说，节点的亲和性可以通过 pod.nodeSelectors 的字段进行配置，如 pod需要运行在 带有 "storage" :"ssd"  的机器上，可以在 pod.nodeSelector 中添加相关的字段。如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeSelector:
    storage: "ssd"
  containers:
  - name: nginx
```

不过上面通过 nodeSelector 是比较硬性的要求，也可以使用节点亲和性的约束更加精细化地优化：

- requiredDuringSchedulingIgnoredDuringExecution
- requiredDuringSchedulingRequiredDuringExecution
- preferredDuringSchedulingIgnoredDuringExecution
- preferredDuringSchedulingRequiredDuringExecution

 上面四个条件 require 表示硬性的条件，preferred 是软性的，前面 DuringScheduling 表示只需要在调度时候满足就可以了，后面 DuringExecution 表示需要在 执行的时候也需要满足，主要是在 node 标签发生变化的时候产生。如下面的 pod 表示需要调度到具有 `e2e-az1`，`e2e-az2` 这两个label的节点上， 另外，在满足这些标准的节点中，具有标签键为 `another-node-label-key` 且标签值为 `another-node-label-value` 的节点应该优先使用。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/

了解了上面的 nodeSelector, nodeAffinity 的行为，我们看一下调度器是怎么做的。对于硬性的亲和性条件，是通过 Filter 函数实现的，具体逻辑在 PodMatchesNodeSelectorAndAffinityTerms 函数中 ，先查看selector，如果有设置 selector 看一下是不是 match，如果 match 就返回 true，如果设置了硬性的 RequiredDuringSchedulingIgnoredDuringExecution，看一下是不是这个node，否则直接通过。

pkg/scheduler/framework/plugins/nodeaffinity/node_affinity.go

```go
func (pl *NodeAffinity) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
   node := nodeInfo.Node()
   if node == nil {
      return framework.NewStatus(framework.Error, "node not found")
   }
   if !pluginhelper.PodMatchesNodeSelectorAndAffinityTerms(pod, node) {
      return framework.NewStatus(framework.UnschedulableAndUnresolvable, ErrReason)
   }
   return nil
}
```

pkg/scheduler/framework/plugins/helper/node_affinity.go

```go
func PodMatchesNodeSelectorAndAffinityTerms(pod *v1.Pod, node *v1.Node) bool {
   // Check if node.Labels match pod.Spec.NodeSelector.
   if len(pod.Spec.NodeSelector) > 0 {
      selector := labels.SelectorFromSet(pod.Spec.NodeSelector)
      if !selector.Matches(labels.Set(node.Labels)) {
         return false
      }
   }
   nodeAffinityMatches := true
   affinity := pod.Spec.Affinity
   if affinity != nil && affinity.NodeAffinity != nil {
      nodeAffinity := affinity.NodeAffinity
      if nodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution == nil {
         return true
      }

      if nodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution != nil {
         nodeSelectorTerms := nodeAffinity.RequiredDuringSchedulingIgnoredDuringExecution.NodeSelectorTerms
         nodeAffinityMatches = nodeAffinityMatches && nodeMatchesNodeSelectorTerms(node, nodeSelectorTerms)
      }

   }
   return nodeAffinityMatches
}
```

软性的affinity 是通过 Score 就是调度时候优选的阶段进行打分的，其实现如下：

```go
func (pl *NodeAffinity) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
   nodeInfo, err := pl.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
   if err != nil {
      return 0, framework.NewStatus(framework.Error, fmt.Sprintf("getting node %q from Snapshot: %v", nodeName, err))
   }

   node := nodeInfo.Node()
   if node == nil {
      return 0, framework.NewStatus(framework.Error, fmt.Sprintf("getting node %q from Snapshot: %v", nodeName, err))
   }

   affinity := pod.Spec.Affinity

   var count int64
   if affinity != nil && affinity.NodeAffinity != nil && affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution != nil {
      // Match PreferredDuringSchedulingIgnoredDuringExecution term by term.
      for i := range affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution {
         preferredSchedulingTerm := &affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution[i]
         if preferredSchedulingTerm.Weight == 0 {
            continue
         }

         // TODO: Avoid computing it for all nodes if this becomes a performance problem.
         nodeSelector, err := v1helper.NodeSelectorRequirementsAsSelector(preferredSchedulingTerm.Preference.MatchExpressions)
         if err != nil {
            return 0, framework.NewStatus(framework.Error, err.Error())
         }

         if nodeSelector.Matches(labels.Set(node.Labels)) {
            count += int64(preferredSchedulingTerm.Weight)
         }
      }
   }

   return count, nil
}
```

如果没有设置 affinity，直接返回零分（不同节点分值相同），如果设置了，看一下是不是match，如果match 根据 PreferredDuringSchedulingIgnoredDuringExecution 的权重进行打分。

​		整体上，nodeAffinity 的实现比较直观的，然后在这里也可以看出不同的 plugin 可以通过实现多个 plugin 的接口来完成更加复杂的调度逻辑。还有其他的 plugin 这里就不一一举例

了。

## Scheduler-plugins

如果要自己实现一个 out-of-tree 的 scheduler，可以参考 kubernetes-sigs 的 [scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins)。这个是 kubernetes 的 scheduler 兴趣小组自己维护的一个 scheduler framework plugins，实现了一些比较常用调度的 plugins，如：协同调度和

podgroup controller 

-   如果有一个 pod 的事件进来了
-   list 所有label 了这个 podgroup 的 pod
-   如果 pods 的数量大于 pg.Spec.MinMember 的数量
-   把 podgroup 的状态调整为 scheduling
-   如果已经开始调度，看 pg 下 pod 的 status，如果 pg succedded+running 大于 minMem 就是在 running。

