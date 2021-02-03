#  （scheduler 03）

​		除了前面聊到的通过 scheduler framework 对 kubernetes 的调度特性进行扩展之外，还可以通过多调度器的方式进行扩展，因为我自己的背景关系，会比较关注跟机器学习工作负载相关的的调度器，现在社区比较成熟的方案有 kube-batch，和由其延伸出来的 volcano，主要是补足了 kubernetes 批调度的短板，其社区也比较活跃。

​		因为 volcano 是植根于 kube-batch，不过添加了更加多的 feature（如提供了一个 controller Manager 用于管理 volcano 自己可扩展的 job CRD 和 queue，podgroup 等资源，还有一系列命令行工具），而 kube-batch 主要以批调度器为主，这篇文章会着重于从源码的角度分析 volcano，中间如果有需要会穿插跟 kube-batch 的异同。文章组织上，我自己想从创建一个 volcano 的 job 开始，逐步对其调度流程进行分析。为了讲述和复现的方便，我们这里源码是根据 volcano v1.1.0 的版本进行分析的。

### 概述

使用 volcano 的调度流程如下，具体分 5 步，我们后面会着重去分析第2，3步的作业处理。

![volcano](volcano_public/volcano.png)

-   使用 kubectl 创建一个 volcano job。
-   vc-controller 监测到这个 job 的创建，会创建对应的 pod 和 podgroup。
-   通过 vc-scheduler 找到合适的 node 
-   向 api-server 申请 pod 和 node 的绑定
-   kubelet 看到有 pod 进行绑定，启动容器。 

### 创建 job

 我们以 examples 中的 tensorflow examples 进行分析：

example/integrations/tensorflow/dist-mnist/tf-dist-mnist-example.yaml

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: tensorflow-benchmark
  labels:
    "volcano.sh/job-type": "Tensorflow"
spec:
  minAvailable: 3
  schedulerName: volcano
  plugins:
    env: []
    svc: []
  policies:
    - event: PodEvicted
      action: RestartJob
  tasks:
    - replicas: 1
      name: ps
      template:
        spec:
          imagePullSecrets:
            - name: default-secret
          containers:
            - command:
                - sh
                - -c
                - |
                  PS_HOST=`cat /etc/volcano/ps.host | sed 's/$/&:2222/g' | tr "\n" ","`;
                  WORKER_HOST=`cat /etc/volcano/worker.host | sed 's/$/&:2222/g' | tr "\n" ","`;
                  python tf_cnn_benchmarks.py --batch_size=32 --model=resnet50 --variable_update=parameter_server --flush_stdout=true --num_gpus=1 --local_parameter_device=cpu --device=cpu --data_format=NHWC --job_name=ps --task_index=${VK_TASK_INDEX} --ps_hosts=${PS_HOST} --worker_hosts=${WORKER_HOST}
              image: volcanosh/example-tf:0.0.1
              name: tensorflow
              ports:
                - containerPort: 2222
                  name: tfjob-port
              resources:
                requests:
                  cpu: "1000m"
                  memory: "2048Mi"
                limits:
                  cpu: "1000m"
                  memory: "2048Mi"
              workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
    - replicas: 2
      name: worker
      policies:
        - event: TaskCompleted
          action: CompleteJob
      template:
        spec:
          imagePullSecrets:
            - name: default-secret
          containers:
            - command:
                - sh
                - -c
                - |
                  PS_HOST=`cat /etc/volcano/ps.host | sed 's/$/&:2222/g' | tr "\n" ","`;
                  WORKER_HOST=`cat /etc/volcano/worker.host | sed 's/$/&:2222/g' | tr "\n" ","`;
                  python tf_cnn_benchmarks.py --batch_size=32 --model=resnet50 --variable_update=parameter_server --flush_stdout=true --num_gpus=1 --local_parameter_device=cpu --device=cpu --data_format=NHWC --job_name=worker --task_index=${VK_TASK_INDEX} --ps_hosts=${PS_HOST} --worker_hosts=${WORKER_HOST}
              image: volcanosh/example-tf:0.0.1
              name: tensorflow
              ports:
                - containerPort: 2222
                  name: tfjob-port
              resources:
                requests:
                  cpu: "2000m"
                  memory: "2048Mi"
                limits:
                  cpu: "2000m"
                  memory: "4096Mi"
              workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
