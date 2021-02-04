---
title: golang系列(35)--etcd源码阅读1
date: 2021-02-04 11:05:17
tags:
- golang
categories:
- golang
---

`etcd`是一种开源的分布式统一键值存储数据库，用于分布式系统或计算机集群的共享配置、服务发现和的调度协调。etcd 有助于促进更加安全的自动更新，协调向主机调度的工作，并帮助设置容器的覆盖网络。它已经广泛用于云原生各大项目中。现在来学习一下它的源码实现。

<!-- more -->

源码`github`地址，https://github.com/etcd-io/etcd。

下载源码后，可以看到`etcd`项目结构非常清晰。

首先从`server`包开始阅读。

## lease

`lease`包位于`etcd\sever`目录下。

`lease`中文是租约，类似于`redis`中的`TTL(Time To Live)`，用于实现过期时间。

`Lease`源码的定义如下,`Lease`结构体中`ID`是一个`int64`的标识，`ttl`是过期时间，`remainingTTL`是剩余过期时间。

```golang
type Lease struct {
	ID           LeaseID // type LeaseID int64
	ttl          int64 // time to live of the lease in seconds
	remainingTTL int64 // remaining time to live in seconds, if zero valued it is considered unset and the full ttl should be used
	// expiryMu protects concurrent accesses to expiry
	expiryMu sync.RWMutex
	// expiry is time when lease should expire. no expiration when expiry.IsZero() is true
	expiry time.Time

	// mu protects concurrent accesses to itemSet
	mu      sync.RWMutex
	itemSet map[LeaseItem]struct{}
	revokec chan struct{}
}
```

`Remaining`方法用于计算剩余过期时间，从源码上可以看出，`expiry`字段用于计算是否过期，当它为零时即永不过期。

```golang
// Remaining returns the remaining time of the lease.
func (l *Lease) Remaining() time.Duration {
	l.expiryMu.RLock()
	defer l.expiryMu.RUnlock()
	if l.expiry.IsZero() {
		return time.Duration(math.MaxInt64)
	}
	return time.Until(l.expiry)
}
```

`persistTo`是`Lease`持久化的方法，接收参数`b backend.Backend`和`ci cindex.ConsistentIndexer`都是接口，因此`Lease`能够支持不同的存储。

```golang
func (l *Lease) persistTo(b backend.Backend, ci cindex.ConsistentIndexer) {
	key := int64ToBytes(int64(l.ID))

	lpb := leasepb.Lease{ID: int64(l.ID), TTL: l.ttl, RemainingTTL: l.remainingTTL}
	val, err := lpb.Marshal()
	if err != nil {
		panic("failed to marshal lease proto item")
	}

	b.BatchTx().Lock()
	b.BatchTx().UnsafePut(leaseBucketName, key, val)
	if ci != nil {
		ci.UnsafeSave(b.BatchTx())
	}
	b.BatchTx().Unlock()
}
```

## Lessor

`Lessor`可以理解成房东，它是`lease`的管理者

`Lessor`定义是一个接口。

```golang
// Lessor owns leases. It can grant, revoke, renew and modify leases for lessee.
type Lessor interface {
	SetRangeDeleter(rd RangeDeleter)
	SetCheckpointer(cp Checkpointer)
	Revoke(id LeaseID) error
	Checkpoint(id LeaseID, remainingTTL int64) error
	Attach(id LeaseID, items []LeaseItem) error
	GetLease(item LeaseItem) LeaseID
	Detach(id LeaseID, items []LeaseItem) error
	Promote(extend time.Duration)
	Demote()
	Renew(id LeaseID) (int64, error)
	Lookup(id LeaseID) *Lease
	Leases() []*Lease
	ExpiredLeasesC() <-chan []*Lease
	Recover(b backend.Backend, rd RangeDeleter)
	Stop()
}
```

`lessor`是`Lessor`接口的实现。

