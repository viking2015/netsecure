[TOC]
### sliece 底层实现
文件 src/runtime/slice.go
```
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
创建切片：
func makeslice(et *_type, len, cap int) slice {
    // 根据切片的数据类型，获取切片的最大容量
    maxElements := maxSliceCap(et.size)
    // 比较切片的长度，长度值域应该在[0,maxElements]之间
    if len < 0 || uintptr(len) > maxElements {
        panic(errorString("makeslice: len out of range"))
    }
    // 比较切片的容量，容量值域应该在[len,maxElements]之间
    if cap < len || uintptr(cap) > maxElements {
        panic(errorString("makeslice: cap out of range"))
    }
    // 根据切片的容量申请内存
    p := mallocgc(et.size*uintptr(cap), et, true)
    // 返回申请好内存的切片的首地址
    return slice{p, len, cap}
}
```
### nil和空切片
nil切片和空切片在使用上没有区别， 因为都可以 做apppend, 
nil切片和空切片也都是常用的。
下面的声明得到一个nil切片：
var slice []int
nil 切片被用在很多标准库和内置函数中，描述一个不存在的切片的时候，就需要用到 nil 切片。
比如函数在发生异常的时候，返回的切片就是nil切片。nil切片的指针指向nil。

空切片一般会用来表示一个空的集合。比如数据库查询，一条结果也没有查到，那么就可以返回一个空切片。
下面的声明得到一个空切片：
```
silce := make( []int , 0 )
slice := []int{ }
```

空切片和nil切片的区别在于：
空切片指向的地址不是nil，指向的是一个内存地址，但是它没有分配任何内存空间，即底层元素包含0个元素。
空切片和nil切片的相同点：
不管是使用nil切片还是空切片，对其调用内置函数append，len和cap的效果都是一样的。

### 扩容
扩容都是放弃原有内存空间，重新开辟一块连续新的内存空间
如果切片的容量小于1024个元素，于是扩容的时候就翻倍增加容量。
一旦元素个数超过 1024 个元素，那么增长因子就变成 1.25 ，即每次增加原来容量的四分之一。
函数：
```
func growslice(et *_type, old slice, cap int) slice {
    if raceenabled {
        callerpc := getcallerpc(unsafe.Pointer(&et))
        racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
    }
    if msanenabled {
        msanread(old.array, uintptr(old.len*int(et.size)))
    }
 
    if et.size == 0 {
        // 如果新要扩容的容量比原来的容量还要小，这代表要缩容了，那么可以直接报panic了。
        if cap < old.cap {
            panic(errorString("growslice: cap out of range"))
        }
 
        // 如果当前切片的大小为0，还调用了扩容方法，那么就新生成一个新的容量的切片返回。
        return slice{unsafe.Pointer(&zerobase), old.len, cap}
    }
 
  // 这里就是扩容的策略
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap {
        newcap = cap
    } else {
        if old.len < 1024 {
            newcap = doublecap
        } else {
            for newcap < cap {
                newcap += newcap / 4
            }
        }
    }
 
    // 计算新的切片的容量，长度。
    var lenmem, newlenmem, capmem uintptr
    const ptrSize = unsafe.Sizeof((*byte)(nil))
    switch et.size {
    case 1:
        lenmem = uintptr(old.len)
        newlenmem = uintptr(cap)
        capmem = roundupsize(uintptr(newcap))
        newcap = int(capmem)
    case ptrSize:
        lenmem = uintptr(old.len) * ptrSize
        newlenmem = uintptr(cap) * ptrSize
        capmem = roundupsize(uintptr(newcap) * ptrSize)
        newcap = int(capmem / ptrSize)
    default:
        lenmem = uintptr(old.len) * et.size
        newlenmem = uintptr(cap) * et.size
        capmem = roundupsize(uintptr(newcap) * et.size)
        newcap = int(capmem / et.size)
    }
 
    // 判断非法的值，保证容量是在增加，并且容量不超过最大容量
    if cap < old.cap || uintptr(newcap) > maxSliceCap(et.size) {
        panic(errorString("growslice: cap out of range"))
    }
 
    var p unsafe.Pointer
    if et.kind&kindNoPointers != 0 {
        // 在老的切片后面继续扩充容量
        p = mallocgc(capmem, nil, false)
        // 将 lenmem 这个多个 bytes 从 old.array地址 拷贝到 p 的地址处
        memmove(p, old.array, lenmem)
        // 先将 P 地址加上新的容量得到新切片容量的地址，然后将新切片容量地址后面的 capmem-newlenmem 个 bytes 这块内存初始化。为之后继续 append() 操作腾出空间。
        memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
    } else {
        // 重新申请新的数组给新切片
        // 重新申请 capmen 这个大的内存地址，并且初始化为0值
        p = mallocgc(capmem, et, true)
        if !writeBarrier.enabled {
            // 如果还不能打开写锁，那么只能把 lenmem 大小的 bytes 字节从 old.array 拷贝到 p 的地址处
            memmove(p, old.array, lenmem)
        } else {
            // 循环拷贝老的切片的值 
            for i := uintptr(0); i < lenmem; i += et.size {
                typedmemmove(et, add(p, i), add(old.array, i))
            }
        }
    }
    // 返回最终新切片，容量更新为最新扩容之后的容量
    return slice{p, old.len, newcap}
}
```

### c++的动态数组与go语言的切片

数组是一种顺序存储的数据结构，在定义数组时，首先要确定数组的大小。静态数组在编译时就需要确定数组的大小，所以，为了防止内存溢出，我们尽量将数组定义的大一些，但是这样太过浪费内存。
C++的动态数组， 不是在编译时确定大小，而是在运行时确定。
go语言的切片（slice）是对数组一个连续片段的引用，所以切片是一个引用类型（因此更类似于 C/C++ 中的数组类型，或者 Python 中的 list 类型）。
这个片段可以是整个数组，或者是由起始和终止索引标识的一些项的子集。
终止索引标识的项不包括在切片内。切片提供了一个与指向数组的动态窗口。

给定项的切片索引可能比相关数组的相同元素的索引小。和数组不同的是，切片的长度可以在运行时修改，最小为 0 最大为相关数组的长度：切片是一个长度可变的数组。


### 两个数组元素加在1个数组的快捷方法：
```
a:=[]interfere{}{5,6,9,3}    b:=[]int{9,8,3,4}
a=append(a, b...) 
```
b后面加...，意思就是把b的元素展开来，一个一个加到a里面
输出结果是
[5,6,9,3,9,8,3,4]

如果不加...， 即a=append(a, b) 输出结果是
[ [5,6,9,3], [9,8,3,4] ],就变成2个数组了

---
slice，len，cap，共享，扩容
append函数，因为slice底层数据结构是由数组、len、cap组成，所以，在使用append扩容时，会查看数组后面有没有连续内存快，有就在后面添加，没有就重新生成一个大的数组， 这个大一点的数组是完全新分配的， 所以这样的append返回的地址已经不是原slice的地址