```

如果 volcano 已经部署，这个 job 会被 controller-manager 中的 job controller 监测到，如果 job 自身的状态没有问题，会进入 pkg/controllers/job/job_controller_actions.go 中 syncJob 的逻辑，然后 job  controller 会做以下几件事：

-   initiate Job：主要是把注册的 plugins 作用到这个 job 上。
-   创建这个 job 对应的 podgroup (默认使用 volcanoJob 都会创建），然后用 job.spec.minAvailable 填写 podgroup 的minAvailable 字段。
-   然后根据 job 中的 template 异步创建对应的 pod，也会把对应的 plugin 等信息添加上。
-   最后 updateJobStatus。

现在集群中有这个 job 对应的 pod，和 podgroup 了。

#### 延伸

#### volcanojob VS batch/v1Job

上面聊到的 volcano job（pkg/apis/batch/v1alpha1/job.go） 对比 kubernetes 下 batch/v1的 job， 主要有以下区别：

-   volcano job 会多了 SchedulerName，MinAvailable， Queue 等字段，主要是为了更契合 volcano 的调度方式。
-   使用  []TaskSpec，替代 PodTemplateSpec 作为 pod 实例的模板，因为一个 taskSpec 有一个 PodTemplateSpec，可以支持多种实例，这个跟 kubeflow 的 CRDjobs (tfjob,pytorchjob) 是比较类似的。
-   Plugins 字段主要是方便做一些 mutating，如注入 env，ssh 等。

下面是 volcano job 的定义：(pkg/apis/batch/v1alpha1/job.go)

```go
// Job defines the volcano job.
type Job struct {
   metav1.TypeMeta `json:",inline"`

   metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

   Spec JobSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

   Status JobStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type JobSpec struct {
   SchedulerName string `json:"schedulerName,omitempty" protobuf:"bytes,1,opt,name=schedulerName"`

   MinAvailable int32 `json:"minAvailable,omitempty" protobuf:"bytes,2,opt,name=minAvailable"`

   Volumes []VolumeSpec `json:"volumes,omitempty" protobuf:"bytes,3,opt,name=volumes"`

   Tasks []TaskSpec `json:"tasks,omitempty" protobuf:"bytes,4,opt,name=tasks"`

   Policies []LifecyclePolicy `json:"policies,omitempty" protobuf:"bytes,5,opt,name=policies"`

   Plugins map[string][]string `json:"plugins,omitempty" protobuf:"bytes,6,opt,name=plugins"`

   RunningEstimate *metav1.Duration `json:"runningEstimate,omitempty" protobuf:"bytes,4,opt,name=runningEstimate"`

   Queue string `json:"queue,omitempty" protobuf:"bytes,7,opt,name=queue"`

   MaxRetry int32 `json:"maxRetry,omitempty" protobuf:"bytes,8,opt,name=maxRetry"`

   TTLSecondsAfterFinished *int32 `json:"ttlSecondsAfterFinished,omitempty" protobuf:"varint,9,opt,name=ttlSecondsAfterFinished"`

   PriorityClassName string `json:"priorityClassName,omitempty" protobuf:"bytes,10,opt,name=priorityClassName"`
}
type TaskSpec struct {

	Name string `json:"name,omitempty" protobuf:"bytes,1,opt,name=name"`

	Replicas int32 `json:"replicas,omitempty" protobuf:"bytes,2,opt,name=replicas"`

	Template v1.PodTemplateSpec `json:"template,omitempty" protobuf:"bytes,3,opt,name=template"`

	Policies []LifecyclePolicy `json:"policies,omitempty" protobuf:"bytes,4,opt,name=policies"`
}

```

#### volcanojob VS CRD job		

另外，从上面流程分析可以看到，volcano job controller 同步之后主要是创建了 pod 和 podgroup，然后 volcano 的 scheduler 也不会单独去 watch volcanojob，所以后续调度流程主要就是通过 pod，podgroup 的信息来实现的，job controller 只会同步一下 status。这样也给开发者带来了灵活性，其他的 CRD jobs 也可以通过这样的机制，让自己的 CRD jobs 使用 volcano 调度起来，具体可能还需要 podgroup controller 的协助，实例如下图。具体可以参考一下，[tf-operator](https://github.com/kubeflow/tf-operator) 的实现这里就不展开了。      

![crdjob](volcano_public/crdjob.png)



### 调度

上面创建 job 的流程经过 job-controller 之后，会创建对应的 pod，和 podgroup，会直接被 scheduler 监测到，然后触发 pod，podgroup 的 addPod 和 addPodGroupV1beta1  eventHandler。

其中 addPod 会干以下几件事：

-   对 pod 生成一个对应的 taskInfo，并生成一个 jobID。
-   如果在 schedulerCache 的 jobs 没有这个 jobID，创建这个 jobInfo，然后把这个 task 放到 job.Tasks 中。
-   如果已经分配了 node，也会在 schedulerCache的 nodes 里面记录这个task







