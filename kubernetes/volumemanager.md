## nestedpendingoperations: 如何保证异步任务并发时，同一类只有一个在执行

### 源码
kubernetes/pkg/volume/util/nestedpendingoperations/nestedpendingoperations.go

### 实现分析
所有任务以key来标识，key有以下字段：

```go
type operationKey struct {
	volumeName v1.UniqueVolumeName
	podName    volumetypes.UniquePodName
	nodeName   types.NodeName
}
```

假设存在任务1和任务2，对应的key分别为key1和key2，以下情形可以为同一类任务，具体见方法（nestedPendingOperations.isOperationExists）：
1. `key1.volumName == key2.volumName`
2. `key1.podName == key2.podName || key1.podName == "" || key2.podName == ""`
3. `key1.nodeName == key2.nodeName || key1.nodeName == "" || key2.nodeName == ""`

开始新任务时：
1. 如果存在同类任务：
	a. 如果存在pending（执行中）的同类任务：不执行该任务，返回报错；
	b. 如果同类任务全部非pending（执行中），执行重试（backOffErr，或者执行该任务的下一阶段 operationName 不同即可认为任务阶段不同）
2. 如果不存在同类任务，执行新任务，并记录任务状态

任务执行时，会记录任务状态：
```go
type nestedPendingOperations struct {
	// operations记录所有任务状态
	operations                []operation
	// ...
}

type operation struct {
	key              operationKey
	operationName    string
	operationPending bool
	expBackoff       exponentialbackoff.ExponentialBackoff
}
```
如果任务执行完成，将从任务状态记录中移除该任务。


`NestedPendingOperations`所有接口：

```go
type NestedPendingOperations interface {

	// 执行新任务，如果有pending（执行中）任务，报错，任务不执行；
	// 其他情形见上面分析
	// 前三个参数组成任务key，最后一个参数为任务的抽象表示
	Run(
		volumeName v1.UniqueVolumeName,
		podName volumetypes.UniquePodName,
		nodeName types.NodeName,
		generatedOperations volumetypes.GeneratedOperations) error

	// 等待所有任务停止执行（注：非全部执行完成）
	Wait()

	// 查看任务是否执行中
	IsOperationPending(
		volumeName v1.UniqueVolumeName,
		podName volumetypes.UniquePodName,
		nodeName types.NodeName) bool

	// 查看任务是否可以重试，避免重试过快
	IsOperationSafeToRetry(
		volumeName v1.UniqueVolumeName,
		podName volumetypes.UniquePodName,
		nodeName types.NodeName, operationName string) bool
}
```

## OperationExecutor and OperationGenerator

### 源码
k8s.io/kubernetes/pkg/volume/util/operationexecutor

### 实现分析
OperationExecutor顾名思义，为Operation（任务）的Executor，并且利用NestedPendingOperations来避免同类任务并发执行。

NewOperationExecutor的唯一入参即为 OperationGenerator对象，用于生成Operation，如Attach/Mount等任务。
```go
// NewOperationExecutor returns a new instance of OperationExecutor.
func NewOperationExecutor(
	operationGenerator OperationGenerator) OperationExecutor {

	return &operationExecutor{
		pendingOperations: nestedpendingoperations.NewNestedPendingOperations(
			true /* exponentialBackOffOnError */),
		operationGenerator: operationGenerator,
	}
}
```
另外一个内部变量为 NestedPendingOperations对象，用于**异步**执行Operation（任务）。


## ExponentialBackoff
```go
// ExponentialBackoff contains the last occurrence of an error and the duration
// that retries are not permitted.
type ExponentialBackoff struct {
	lastError           error
	lastErrorTime       time.Time
	durationBeforeRetry time.Duration
}
```
ExponentialBackoff是一个简单的重试机制。其字段说明：
lastErr用于记录最后一次error
lastErrorTime用于记录最后一次error的发生时间
durationBeforeRetry用于记录下次重试距离lastErrorTime的时间。

durationBeforeRetry计算方式：
初始值为initialDurationBeforeRetry = 500 * time.Millisecond
最大值为maxDurationBeforeRetry = 2*time.Minute + 2*time.Second

### ExponentialBackoff.Update
每次任务（OperationName）error，执行`Update`方法，即会更新lastErr/lastErrorTime/durationBeforeRetry字段，
durationBeforeRetry = durationBeforeRetry * 2

### ExponentialBackoff.SafeToRetry
如果当前`time.Now()`还在`lastErrorTime + durationBeforeRetry`范围内，返回`exponentialBackoffError`，即还不能重试。

