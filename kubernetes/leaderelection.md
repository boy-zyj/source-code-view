## 1. Usage

```go
// specify a resourcelock
// lock := &resourcelock.LeaseLock{}

// start the leader election code loop
leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
	Lock: lock,
	// IMPORTANT: you MUST ensure that any code you have that
	// is protected by the lease must terminate **before**
	// you call cancel. Otherwise, you could have a background
	// loop still running and another process could
	// get elected before your background loop finished, violating
	// the stated goal of the lease.
	// 如果设置为true，锁会主动释放，必须保证锁释放前，执行的异步任务要停止
	ReleaseOnCancel: true,
	// LeaseDuration必须大于RenewDeadline，这样如果leader renew失败，至少有LeaseDuration - RenewDeadline的时间来退出异步任务（这段时间内，锁不会被其他进程获得）
	LeaseDuration:   60 * time.Second,
	RenewDeadline:   15 * time.Second,
	RetryPeriod:     5 * time.Second,
	Callbacks: leaderelection.LeaderCallbacks{
		OnStartedLeading: func(ctx context.Context) {
			// we're notified when we start - this is where you would
			// usually put your code
			run(ctx)
		},
		OnStoppedLeading: func() {
			// we can do cleanup here
			klog.Infof("leader lost: %s", id)
			os.Exit(0)
		},
		OnNewLeader: func(identity string) {
			// we're notified when new leader elected
			if identity == id {
				// I just got the lock
				return
			}
			klog.Infof("new leader elected: %s", identity)
		},
	},
})
```

## 2. 实现分析

### 2.1 resourcelock
resourcelock为通过CAS原理实现的分布式锁。

```go
type Interface interface {
	// Get returns the LeaderElectionRecord
	Get(ctx context.Context) (*LeaderElectionRecord, []byte, error)

	// Create attempts to create a LeaderElectionRecord
	Create(ctx context.Context, ler LeaderElectionRecord) error

	// Update will update and existing LeaderElectionRecord
	Update(ctx context.Context, ler LeaderElectionRecord) error

	// RecordEvent is used to record events
	RecordEvent(string)

	// Identity will return the locks Identity
	Identity() string

	// Describe is used to convert details on current resource lock
	// into a string
	Describe() string
}
```

LeaseLock为`resourcelock`的一种实现，其获得锁的过程为：

1.执行`Get(ctx context.Context) (*LeaderElectionRecord, []byte, error)`，将请求kube-apiserver，根据LeaseLocak.LeaseMeta的namespace/name查询获取到`coordinationv1.Lease`，如果查询成功，其字段`spec`转换为`LeaderElectionRecord`:

```go
func LeaseSpecToLeaderElectionRecord(spec *coordinationv1.LeaseSpec) *LeaderElectionRecord {
	var r LeaderElectionRecord
	if spec.HolderIdentity != nil {
		r.HolderIdentity = *spec.HolderIdentity
	}
	if spec.LeaseDurationSeconds != nil {
		r.LeaseDurationSeconds = int(*spec.LeaseDurationSeconds)
	}
	if spec.LeaseTransitions != nil {
		r.LeaderTransitions = int(*spec.LeaseTransitions)
	}
	if spec.AcquireTime != nil {
		r.AcquireTime = metav1.Time{Time: spec.AcquireTime.Time}
	}
	if spec.RenewTime != nil {
		r.RenewTime = metav1.Time{Time: spec.RenewTime.Time}
	}
	return &r

}
```

如果请求失败，`!errors.IsNotFound(err)`，即返回非404报错，则认为获取锁失败。

如果返回404报错，则继续执行`Create(ctx context.Context, ler LeaderElectionRecord) error`，如果成功，则获得锁；反之，获取锁失败。

2.如果查询成功，则将`spec`转换为`LeaderElectionRecord`，该对象为`oldLeaderElectionRecord`。此时：

* 如果之前未缓存过`LeaderElectionRecord`，缓存该对象，并记录当前时间`le.observedTime = le.clock.Now()`
* 如果之前缓存过`LeaderElectionRecord`，将该对象与`oldLeaderElectionRecord`进行比对，如果两者不同，则更新缓存对象，并记录当前时间`le.observedTime = le.clock.Now()`

之后根据以下条件判断是否要尝试获取锁：

1. 如果锁已经释放，即`oldLeaderElectionRecord.HolderIdentity == 0`，则尝试获取锁
2. 如果锁已经超期，即将当前时间与`le.observedTime`与进行比较，如果两者间隔时间超过`oldLeaderElectionRecord.LeaseDurationSeconds`，则认为锁超期，尝试获取锁
3. 如果当前锁的所有者为当前进程，即`le.IsLeader()`，即继续尝试获取锁，此时尝试获取锁实际作用其实为更新锁信息

当满足任意上述条件其中之一，则执行`Update(ctx context.Context, ler LeaderElectionRecord) error`获取锁，更新字段为：

* 如果当前进程即为锁的所有者，更新字段为`LeaderElectionRecord.RenewTime`
* 更新字段为`LeaderElectionRecord`所有字段

> **当多个进程同时获取锁时，有且仅有一个能够成功。**这是由kube-apiserver来保证的，即当多个进程对同一个资源对象进行Create或者Update操作时，有且仅有一个能够成功。

`LeaderElectionRecord`保存了`leader`的信息：

