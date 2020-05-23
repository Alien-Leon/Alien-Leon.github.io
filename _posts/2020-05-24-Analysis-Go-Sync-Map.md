---
title: 浅析 sync.Map
tags: go 并发 源码分析
key: Analysis-Go-Sync-Map
modify_date: 2020-05-24 01:22:43 
---

# 浅析 sync.Map 
[`sync.Map`](https://github.com/golang/go/blob/release-branch.go1.14/src/sync/map.go)提供并发安全的泛型Map结构。尽管`sync.Map`能够提供并发安全的机制，但在实际使用中，`sync.Map`通常都不是首选。

首先，并发安全的Map可以选择使用`Mutex`或者`RWMutex`互斥访问，搭配`Mutex`和`RWMutex`可以满足大多数并发场景，并且能提供较好的类型约束，因此在实践上是首选。那么什么时候该使用`sync.Map`呢？要了解`sync.Map`的使用场景，就需要从`sync.Map`的设计目标出发。

<!--more-->

## 设计目标
在[GopherCon 2017 - Lightning Talk: Bryan C Mills - An overview of sync.Map](https://www.youtube.com/watch?v=C1EtfDnsdDs)中提到一种场景：在核数较多的情况下使用`RWMutex`，多核同时访问同一键值对时，由于频繁的`cache contention (缓存竞争)`，会使得原子操作`AddUint32`运行缓慢，因此`RWMutex`在核数较多的情况下得不到相应的性能提升。

<div style="text-align: center">
<img title="why RWMutex is slow?" src="/assets/Analysis-Go-Sync-Map/Analysis-Go-Sync-Map-2020-05-15T19-52-51.png">
</div>

为了尽可能地提升多核场景下的读取效率，需要尽量避免昂贵的原子加操作，这就促使了`sync.Map`的开发。

## 使用场景
`sync.Map`对以下特定情况做了优化：
- 每个key只写一次，但会读取多次，即读多写少的情况
- 并发的Goroutine同时读写不同的key集（防止`伪共享`带来的缓存竞争）

如果不是上述特殊的情况，应使用`Mutex`或`RWMutex`来实现并发安全的map。一方面，使用锁来包装的map可以获得更好的类型安全性；另一方面，`sync.Map`无法在遍历时提供一个一致性的快照，并且遍历时不管用户是否提前退出遍历循环，`Range`都有着O(n)的时间复杂度。

<div style="text-align: center">
<img title="Performance of sync.Map vs builtin map(RWMutex)" src="/assets/Analysis-Go-Sync-Map/Analysis-Go-Sync-Map-2020-05-24T00-06-45.png">
</div>

文章[The new kid in town — Go’s sync.Map](https://medium.com/@deckarep/the-new-kid-in-town-gos-sync-map-de24a6bf7c2c)中展示了在多核情况下`sync.Map`和`map(RWMutex)`对稳定键访问的速度，可以看到尽管`sync.Map`在核数少时不如`map(RWMutex)`，但是核数多的情况下，`map(RWMutex)`由于`ADD`原子操作而无法提高性能，`sync.Map`由于避免了昂贵的原子加操作，在核数较多的情况下能有效提升性能。

了解设计目标和使用场景后再来看看实际的实现。

## 数据结构
```go
type Map struct {
    mu Mutex    
    read atomic.Value // type: readOnly，存储着改动较少的数据，无锁访问。
    dirty map[interface{}]*entry  // read的拷贝，存储着最新数据，需要带锁访问
    misses int  // 记录带锁访问dirty中的次数，次数达到阈值后会将dirty转换为read
}

// readOnly is an immutable struct stored atomically in the Map.read field.  
type readOnly struct {  
   m       map[interface{}]*entry  
   amended bool // true if the dirty map contains some key not in m.  
}  
  
// 特殊的状态标记指针，标记已从dirty map中删除的entries
var expunged = unsafe.Pointer(new(interface{}))  
  
// An entry is a slot in the map corresponding to a particular key.  
type entry struct {  
    // p points to the interface{} value stored for the entry.

    // An entry can be deleted by atomic replacement with nil: when m.dirty is
    // next created, it will atomically replace nil with expunged and leave m.dirty[key] unset.
    //
    // An entry's associated value can be updated by atomic replacement, provided
    // p != expunged. If p == expunged, an entry's associated value can be updated
    // only after first setting m.dirty[key] = e so that lookups using the dirty map find the entry.
    p unsafe.Pointer // *interface{}  
}

// 指向val的包装器
func newEntry(i interface{}) *entry {
    return &entry{p: unsafe.Pointer(&i)}
}
```

> :memo: `atomic.Value`支持值的原子存取。

`entry`结构体尽管只有一个字段，但是指针的值会用于表示状态会比较复杂：
- p == nil: entry已被删除且 map.dirty == nil
- p == expunged: entry已被删除，并且相应值不存在于map.dirty中
- 否则，entry未被删除，存在于map.read并且存在于map.dirty中如果其不为nil


## Load
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 双重检验 1：read中无法取得数据且read.amended为true时说明dirty中有read中未拥有的数据
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        // 带锁访问dirty
        m.mu.Lock()
        
        // 双重检验 2：防止阻塞在mutex时，dirty中的数据已经被转移到read中
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            // 从dirty中取得最新的数据
            e, ok = m.dirty[key]
            
            // 记录带锁访问dirty的次数，判断是否需要将dirty转移到read
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load()
}

// 从map[key]对应的slot中取得value
func (e *entry) load() (value interface{}, ok bool) {
    p := atomic.LoadPointer(&e.p)
    if p == nil || p == expunged {
        return nil, false
    }
    return *(*interface{})(p), true
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    // 达到访问阈值，转移m.dirty至m.read，原先的m.read交由垃圾回收处理
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```
流程小结：
1. 通过双重检验从read中看是否能读取数据，其中第二层检验的原因是防止阻塞在mutex时，dirty中的数据已经被转移到read中
2. 无法从read中读取时选择从dirty中取出数据，并记录带锁访问dirty的次数以判断是否需要将dirty转移到read，转移之后read中的数据变为最新，可以减少带锁访问dirty的次数。


## Store
对于Store来说分为直接对`read`写入的`quick path` 和对`dirty`写入的`slow path`，首先看看`quick path`的实现：
```go
func (m *Map) Store(key, value interface{}) {
    // quick path
    read, _ := m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // slow path
    ...
}

// tryStore stores a value if the entry has not been expunged.
//
// If the entry is expunged, tryStore returns false and leaves the entry
// unchanged.
func (e *entry) tryStore(i *interface{}) bool {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
    }
}
```
`quick path`中首先会检查key是否已在read中存在，如果存在则会判断`read.m[key]`是否已删除，若未删除则通过`CAS`来对`read.m[key]`的值进行覆盖。

接下来看看`slow path`：
```go
func (m *Map) Store(key, value interface{}) {
    // quick path
    ...
    
    // slow path
    m.mu.Lock()
    // 防止阻塞在mutex时，dirty中的数据已经被转移到read中
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok {
        if e.unexpungeLocked() {
            // The entry was previously expunged, which implies that there is a
            // non-nil dirty map and this entry is not in it.
            m.dirty[key] = e
        }
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        e.storeLocked(&value)
    } else {
        if !read.amended {
            // We're adding the first new key to the dirty map.
            // Make sure it is allocated and mark the read-only map as incomplete.
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

// unexpungeLocked ensures that the entry is not marked as expunged.
//
// If the entry was previously expunged, it must be added to the dirty map
// before m.mu is unlocked.
func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

// The entry must be known not to be expunged.
func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }
    // 从m.read.m 拷贝非空键 m.dirty
    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        // 重要：如果read.m[key]已被删除，那么需要标记从nil标记为expunged，表示之后对key的读写会转移到dirty中
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}
```

slow path 中分为3种情况
1. key存在于read中
    - 如果`read.m[key]`已删除，需要将覆盖的val写入到dirty中
    - 将val覆盖到对应的`read.m[key]`中
3. key已经存在于dirty中，直接写入dirty
4. key既不存在read中，也不存在dirty中
    - 如果`read.amened`为false，说明dirty未被创建（或者已经转移到read），从read中拷贝快照到dirty中并标记`read.amened`，并且通过`tryExpungeLocked`将read中已删除的key标记为`expunged`，保证之后对该key的读写会转移到dirty中 
    - 将`newEntry(value)`写入`m.dirty[key]`


## Delete
```go
// Delete deletes the value for a key.
func (m *Map) Delete(key interface{}) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            delete(m.dirty, key)
        }
        m.mu.Unlock()
    }
    if ok {
        e.delete()
    }
}

