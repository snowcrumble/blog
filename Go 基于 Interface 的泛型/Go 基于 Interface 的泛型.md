# Go 基于 Interface 的泛型

Go 泛型的[草案](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md)截止今日已经基本定型了，与其他语言最大的不同应该就是 Go 的泛型利用 Interface 做 Constraint，可以说是与现有的 Interface 充分结合，之前的草案本来要引入新的关键字 contracts 在这次改动后被现有的 interface 代替，这使得 Interface 的概念更像 Rust 的 trait（实际上 Go 的泛型概念也与 Rust 相似），不过 Go 中实现 Interface 不需要显示声明，所以我想先谈一下 Interface 这个 Go 语言中最出色的“发明”。

## Interface

Interface 实现了 Go 风格的 `Duck typing`。它实现的方法查表方式与其他语言有些不同，有方法的语言大概有两个阵营

1. C++ 和 Java 在编译时生成方法的静态方法表，比如 C++ 的 vtable
2. Js 和 Python 动态查询，并花式缓存

Go 是少数取二者之间的做法，在运行时构建方法表，叫做[itable](https://github.com/golang/go/blob/master/src/runtime/iface.go)。为什么要在运行时构建呢，这是由于 Go 的 Interface 区别于其他大多数语言的“接口”有一个最大的不同，就是不需要显式定义类型实现了哪些接口，仅在用时才写到 itable 中，这样就会有大量的隐式实现的接口在运行时通常是用不到的，如果用静态构建方法表的话会造成了大量冗余。你可能想说在编译时可以只把代码中用到的方法编进 itable 就好了啊，但那对编译器来说除了实现难度之外还有指数级的时间复杂度，你应该知道 Go 有多在乎编译速度，比如迟迟没有引入泛型，还有现在的泛型草案不引入可以提高代码可读性的尖括号等等，Go 编译器团队的固执其实是有很大争议的，就像 Rob Pike 在一次专访中说的：“忽略那些讨厌你的人，只倾听那些理解你的目标的声音”，而编译速度可以说是其中一个目标了，所以 Go 当初采用动态方法表也就不难理解了。那动态方法表的性能怎么样？由于 Go 是在用时加载一次，以后都直接查表，所以只有一次初始化开销，时间复杂度是[O(ni+nt)](https://github.com/golang/go/blob/0a820007e70fdd038950f28254c6269cd9588c02/src/runtime/iface.go#L199)，ni 和 nt 分别是 interface 和 struct 的方法数量。如果你对性能很敏感，可以在代码里通过赋值给接口的匿名全局变量，比如：

```go
var _ IterfaceA = &StructB{}
```

这行代码有两个作用，一个是在编译时可以确保 StructB 实现了 IterfaceA，另一个就是启动时写入 itable，当然前者应该是主要作用，一般没人会差这点性能，这个例子可以便于理解 itable 的工作原理。

## 泛型

泛型是一个宽泛的概念，在不同语言里有不同的实现。泛型是实现多态的一种方式，多态可以划分为三种

1. `parametric polymorphism` 参数化多态
2. `Ad-hoc polymorphism` 特设多态
3. `Subtype polymorphism` 子类型多态

Go 的泛型实现的是参数化多态，与 Rust 一样承袭 Haskell，并且不支持模板元编程以及其他形式的编译时编程。

C++模板类型可调用任何方法（在 C++20 加入`concept`之前），会在编译期校验以及报错，但是报错通常会随着调用层级而变得很长导致难以理解，而且修改模板可能会导致很**远**的调用报错。Go 期望作为大型系统的编程语言，各个 package 的依赖层级可能会非常深，所以没有采用 C++模板的方式。“灵活”的 C++可能是 Bjarne Stroustrup 团队非常开放的接受各种提案的后果，这与 Go 团队的保守程度以及对社区提案的接受度形成鲜明对比。客观的说，两种选择各有其优劣，就像皿煮和专政。

Go 的参数多态大体可分为*函数的类型参数*、*类型的类型参数*两种，而类型的方法暂时不会接受类型参数，这是考虑到语言复杂程度和一些未知问题。

### 基本原则

Go 泛型编程的基本原则是：泛型代码只能使用参数类型已实现的操作。
一个最简单的例子：

```go
func Stringify(type T)(s []T) (ret []string) {
    for _, v := range s {
        ret = append(ret, v.String()) //编译失败，不能确定v是否有String()方法
    }
    return ret
}
```

```go
type Stringer interface {
    String() string
}

// 限制类型T必须实现Stringer接口
func Stringify(type T Stringer)(s []T) (ret []string) {
    for _, v := range s {
        ret = append(ret, v.String()) // OK
    }
    return ret
}
```

### chunked any slice

这次修订的草案附带了一个官方的 Playground，我就跑去[试玩](https://go2goplay.golang.org/p/tsvYgOz-fNe)试玩了下，最近开发时正好遇到个蹩脚的问题，就是把一个 slice 切成 n 个 slice 的函数，如果接收参数是`[]interface{}`，那`[]int` 在切块前就要转换为`[]interface{}`，这个时间复杂度和空间复杂的都是 On 的操作不仅浪费性能，而且转换后使用时还要配合`type assertion`这种不优雅的小尾巴，着实令人蛋疼。为了榨干这点性能，只能为每个类型都写一个切块函数，或者用类似[betterGo](https://github.com/PioneerIncubator/betterGo)这种生成工具，但是如果有了泛型，就可以直接抽象了：

```go
package main

import (
    "fmt"
)

// 把objArr按size切块
func splitObjects(type T)(objArr []T, size int) [][]T {
    var chunkSet [][]T
    var chunk []T

    for len(objArr) > size {
        chunk, objArr = objArr[:size], objArr[size:]
        chunkSet = append(chunkSet, chunk)
    }
    if len(objArr) > 0 {
        chunkSet = append(chunkSet, objArr[:])
    }

    return chunkSet
}

type A struct {}

func main() {
    intSlices := splitObjects([]int{1,2,3,4,5,6,7,8,9}, 4)
    for i, s := range intSlices {
        fmt.Printf("chunked slice %d: %+v\n", i, s)
    }

    strSlices := splitObjects([]string{"1","2","3","4","5","6","7","8","9"}, 4)
    for i, s := range strSlices {
        fmt.Printf("chunked slice %d: %+v\n", i, s)
    }

    aSlices := splitObjects([]A{{},{},{},{},{},{},{},{},{}}, 4)
    for i, s := range aSlices {
        fmt.Printf("chunked slice %d: %+v\n", i, s)
    }
}
```

### 类型限制

C++的操作符重载支持几乎全部内置的操作符，滥用操作符重载会带来难以读懂的代码，Go 为了避免这个问题，整理了操作符重载的最常用的场景，通过类型限制列表(type list in constraints)来限制支持比较操作符的类型，这是因为只有 Go 的一些内置类型才支持`>` `<`操作，`==` 和 `!=` 例外。

在之前，Go 的排序需要自己定义类型的 slice 类型，甚至连基础类型 int8/int16/int32/int64 都需要自己定义，比如：

```go
type SortByUint32 []uint32
func (a SortByUint32) Len() int           { return len(a) }
func (a SortByUint32) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a SortByUint32) Less(i, j int) bool { return a[i] < a[j] }
```

如果有泛型，sort 包就可以提供所有基础类型的排序泛型函数，像这样：

```go
package sort

// Ordered 类型限制
type Ordered interface {
    type int, int8, int16, int32, int64,
        uint, uint8, uint16, uint32, uint64, uintptr,
        float32, float64,
        string
}

type orderedSlice(type T Ordered) []T

func (s orderedSlice(T)) Len() int           { return len(s) }
func (s orderedSlice(T)) Less(i, j int) bool { return s[i] < s[j] }
func (s orderedSlice(T)) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

func OrderedSlice(type T Ordered)(s []T) {
    sort.Sort(orderedSlice(T)(s))
}
```

排序代码就可以直接这么写了：

```go
s1 := []int32{3, 5, 2}
sort.OrderedSlice(s1)
// Now s1 is []int32{2, 3, 5}

s2 := []string{"a", "c", "b"})
sort.OrderedSlice(s2)
// Now s2 is []string{"a", "b", "c"}
```

But, 没有操作符重载意味着不能为其他类型在泛型函数中比较，`==` 和 `!=` 除外，可以利用 Go 预定义的 constraints: `comparable`

```go
// Index returns the index of x in s, or -1 if not found.
func Index(type T comparable)(s []T, x T) int {
    for i, v := range s {
        // v and x are type T, which has the comparable
        // constraint, so we can use == here.
        if v == x {
            return i
        }
    }
    return -1
}
```

`comparable`基本满足了实现各种容器可以不需要多余的`type assertion`了，就像[emirpasic/gods](https://github.com/emirpasic/gods)的做法。

另外，操作符重载会大幅降低语言可读性，Go 团队目前也没有打算支持操作符重载，因为有太多不确定性和不可控性，但是他们也考虑到了将来如果要引入操作符重载的话也是会符合 Go 泛型的基本原则的。

### 函数式编程

虽然纯函数式编程的语言 Haskell 并不常用，但是 Scala 甚至 JavaScript 都可以容易的进行函数式编程，而缺少泛型的 Go 对函数式编程的支持是一个很大的遗憾，现在有了泛型，《Learning Functional Programming in Go》这本书就需要出第二版了。

**Map/Reduce/Filter**

MapReduce 是函数式编程最常见的操作，下面是草案里给的一个例子：

```go
// Package slices implements various slice algorithms.
package slices

// Map turns a []T1 to a []T2 using a mapping function.
// This function has two type parameters, T1 and T2.
// There are no constraints on the type parameters,
// so this works with slices of any type.
func Map(type T1, T2)(s []T1, f func(T1) T2) []T2 {
    r := make([]T2, len(s))
    for i, v := range s {
        r[i] = f(v)
    }
    return r
}

// Reduce reduces a []T1 to a single value using a reduction function.
func Reduce(type T1, T2)(s []T1, initializer T2, f func(T2, T1) T2) T2 {
    r := initializer
    for _, v := range s {
        r = f(r, v)
    }
    return r
}

// Filter filters values from a slice using a filter function.
// It returns a new slice with only the elements of s
// for which f returned true.
func Filter(type T)(s []T, f func(T) bool) []T {
    var r []T
    for _, v := range s {
        if f(v) {
            r = append(r, v)
        }
    }
    return r
}
```

调用时：

```go
    s := []int{1, 2, 3}

    floats := slices.Map(s, func(i int) float64 { return float64(i) })
    // Now floats is []float64{1.0, 2.0, 3.0}.

    sum := slices.Reduce(s, 0, func(i, j int) int { return i + j })
    // Now sum is 6.

    evens := slices.Filter(s, func(i int) bool { return i%2 == 0 })
    // Now evens is []int{2}.
```

**构造函数范式**

Go 有一个很常用的构造函数范式，在 gRPC 等开源代码中应用广泛。其思想是利用**Applicative Functor**把构造参数**Currying**，现在有泛型之后可以将 ap 函子抽象出来，像这样：

```go
// A Option sets options.
type Option(type T) interface {
	apply(T)
}

// Option wraps a function that modifies options into an
// implementation of the Option interface.
type applyOption(type T) struct {
	f func(T)
}

func (x *applyOption(T)) apply(do T) {
	x.f(do)
}

//NewApplyOption  create an ap functor
func NewApplyOption(type T)(f func(T)) *applyOption(T) {
	return &applyOption(T){
		f: f,
	}
}
```

以后类型在构造时就不需要重复写上面的代码，直接定义函子：

```go
type options struct {
	cache bool
	size int
}

var defaultOptions = options{}

func Cache() Option(*options) {
	return NewApplyOption(*options)(func(o *options) {
		o.cache = true
	})
}

func Size(size int) Option(*options) {
	return NewApplyOption(*options)(func(o *options) {
		o.size = size
	})
}

type Service struct {
    opts options
}

func New(opt ...Option(*options)) *Service {
    opts := defaultOptions
    for _, o := range opt {
        o.apply(&opts)
    }

    return &Service{
        opts: opts,
    }
}

func main() {
	s := New(Cache(), Size(10))
	fmt.Println(s.opts)
}
```

完整代码见：[The go2go Playground](https://go2goplay.golang.org/p/Uw92pH3DG1y)

### 注意空指针遗毒

我之所以称空指针为“遗毒”是因为我更喜欢 Rust 摒弃空指针的做法，不过等 Go 将来补充 sum types 之后会好一点。

使用 Go 的泛型需要注意空指针的问题，这需要结合 Go 本身的一些既定逻辑

1. 方法接收者是类型或类型指针的区别，其中之一就是方法接收者是类型指针才可以修改非指针的成员变量
2. make 类型指针的 slice 默认不初始化成员，全部是 nil

比如你想写一个 FromStrings 泛型函数，可以从字符串转换为各种类型

```go
type Setter interface {
    Set(string)
}

func FromStrings(type T Setter)(s []string) []T {
    result := make([]T, len(s))
    for i, v := range s {
        result[i].Set(v)
    }
    return result
}
```

但是调用的时候发现无效

```go
type Settable int

func (p *Settable) Set(s string) {
    n, _ := strconv.Atoi(s)
    *p = Settable(n)
}

func main() {
    // 方式①
    // 编译失败: Settable 没有实现Set方法
    nums := FromStrings(Settable)([]string{"1", "2"})

    // 方式②
    // 运行panic: *Settable 实现了Set方法，但 result[i].Set(v) 会造成nil panic
    nums := FromStrings(Settable)([]string{"1", "2"})
}
```

对于这样的需求，需要用到泛型提供的指针类型参数`(type *T Constraint)`，指针类型参数允许在泛型代码里调用类型的指针方法，所以只要将`FromStrings`函数签名改为：

```
FromStrings(type *T Setter)(s []string) []T
```

调用的时候直接用上面代码中的`方式①`即可。

进一步的，如果想返回是类型指针的切片的话，就把返回值也改为类型指针，并改一下实现，代码如下：

```go
func FromStrings(type *T Setter)(s []string) []*T {
    result := make([]*T, len(s)) // make([]*Settable, len(s))
    for i, v := range s {
        t := new(T) // new(Settable)
        t.Set(v)    // *Settable.Set(v)
        result[i] = t
    }
    return result
}
```

### 泛型和 Interface 的区别

泛型和 Interface 本质上就是不同的东西，但有一个细微的区别可以帮助我们更好的理解它们本质的区别。interface 由两个指针组成，一个指向 itable 中的数据的类型，另一个指向数据
![](Go%20%E5%9F%BA%E4%BA%8E%20Interface%20%E7%9A%84%E6%B3%9B%E5%9E%8B/v2-0b17e0be86c6f2b85d12aed262e941a0_1440w.jpg)
当你将一个类型实例赋值给 Interface 时，其实是一个”拷贝构造“，Interface 会开辟一块堆内存，将 data 指针指向那块堆内存，除非类型实例本身是一个`one word`，比如指针。换句话说，Interface 的 data 只保留一个指针，这个概念叫做 boxed。与 Interface 不同，泛型不会 boxed，类型实例赋值给一个类型参数实际上是编译器将其翻译为具体的类型，比如：

```go
type GenericsType(type T1, T2) struct {
    a  T1
    b  T2
}

var v GenericsType(int, string)
// 实际上等于 struct { a int; b string }
```

换句话说，Go 泛型的类型参数只能代表类型，不能携带对象，否则就属于模板了。

## 结尾

Go 泛型的最新草案体现了 Go 团队想要基于老版本进行最小的改动，而最大限度的复用现有 Go 的特点，这体现了一个语言设计上的一致性，与 Swift 语法频繁变更形成鲜明对比。不考虑模板元编程和操作符重载也说明他们不想让 Go 变得不可控，甚至造成语言的”割裂“，因为 C++的模板元编程是一个意外发现并且是图灵完备的，与 C++本身某种程度成了两种语言。Go 团队对各种激进的建议持有非常保守的态度对语言的稳定和可持续发展有很重要的意义。这算是最后一版草案了，Go 团队现在已经开始按照这个草案开发了，到完成以前可能还会有一些改动，但大框架估计不会有改动了，顺利的话再过一年就可以用上了。

## Reference

[research!rsc: Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
[Type Parameters - Draft Design](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md)
