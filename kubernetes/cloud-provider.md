## k8s.io/cloud-provider


-----------------------

## 1. 代码框架设计分析

### 2. 模块解析

```
// k8s.io/cloud-provider/app/core.go

   -startCloudNodeController(ctx context.Context, initContext ControllerInitContext, controlexContext controllermanagerapp.ControllerContext, completedConfig *config.CompletedConfig, cloud cloudprovider.Interface) : controller.Interface, bool, error
   -startCloudNodeLifecycleController(ctx context.Context, initContext ControllerInitContext, controlexContext controllermanagerapp.ControllerContext, completedConfig *config.CompletedConfig, cloud cloudprovider.Interface) : controller.Interface, bool, error
   -startRouteController(ctx context.Context, initContext ControllerInitContext, controlexContext controllermanagerapp.ControllerContext, completedConfig *config.CompletedConfig, cloud cloudprovider.Interface) : controller.Interface, bool, error
   -startServiceController(ctx context.Context, initContext ControllerInitContext, controlexContext controllermanagerapp.ControllerContext, completedConfig *config.CompletedConfig, cloud cloudprovider.Interface) : controller.Interface, bool, error

```
k8s.io/cloud-provider从功能设计上说，就是启动上述Controller，每个Controller负责一部分功能实现。


#### 2.1 ServiceController
功能说明：ServiceController负责实现LoadBalancer类型（即`.spec.type == "LoadBalancer"`）的Service

ServiceController对service、node对象进行监听，调用LoadBalancer创建、删除、更新负载均衡。
```go
type LoadBalancer interface {
	GetLoadBalancer(ctx context.Context, clusterName string, service *v1.Service) (status *v1.LoadBalancerStatus, exists bool, err error)
	GetLoadBalancerName(ctx context.Context, clusterName string, service *v1.Service) string
	// 创建、或者更新负载均衡，注意：也包括了更新！
	EnsureLoadBalancer(ctx context.Context, clusterName string, service *v1.Service, nodes []*v1.Node) (*v1.LoadBalancerStatus, error)
	// 更新负载均衡
	UpdateLoadBalancer(ctx context.Context, clusterName string, service *v1.Service, nodes []*v1.Node) error
	// 删除负载均衡
	EnsureLoadBalancerDeleted(ctx context.Context, clusterName string, service *v1.Service) error
}
```

使用方只需要负责实现LoadBalancer的各个接口，作为ServiceController参数的一部分创建ServiceController启动并运行（见`startServiceController`），即可完成LoadBalancer类型的service的实现。


##### 2.1.1 源码解析
ServiceController代码实现见`controllers/service/controller.go`。

Controller解析：
```
-+Controller : struct
    [methods]
	// 启动Controller，workers控制并发，即处理service的协程数；controllerManagerMetrics用于记录prometheus监控数据
   +Run(ctx context.Context, workers int, controllerManagerMetrics *controllersmetrics.ControllerManagerMetrics)

   // 初始化，如从cloudprovider.Interface中获取了LoadBalancer
   -init() : error

   // 创建service对应的负载均衡之前，会给service加上finalizer: service.kubernetes.io/load-balancer-cleanup
   -addFinalizer(service *v1.Service) : error

   // service队列入列
   -enqueueService(obj interface{})

   // 创建负载均衡
   -ensureLoadBalancer(ctx context.Context, service *v1.Service) : *v1.LoadBalancerStatus, error

   // 判断service的Update事件，是否要触发负载均衡的更新（从这个方法里可以看到service哪些字段和负载均衡是相关的）
   -needsUpdate(oldService *v1.Service, newService *v1.Service) : bool

   // ------------------------------ node sync ------------------------------

   // nodes变化触发的对全量或者部分service的Sync
   // 判断是否需要全量nodeSyncService，并设置Controller.needFullSync为false，避免反复全量同步
   -needFullSyncAndUnmark() : bool

   // 循环监听`nodeSyncCh chan interface{}`，触发nodeSyncInternal
   // 将调用nodeSyncInternal
   -nodeSyncLoop(ctx context.Context, workers int)

   // 执行全量或者部分（根据方法needFullSyncAndUnmark判断）service的Sync
   // 如果有同步失败的service，将缓存在servicesToUpdate中
   // 将调用updateLoadBalancerHosts执行批量同步
   -nodeSyncInternal(ctx context.Context, workers int)

   // 批量同步services
   // 将调用nodeSyncService，返回值：同步失败的services的keys
   -updateLoadBalancerHosts(ctx context.Context, services []*v1.Service, workers int) : sets.String

   // 调用lockedUpdateLoadBalancerHosts，如果失败，返回true，即需要重试
   -nodeSyncService(svc *v1.Service) : bool

   // nodes发生变动，更新其中单个service的负载均衡，方法里会调用LoadBalancer.UpdateLoadBalancer
   -lockedUpdateLoadBalancerHosts(service *v1.Service, hosts []*v1.Node) : error

   // 触发同步任务，即往`nodeSyncCh chan interface{}`写入`struct{}{}`
   // 以下场景会触发同步任务：
   // 1. node addEvent
   // 2. node deleteEvent
   // 3. node updateEvent, and shouldSyncNode（参考方法shouldSyncNode）
   // 这个方法里还同时设置是全量还是部分同步
   // 如果nodes的可用数量（注意：是可用数量，见getNodeConditionPredicate）发生变化，则全量同步；
   // 否则，则部分同步，即重试上次同步失败的service；
   // 
   // 注：`nodeSyncCh chan interface{}`长度为1，既避免了重复触发同步任务，又不会遗漏同步任务
   -triggerNodeSync()

   // 过滤nodes，过滤后的nodes为LoadBalancer.EnsureLoadBalancer的入参
   -getNodeConditionPredicate() : NodeConditionPredicate
   // ------------------------------ node sync ------------------------------

   -patchStatus(service *v1.Service, previousStatus, newStatus *v1.LoadBalancerStatus) : error
   -processLoadBalancerDelete(ctx context.Context, service *v1.Service, key string) : error
   -processNextWorkItem(ctx context.Context) : bool
   -processServiceCreateOrUpdate(ctx context.Context, service *v1.Service, key string) : error
   -processServiceDeletion(ctx context.Context, key string) : error
   -removeFinalizer(service *v1.Service) : error
   -syncLoadBalancerIfNeeded(ctx context.Context, service *v1.Service, key string) : loadBalancerOperation, error
   -syncService(ctx context.Context, key string) : error
   -worker(ctx context.Context)
    [functions]
   // New方法创建Controller对象。`cloudprovider.Interface`为使用方负责实现的所有接口，其中包括`LoadBalancer`
   +New(cloud cloudprovider.Interface, kubeClient clientset.Interface, serviceInformer coreinformers.ServiceInformer, nodeInformer coreinformers.NodeInformer, clusterName string, featureGate featuregate.FeatureGate) : *Controller, error
```

##### 2.1.2 如何感知service对应pods的变化?

#### 2.2 CloudNodeController

#### 2.3 CloudNodeLifecycleController

#### 2.4 RouteController