```golang
type lessor struct {
	mu sync.RWMutex

	demotec chan struct{}
  // 租约存储
  leaseMap             map[LeaseID]*Lease
  // 租约过期通知组件
  leaseExpiredNotifier *LeaseExpiredNotifier
  // LeaseQueue是一个最小堆，用于获取当前过期的租约
	leaseCheckpointHeap  LeaseQueue
	itemMap              map[LeaseItem]LeaseID

	rd RangeDeleter
  // 确保 etcd 集群切换leader 后lease ttl 的准确计算
	cp Checkpointer
  // 存储
	b backend.Backend

	minLeaseTTL int64

	expiredC chan []*Lease
	stopC chan struct{}
	doneC chan struct{}

	lg *zap.Logger

	checkpointInterval time.Duration
	expiredLeaseRetryInterval time.Duration
	ci                        cindex.ConsistentIndexer
}
```

## LeaseQueue

`Lessor`管理租约，需要定期清理过期的租约，当租约数量少的时候线性查找性能还不是问题，随着租约规模扩大淘汰过期租约就会成为瓶颈性能，堆这种优先队列是最好的解决办法。

`LeaseWithTime`是最小堆的元素，结构体中包含了每个租约的过期时间。

```golang
type LeaseWithTime struct {
	id    LeaseID
	time  time.Time // 过期时间
	index int
}
```

`LeaseQueue`也是通过实现标准库中`container/heap`包中的`Interface`实现堆。

```golang
type LeaseQueue []*LeaseWithTime

func (pq LeaseQueue) Len() int { return len(pq) }

func (pq LeaseQueue) Less(i, j int) bool {
	return pq[i].time.Before(pq[j].time)
}

func (pq LeaseQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

func (pq *LeaseQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*LeaseWithTime)
	item.index = n
	*pq = append(*pq, item)
}

func (pq *LeaseQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	item.index = -1 // for safety
	*pq = old[0 : n-1]
	return item
}
```

有了堆实现过期查找就非常容易了，只需要不断的检查堆顶元素是否过期即可。

```golang
// expireExists returns true if expiry items exist.
// It pops only when expiry item exists.
// "next" is true, to indicate that it may exist in next attempt.
func (le *lessor) expireExists() (l *Lease, ok bool, next bool) {
	if le.leaseExpiredNotifier.Len() == 0 {
		return nil, false, false
	}

	item := le.leaseExpiredNotifier.Poll()
	l = le.leaseMap[item.id]
	if l == nil {
		// lease has expired or been revoked
		// no need to revoke (nothing is expiry)
		le.leaseExpiredNotifier.Unregister() // O(log N)
		return nil, false, true
	}
	now := time.Now()
	if now.Before(item.time) /* item.time: expiration time */ {
		// Candidate expirations are caught up, reinsert this item
		// and no need to revoke (nothing is expiry)
		return l, false, false
	}

	// recheck if revoke is complete after retry interval
	item.time = now.Add(le.expiredLeaseRetryInterval)
	le.leaseExpiredNotifier.RegisterOrUpdate(item)
	return l, true, false
}

func (le *lessor) findDueScheduledCheckpoints(checkpointLimit int) []*pb.LeaseCheckpoint {
	if le.cp == nil {
		return nil
	}

	now := time.Now()
	cps := []*pb.LeaseCheckpoint{}
	for le.leaseCheckpointHeap.Len() > 0 && len(cps) < checkpointLimit {
    // 检查顶项元素是否过期，过过期直接返回
		lt := le.leaseCheckpointHeap[0]
		if lt.time.After(now) /* lt.time: next checkpoint time */ {
			return cps
		}
		heap.Pop(&le.leaseCheckpointHeap)
		var l *Lease
		var ok bool
		if l, ok = le.leaseMap[lt.id]; !ok {
			continue
		}
		if !now.Before(l.expiry) {
			continue
		}
		remainingTTL := int64(math.Ceil(l.expiry.Sub(now).Seconds()))
		if remainingTTL >= l.ttl {
			continue
		}
		if le.lg != nil {
			le.lg.Debug("Checkpointing lease",
				zap.Int64("leaseID", int64(lt.id)),
				zap.Int64("remainingTTL", remainingTTL),
			)
		}
		cps = append(cps, &pb.LeaseCheckpoint{ID: int64(lt.id), Remaining_TTL: remainingTTL})
	}
	return cps
}
```

## LeaseExpiredNotifier

`LeaseExpiredNotifier`是一个租约过期通知组件。

