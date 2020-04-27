---
title: RWMutex
tags: Go 源码分析 并发
key: Analysis-Go-Sync-RWMutex
modify_date: 2020-04-27 15:10:43 
---

# RWMutex
`RWMutex`为借助`Mutex`实现的不可重入读写锁，在读多写少的情况下有着比互斥锁`Mutex`更好的效率，下面让我们从源码实现上看看`RWMutex`的高效性。

<!--more-->

## 数据结构
基本结构如下：
```go
type RWMutex struct {
    w           Mutex  // held if there are pending writers
    writerSem   uint32 // semaphore for writers to wait for completing readers
    readerSem   uint32 // semaphore for readers to wait for completing writers
    readerCount int32  // number of pending readers
    readerWait  int32  // number of departing readers
}
```
其中`readerSem`和`writerSem`为信号量，对应当前被阻塞的读者队列与写者队列。`readerCount`和`readerWait`会用于动态的表示活跃的读者数量，这两个变量会同时在读写锁中涉及到，因此在研究这两个变量时，需要同时结合读写锁实现来研究。


## 读锁
读锁的实现相对简单，首先看看如何获取读锁。

### RLock
```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
        // A writer is pending, wait for it.
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
```
读锁的获取需要查看当前是否有写者处于等待之中，判断的依据是`atomic.AddInt32(&rw.readerCount, 1)`原子加后`readerCount`是否小于0：
- 如果小于0则请求`readcerSem`信号量，阻塞直到写者完成操作，由写者唤醒阻塞在信号量中的读者。
- 如果大于等于0则说明当前没有写者，原子加记录读者数量后即可立即返回

至于为什么从`readerCount`来判断是否有写者？可以带着疑问来看下一节中写锁实现。

### RUnlock
```go
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        // Outlined slow-path to allow the fast-path to be inlined
        rw.rUnlockSlow(r)
    }
}

func (rw *RWMutex) rUnlockSlow(r int32) {
    // A writer is pending.
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // The last reader unblocks the writer.
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```
读锁释放需要判断`r := atomic.AddInt32(&rw.readerCount, -1)`原子减后`readerWait`是否小于0：
- 如果小于0说明当前有写者处于等待队列中，由最后的读者来唤醒等待中的写者。
- 如果大于等于0则说明无写者等待，因此原子减读者数量后即可立即返回。

`readerWait`变量的使用同样需要结合写锁的实现来仔细学习。


### 小结
在使用读锁时，会根据`readerCount`的值来判断当前是否有写者处于等待状态中，如果没有写者，那么直接通过quick path的原子操作修改读者数量即可返回。而如果发现有写者等待的话，则得通过slow path唤醒写者（RUnlock）或读者进入等待状态（RLock）

## 写锁
写者的实现则相对复杂，与读锁类似，同样分为slow path和quick path。

### Lock
```go
func (rw *RWMutex) Lock() {
    // First, resolve competition with other writers.
    rw.w.Lock()
    // Announce to readers there is a pending writer.
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // Wait for active readers.
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```
对于写锁来说，必须得跟所有其他锁互斥，因此第一步便是取得互斥锁，将其他写者阻塞。

随后则是一条十分重要的语句
```go
r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
```

这条语句需要拆分成两部分看，前一部分为原子减，减去了常量`rwmutexMaxReaders=1<<30`，减去这一步的目的是使得`readerCount`一定远远小于0，以此来表示写者**正在获取锁的状态**。这也是为什么在`RLock()`中根据`readerCount+1`后是否小于0可以判断当前是否已存在执行中的写者，以便将锁交给写者。

后半部分则是将常量加回表示当前活跃读者数量的快照，由于可能会有读者释放读锁，因此变量`r`作为当前的快照是可能比实际数量大的。因此如果快照`r`比实际读者数量大，则说明当前有读者正在释放读锁。

随后则通过快照`r`的值来判断当前是否存在已获取读锁的活跃读者，并通过`readerWait`来判断实际活跃的读者。此时，`readerWait`处于[-r, r]的区间之中，具体的数值取决于读锁的释放时机：
- 假如`readerWait == -r`，这说明在写者获取锁时的活跃读者已将全部读锁释放，因此`readerWait`在原子加后必定为0，写者可以直接获取写锁而无需进入slow path。
- 如果`readerWait`处于(-r, 0)，说明已有部分活跃读者已将读锁释放，但由于仍存在活跃读者，原子加后`readerWait`处于(0, r)中，写者需要通过写者信号量进入slow path等待。
- 如果`readerWait == 0`，说明没有活跃读者释放读锁，原子加后`readerWait`即等于活跃读者数。

### Unlock
```go
func (rw *RWMutex) Unlock() {
    // Announce to readers there is no active writer.
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)

    // Unblock blocked readers, if any.
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // Allow other writers to proceed.
    rw.w.Unlock()
}
```
写锁的释放相对简单，因为写锁一旦获得说明当前只有自身一个写者。首先通过原子加 `readerCount + rwmutexMaxReaders`，来使得新进入的读者可以直接获取读锁。随后则是将阻塞在读者信号量中的读者逐一唤醒，最后释放互斥锁来允许新的写者。

## 总结
`RWMutex` 通过细粒度的状态控制将读写锁实现为quick path和slow path，并且通过原子操作来使得读不加锁，以此最大化读写锁的效率。