```go
type LeaderElectionRecord struct {
	// HolderIdentity is the ID that owns the lease. If empty, no one owns this lease and
	// all callers may acquire. Versions of this library prior to Kubernetes 1.14 will not
	// attempt to acquire leases with empty identities and will wait for the full lease
	// interval to expire before attempting to reacquire. This value is set to empty when
	// a client voluntarily steps down.
	// 一句话总结：leader的ID
	HolderIdentity       string      `json:"holderIdentity"`
	// LeaseDurationSeconds字段用于判断锁是否超期
	LeaseDurationSeconds int         `json:"leaseDurationSeconds"`
	// leader第一次获得锁时的时间
	AcquireTime          metav1.Time `json:"acquireTime"`
	// leader自第一次获得锁后，会定期重复获得锁，以避免超期，这里记录每次获取锁的时间
	RenewTime            metav1.Time `json:"renewTime"`
	// 如果锁之前为其他leader所有，获取到当前锁后，LeaderTransitions++
	LeaderTransitions    int         `json:"leaderTransitions"`
}
```

### 2.2 LeaderElectionConfig & LeaderElector

```go
// NewLeaderElector creates a LeaderElector from a LeaderElectionConfig
func NewLeaderElector(lec LeaderElectionConfig) (*LeaderElector, error) {
	// 对LeaderElectionConfig各个字段进行校验
	if lec.LeaseDuration <= lec.RenewDeadline {
		return nil, fmt.Errorf("leaseDuration must be greater than renewDeadline")
	}
	if lec.RenewDeadline <= time.Duration(JitterFactor*float64(lec.RetryPeriod)) {
		return nil, fmt.Errorf("renewDeadline must be greater than retryPeriod*JitterFactor")
	}
	if lec.LeaseDuration < 1 {
		return nil, fmt.Errorf("leaseDuration must be greater than zero")
	}
	if lec.RenewDeadline < 1 {
		return nil, fmt.Errorf("renewDeadline must be greater than zero")
	}
	if lec.RetryPeriod < 1 {
		return nil, fmt.Errorf("retryPeriod must be greater than zero")
	}
	if lec.Callbacks.OnStartedLeading == nil {
		return nil, fmt.Errorf("OnStartedLeading callback must not be nil")
	}
	if lec.Callbacks.OnStoppedLeading == nil {
		return nil, fmt.Errorf("OnStoppedLeading callback must not be nil")
	}

	if lec.Lock == nil {
		return nil, fmt.Errorf("Lock must not be nil.")
	}
	id := lec.Lock.Identity()
	if id == "" {
		return nil, fmt.Errorf("Lock identity is empty")
	}

	le := LeaderElector{
		config:  lec,
		clock:   clock.RealClock{},
		metrics: globalMetricsFactory.newLeaderMetrics(),
	}
	le.metrics.leaderOff(le.config.Name)
	return &le, nil
}
```

`LeaderElector`为`leader`的实现，`leader`的行为模式可以参考`LeaderElectionConfig`各个字段的解释：

```go
type LeaderElectionConfig struct {
	// Lock is the resource that will be used for locking
	// 用于获得锁
	Lock rl.Interface

	// LeaseDuration is the duration that non-leader candidates will
	// wait to force acquire leadership. This is measured against time of
	// last observed ack.
	//
	// A client needs to wait a full LeaseDuration without observing a change to
	// the record before it can attempt to take over. When all clients are
	// shutdown and a new set of clients are started with different names against
	// the same leader record, they must wait the full LeaseDuration before
	// attempting to acquire the lease. Thus LeaseDuration should be as short as
	// possible (within your tolerance for clock skew rate) to avoid a possible
	// long waits in the scenario.
	//
	// Core clients default this value to 15 seconds.
	// 一句话解释：用于判断锁是否超期
	LeaseDuration time.Duration
	// RenewDeadline is the duration that the acting master will retry
	// refreshing leadership before giving up.
	//
	// Core clients default this value to 10 seconds.
	// 如果leader renew锁超时(> RenewDeadline)，则退出renew，执行.Callbacks.OnStoppedLeading
	// 当退出renew时，任异步执行中的任务的ctx将cancel
	RenewDeadline time.Duration
	// RetryPeriod is the duration the LeaderElector clients should wait
	// between tries of actions.
	//
	// Core clients default this value to 2 seconds.
	// 重复获取锁的重试间隔
	RetryPeriod time.Duration

	// Callbacks are callbacks that are triggered during certain lifecycle
	// events of the LeaderElector
	// .Callbacks.OnStoppedLeading，参见字段RenewDeadline
	// . Callbacks.OnStartedLeading，当前进程成为leader后，执行异步任务
	// .Callbacks.OnNewLeader，当识别至leadership发生变化时的回调方法
	Callbacks LeaderCallbacks

	// WatchDog is the associated health checker
	// WatchDog may be null if it's not needed/configured.
	// NewLeaderHealthzAdaptor(timeout time.Duration) *HealthzAdaptor创建
	// 本质为HealthzChecker：当前进程为leader，但锁却超期，且超期时间超过timeout，则认为unhealthy
	WatchDog *HealthzAdaptor

	// ReleaseOnCancel should be set true if the lock should be released
	// when the run context is cancelled. If you set this to true, you must
	// ensure all code guarded by this lease has successfully completed
	// prior to cancelling the context, or you may have two processes
	// simultaneously acting on the critical path.
	// 如果renew失败（参考字段RenewDeadline的解释），该字段为true时，主动释放锁
	ReleaseOnCancel bool

	// Name is the name of the resource lock for debugging
	// 仅用于debugging
	Name string
}
```