func (e *entry) delete() (hadValue bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        // 本身已经为空，删除失败
        if p == nil || p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```
删除操作与访问操作类似。通过双重检验，找到key所在的位置，如果是在dirty中，则通过map的delete操作直接删除dirty中的key；如果是在read中，则通过`*entry.delete()`以指针状态的形式标记删除位。

## Range
```go
func (m *Map) Range(f func(key, value interface{}) bool) {
    read, _ := m.read.Load().(readOnly)
    if read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        if read.amended {
            read = readOnly{m: m.dirty}
            m.read.Store(read)
            m.dirty = nil
            m.misses = 0
        }
        m.mu.Unlock()
    }
    // 简单遍历
    for k, e := range read.m {
        v, ok := e.load()
        if !ok {
            continue
        }
        if !f(k, v) {
            break
        }
    }
}
```
`Range`分为两部分，关键在于第一部分中对`read`的检查。如果`read`处于`amended`状态，那么为了保证能遍历所有的键值对，会将`dirty`转移为`read`；第二部分则是对应简单的遍历，由于`read`可能会被修改，因此遍历时会读取到最新的修改。

## 总结
`Map`中通过对数据的冗余存储，将数据分布在`read`和`dirty`中。其中`read`对应着`readOnly`结构，但这并不意味着`read`为只读结构，如果是对read中已有的键进行增删查改，首先都会优先修改`read`。如果是新增键的话，则会对`dirty`进行修改。在访问过程中，如果需要带锁到`dirty`中寻找数据时，会计算访问次数，并在次数达到阈值后将`dirty`转移到`read`中。

从行为上看，如果对map的使用是频繁增删查改键值，那么Map的性能不如直接使用`Mutex`的性能好；如果是读多写少并且多核下未达到读写瓶颈，也不如直接使用`RWMutex`的好。因此在选择使用`sync.Map`的时候一定要注意其优化的场景和相应的缺点。

从实现思想来看，整体思想类似于垃圾回收中的分代回收，`read`中是较少改动的键值对，`dirty`中存储着`read`的副本，并且附带最近改动过的且还有可能继续改动的键值对。那么由于`read`中是较少改动，对其的读写可以不带锁，使用CAS操作以提高整体性能；而`dirty`中对应频繁修改，那么带锁访问可拥有更好的性能。


## Reference
- [The new kid in town — Go’s sync.Map](https://medium.com/@deckarep/the-new-kid-in-town-gos-sync-map-de24a6bf7c2c)
- [GopherCon 2017 - Lightning Talk: Bryan C Mills - An overview of sync.Map](https://www.youtube.com/watch?v=C1EtfDnsdDs)