```golang
type LeaseExpiredNotifier struct {
	m     map[LeaseID]*LeaseWithTime
	queue LeaseQueue
}
```

前面提到的`expireExists`方法即是通过`LeaseExpiredNotifier`中的堆`LeaseQueue`来实现的。

`LeaseExpiredNotifier`的`Unregister`方法即为堆的`Pop`，`RegisterOrUpdate`则完成的是堆的修改或插入。

```golang
func (mq *LeaseExpiredNotifier) RegisterOrUpdate(item *LeaseWithTime) {
	if old, ok := mq.m[item.id]; ok {
		old.time = item.time
		heap.Fix(&mq.queue, old.index)
	} else {
		heap.Push(&mq.queue, item)
		mq.m[item.id] = item
	}
}

func (mq *LeaseExpiredNotifier) Unregister() *LeaseWithTime {
	item := heap.Pop(&mq.queue).(*LeaseWithTime)
	delete(mq.m, item.id)
	return item
}
```

## loop

`newLessor`方法在新建`lessor`之后会起一个`goroutine`循环检查租约。

```golang
func newLessor(lg *zap.Logger, b backend.Backend, cfg LessorConfig, ci cindex.ConsistentIndexer) *lessor {
	checkpointInterval := cfg.CheckpointInterval
	expiredLeaseRetryInterval := cfg.ExpiredLeasesRetryInterval
	if checkpointInterval == 0 {
		checkpointInterval = defaultLeaseCheckpointInterval
	}
	if expiredLeaseRetryInterval == 0 {
		expiredLeaseRetryInterval = defaultExpiredleaseRetryInterval
	}
	l := &lessor{
		leaseMap:                  make(map[LeaseID]*Lease),
		itemMap:                   make(map[LeaseItem]LeaseID),
		leaseExpiredNotifier:      newLeaseExpiredNotifier(),
		leaseCheckpointHeap:       make(LeaseQueue, 0),
		b:                         b,
		minLeaseTTL:               cfg.MinLeaseTTL,
		checkpointInterval:        checkpointInterval,
		expiredLeaseRetryInterval: expiredLeaseRetryInterval,
		// 收集失效的租约，缓冲长度16，避免堵塞
		expiredC: make(chan []*Lease, 16),
		stopC:    make(chan struct{}),
		doneC:    make(chan struct{}),
		lg:       lg,
		ci:       ci,
	}
	l.initAndRecover()

	go l.runLoop()

	return l
}
```

500毫秒一次循环检查并撤销过期租约，最后失效的租约都会写入`expiredC`这个channel中。

```golang
func (le *lessor) runLoop() {
	defer close(le.doneC)

	for {
		le.revokeExpiredLeases()
		le.checkpointScheduledLeases()

		select {
		case <-time.After(500 * time.Millisecond):
		case <-le.stopC:
			return
		}
	}
}

// revokeExpiredLeases finds all leases past their expiry and sends them to expired channel for
// to be revoked.
func (le *lessor) revokeExpiredLeases() {
	var ls []*Lease

	// rate limit
	revokeLimit := leaseRevokeRate / 2

	le.mu.RLock()
	if le.isPrimary() {
		ls = le.findExpiredLeases(revokeLimit)
	}
	le.mu.RUnlock()

	if len(ls) != 0 {
		select {
		case <-le.stopC:
			return
		case le.expiredC <- ls:
		default:
			// the receiver of expiredC is probably busy handling
			// other stuff
			// let's try this next time after 500ms
		}
	}
}

// checkpointScheduledLeases finds all scheduled lease checkpoints that are due and
// submits them to the checkpointer to persist them to the consensus log.
func (le *lessor) checkpointScheduledLeases() {
	var cps []*pb.LeaseCheckpoint

	// rate limit
	for i := 0; i < leaseCheckpointRate/2; i++ {
		le.mu.Lock()
		// 主节点
		if le.isPrimary() {
			cps = le.findDueScheduledCheckpoints(maxLeaseCheckpointBatchSize)
		}
		le.mu.Unlock()

		if len(cps) != 0 {
			le.cp(context.Background(), &pb.LeaseCheckpointRequest{Checkpoints: cps})
		}
		if len(cps) < maxLeaseCheckpointBatchSize {
			return
		}
	}
}
```


