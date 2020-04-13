---
title: Go Slice
tags: Go
key: All-About-Slice
modify_date: 2020-04-13 22:50:19 
---

# Go Slice
slice在Go中扮演着动态数组的角色，有着广泛的使用。并且由于Go值传递的特性，通常使用slice会比使用定值数组Array带来更多的便利。接下来，本文将会从slice的实现出发来介绍slice中隐藏的Tricks。

<!--more-->
## 数据结构
Go程序在编译时会将一些内置方法进行转换调用实际的runtime函数，slice相关的的runtime代码位于[`runtime.slice`](https://github.com/golang/go/blob/release-branch.go1.14/src/runtime/slice.go)。

slice的关键结构如下
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
其中`array`对应着底层数组的指针，`len`代表切片的长度，`cap`则是底层数组从切片起始点开始的容量，在64位中总共是24个字节的存储。从下面一图可以清晰地明白slice的结构表示：
<div style="text-align: center">
<img title="Slice Internal" src="/assets/All-About-Slice/All-About-Slice2020-04-13T16-21-37.png">
</div>

## 切片操作
### 创建
切片主要有三种方式进行初始化：
```go
// 1. 使用字面量直接定义切片
slice := []int{1,2,3,4} 

// 2. 通过make函数来创建
slice := make([]int, 8)

// 3. 基于已存在的切片或数组
array := [8]int{1,2,3,4,5,6,7,8}
slice1 := array[2:6] // [3 4 5 6]
slice2 := slice1[2:] // [5 6]
slice3 := slice2[:] // [5 6]
```
这些方法在编译时期都会创建新的切片结构体，并将对应的底层数组指针放入其中。


### 访问
切片的访问即对应着底层数组元素的访问，编译时会将可识别的索引直接转换为底层数组对应位置的指针进行访问，不可识别的索引则是在运行时计算并添加索引边界检查。

由于切片的修改对应着底层元素的修改，因此如果有多个切片对应着同一个底层数组，那么一个切片的修改会同步到其他切片中，如下：
```go
array := [8]int{1,2,3,4,5,6,7,8}
slice1 := array[2:6] // [3 4 5 6]
slice2 := slice1[2:] // [5 6]
slice2[0] = 0
fmt.Println(slice2[0] == slice1[2]) // true
fmt.Println(slice2[0] == array[4]) // true
// array = {1,2,3,4,0,6,7,8}
```


### 复制
内置方法`func copy(dst, src []Type) int`为切片间的复制提供了遍历的操作，其底层主要通过汇编编写的[`memmove()`](https://github.com/golang/go/blob/release-branch.go1.14/src/runtime/stubs.go#L98)函数来实现，兼容了前向拷贝与后向拷贝防止同数组复制下的覆盖，例如；

```go
array := [...]int{1,2,3,4,0,0}
copy(array[2:], array[:])	
fmt.Println(array) // [1 2 1 2 3 4]

array := [...]int{0,0,1,2,3,4}
copy(array[:], array[2:])	
fmt.Println(array) // [1 2 3 4 3 4]
```

### 追加与扩容
作为动态数组，经常有动态扩容的需求，Go中通过内置方法`func append(slice []Type, elems ...Type) []Type`来返回追加后的新切片。

在编译时期，Go会根据原切片是否被覆盖而分为[两种流程](https://github.com/golang/go/blob/release-branch.go1.14/src/cmd/compile/internal/gc/ssa.go#L2733-L2885)：

第一种是未对原切片进行覆盖的流程如， `append(s, e1, e2, e3)`，
```go
    ptr, len, cap := s
    newlen := len + 3
    if newlen > cap {
        ptr, len, cap = growslice(s, newlen)
        newlen = len + 3 
    }
    
    *(ptr+len) = e1
    *(ptr+len+1) = e2
    *(ptr+len+2) = e3
    return makeslice(ptr, newlen, cap)
```  
如果新长度大于当前切片的容量，那么会调运行时函数[`runtime.growslice()`](https://github.com/golang/go/blob/release-branch.go1.14/src/runtime/slice.go#L76-L191)开辟一个新的数组，并通过`memmove()`将数据转移，随后基于新数组来创建新的切片。

另一种则是对原切片进行覆盖的流程，如`s = append(s, e1, e2, e3)`。这种方法与上一种方法的主要差异在于这种方法会直接对原切片进行修改，而不是创建一个新的切片。
```go
    a := &s
    ptr, len, cap := s
    newlen := len + 3
    if uint(newlen) > uint(cap) {
       newptr, len, newcap = growslice(ptr, len, cap, newlen)
       vardef(a)       
       *a.cap = newcap 
       *a.ptr = newptr 
    }
    newlen = len + 3 
    *a.len = newlen

    *(ptr+len) = e1
    *(ptr+len+1) = e2
    *(ptr+len+2) = e3
```

**扩容**

了解了追加的方式后，接下来看看切片是如何实现扩容的。

首先`growslice()`会根据`append()`传入的所需容量`cap`来计算新切片的容量：
- 如果所需切片的容量大于旧切片的两倍容量，那么直接使用所需切片容量
- 如果旧切片的长度小于1024，那么使用旧切片的两倍容量
- 如果旧切片的长度大于1024，那么每次递增25%直到满足需求


```go
func growslice(et *_type, old slice, cap int) slice {
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            newcap = doublecap
        } else {
            for 0 < newcap && newcap < cap {
                newcap += newcap / 4
            }
            // Set newcap to the requested cap when
            // the newcap calculation overflowed.
            if newcap <= 0 {
                newcap = cap
            }
        }
    }
```

确定了切片的需求长度之后，便可以计算数组实际占用的空间大小。如果要求的空间过大，则会直接panic终止程序

```go
    var overflow bool
    var lenmem, newlenmem, capmem uintptr

    switch {
    case et.size == 1:
        ...
    case et.size == sys.PtrSize:
        ...
    case isPowerOfTwo(et.size):
        ...
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
        capmem = roundupsize(capmem)
        newcap = int(capmem / et.size)
    }

    if overflow || capmem > maxAlloc {
        panic(errorString("growslice: cap out of range"))
    }

```

确定了所需的空间大小之后，便通过`mallocgc()`进行实际的内存申请。在申请空间时，会根据数组的使用场景来判断新数组是否需要被清零。如果是追加型，那么`newlenmem`之前的内存都会被追加覆盖而不需要额外的清零。
```go
    var p unsafe.Pointer
    if et.ptrdata == 0 {
        p = mallocgc(capmem, nil, false)
        // The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
        // Only clear the part that will not be overwritten.
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        p = mallocgc(capmem, et, true)
        ...
    }
    memmove(p, old.array, lenmem)

    return slice{p, old.len, newcap}
```

## 使用技巧
上文主要介绍了切片相关的操作实现，了解这些内部实现有助于我们对切片的使用，并且避免一些常见的坑。接下来将会结合上文讲的操作实现来看看一些具体的Tricks。

### 多维切片
简单的多维切片可以通过以下方式创建
```go
// slices[n][m]
slices := make([][]int, n)
for i := 0; i < n; i++ {
    slices[i] = make([]int, m)
}
```
另一种方式则是分割一维数组
```go
ar := make([]int, n*m)
slices := make([][]int, n)
for i := 0; i < n; i++ {
    slices[i] = ar[i*m:(i+1)*m]
}
```

通常来说如果多维数组在使用的过程中每一行不需要增长的话，第二种方式会比较高效。这是因为第二种方式的多维切片的底层数组是连续数组，对于CPU cache更加友好。而如果每一行经常需要变更大小的话，则是使用第一种方式以获得更好的灵活性。



### 值传递
由于Go中的函数传参为**值传递**而不是引用传递，因此在函数传递的过程中，调用方会将参数拷贝一份在函数栈上，因此函数中的参数地址与调用方的参数地址是不同的。对应切片的传递，可能会出现一些较难理解的现象。接下来我们通过一些例子来看看这些现象：

第一个例子对应着在函数中传递切片，并在函数中改变切片元素。首先可以看到在`changeSlice()`函数中的切片`s1`与调用方`main()`中的切片`s0`地址并不相同，可以印证这两个切片是不同的对象。随后在`changeSlice()`对`s1[0]`进行修改，可以看到切片修改直接影响底层数组，因此函数内外的切片数组值是相同的。
```go
func changeSlice(s1 []int) {
    fmt.Printf("changeSlice: val=%v address=%p \n", s1, &s1)
    // changeSlice: val=[1 2 3] address=0xc0000964a0
    s1[0] = 0
    fmt.Printf("changeSlice: val=%v address=%p \n", s1, &s1)
    // changeSlice: val=[0 2 3] address=0xc0000964a0
}

func main() {
    s0 := []int{1, 2, 3}
    fmt.Printf("slice val=%v address=%p \n", s0, &s0)
    // slice val=[1 2 3] address=0xc000096440
    changeSlice(s0)
    fmt.Printf("slice val=%v address=%p \n", s0, &s0)
    // slice val=[0 2 3] address=0xc000096440   
    // 修改在底层数组，因此切片的数组值一致
}
```

第二个例子对应着在函数中对切片进行扩容。可以看到，在函数中追加扩容后的切片与main函数中的切片数组值是不相同的。从append扩容的行为来看，我们知道`appendSlice()`中的切片`s1`更新了底层指向的数组，但是由于切片`s0`和切片`s1`不是相同的对象，因此二者在`s1`扩容后指向的底层数组是不相同的。

```go
func appendSlice(s1 []int) {
    fmt.Printf("appendSlice: val=%v address=%p \n", s1, &s1)
    // appendSlice: val=[1 2 3] address=0xc000004500
    s1 = append(s1, 4)
    fmt.Printf("appendSlice: val=%v address=%p \n", s1, &s1)
    // appendSlice: val=[1 2 3 4] address=0xc000004500
}

func main() {
    s0 := []int{1, 2, 3}
    fmt.Printf("slice val=%v address=%p \n", s0, &s0)
    // slice val=[1 2 3] address=0xc0000044a0
    appendSlice(s0)
    fmt.Printf("slice val=%v address=%p \n", s0, &s0)
    // slice val=[1 2 3] address=0xc0000044a0
    // s1中改变了底层数组指针，而s0的底层数组指针不变
}
```

从第二个例子可以看到，直接传递切片时，如果函数内有可能造成切片扩容，那么函数内外的切片数组值是不相同的。如果想要函数内外的切片数组值始终一致，那可以通过传递切片指针的方式来达成。

```go
func appendSlicePtr(s1 *[]int) {
    fmt.Printf("appendSlicePtr: val=%v address=%p \n", *s1, s1)
    // appendSlicePtr: val=[1 2 3] address=0xc0000044a0
    *s1 = append(*s1, 4)
    fmt.Printf("appendSlicePtr: val=%v address=%p \n", *s1, s1)
    // appendSlicePtr: val=[1 2 3 4] address=0xc0000044a0
}

func main() {
    s0 := []int{1, 2, 3}
    fmt.Printf("slice val=%v address=%p \n", s0, &s0)
    // slice val=[1 2 3] address=0xc0000044a0
    appendSlicePtr(&s0)
    fmt.Printf("slice val=%v address=%p \n", s0, &s0)
    // slice val=[1 2 3 4] address=0xc0000044a0
}
```

从代码输出可以看到，通过传递切片指针的方式可以保证函数内外修改的始终是一个切片对象，因此对底层数组的修改会在函数内外同步。


## 总结
切片在使用过程中十分灵活，但一不小心也容易造成一些坑。但只要我们对切片的底层实现了然于胸，就可以尽量地避免踩坑并且灵活运用。


## Reference
- [SliceTricks](https://github.com/golang/go/wiki/SliceTricks)
- [Go Slices: usage and internals](https://blog.golang.org/slices-intro)
- [Draveness：Go 语言设计与实现](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/)