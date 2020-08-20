# Optional，错误处理，部分更新

## Go2 提案

前几天无意间看到 Go2 有一个关于语言修改的[提案](https://github.com/golang/go/issues/19412)，增加 sum types 或者叫 union types，与 Protobuffer 中的 oneof 语义很像，基本概念是定义一个新的类型，为其他类型之和，用`|`分割不同的类型，组合起来的类型 type 只能是各个类型的其中一个。比如

```go
type maybeInt nil | int
```

代表类型 maybeInt 要么是 nil 要么是 int，再比如

```go
var x int|float64 = 13
```

代表类型 x 要么是 int 要么是 float64
这其实是 go 相比很多其他语言缺失的语义表达，比如 c++、java、rust、typescript 都有的 optional 类型，*巧合*的是，与 go 同门的 dart 也没有 sum types，我该说这是 Google 的开发理念一致性贯彻得真彻底吗。这种语义可以比 tuple 更优雅的或者更精准的表达单返回值代替多返回值的意图，比如错误处理。

## 错误处理

go 的设计是倾向用多返回值表达函数的意图，由此引出的差异是与单返回值不同的错误处理方式与理念，这可能也是 go 没有设计 optional 类型的一个原因，从某种角度看来这好像是一种“硬拗”，另外作为静态语言，c++、java、rust 的 optional 都是利用泛型来实现的，而 Go 没有泛型，这可能是另外一个原因，别问，再问就是因为泛型会降低编译速度。go2 这个提案里的 sum types 采用的表达方式跟 typescript 的语法差不多，只不过后者需要额外兼容 js 坑爹的 undefined，这也是由于 typescript 的定位是要做 js 的超集所以无法避免的。

go 的错误处理自有它的理念，但也有它的问题，通常开发者会遵循一条原则——如果返回 error，则返回值为空值，如果返回值非空，则需要返回 error。这条原则带来的好处是函数返回值表达的意图组合由四个简化为两个，这种返回值和 error 之间的“零和博弈”，再适合用 optional 不过了。所以在我看来多返回值和错误处理并不是很搭，反倒是 rust 利用多返回值来表达[descructure](https://doc.rust-lang.org/stable/rust-by-example/flow_control/match/destructuring/destructure_structures.html)显得更恰如其分。用 optional 做错误处理既降低开发写出烂代码的可能性，也提高了代码的可读性。

## REST API

无论什么原因，当你遇到一个 user story 是用 go 写一个解析 JSON 请求的 REST API ，同时想要支持部分更新的话，optional 类型就可以比较优雅的解决，而 Go 就显得捉襟见肘，要么给每个字段都改为指针或者封装成结构体，要么用[Google FieldMask](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#fieldmask)做部分响应、部分更新的思路，从表达力上来看，前者算是一种隐式的表达，需要开发者知道空指针 nil 的业务含义，而后者则是显示的表达，职责更明确，不过这两种方式目前都不能完美解决我的需求。
我在最近的开发中就遇到了这个问题，如果前一种方式，我们现行的接口代码自动生成工具套件 Protobuffer 支持的并不完整，Protobuffer 定义时可以引用一个[wrapper](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/wrappers.proto)，包含了对 go 基础类型的封装，这样接口中的每个基础类型就变成了一个结构体指针，但是对于自定义枚举类型则没办法适配，这可能是 go 没有把枚举类型作为亲儿子的另一个代价，毕竟 rust 的枚举已经高级到可以嵌套元组、结构体了，go 的枚举连个关键字都不给。扯远了，如果用第二种 FieldMask 的思路做的话，我们用的[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)会在解析 JSON 转 pb 时填充好这个字段，只需要我们在用 go 改数据库的代码处对此字段进行解析并组装部分更新的 SQL 即可。但是另一个问题又出现了，我们用[envoyproxy/protoc-gen-validate](https://github.com/envoyproxy/protoc-gen-validate)中间件做了接口参数限制，它可以在 Protobuffer 定义中附加对请求字段限制，在部分更新请求时，如果我们想要某个字段既可以选填，又要在填写的时候有最小长度限制，FieldMask 就无能为力了，比如下面这个例子：

**Protobuffer 定义**

```go
//员工信息，更新用
message StaffInfoForUpdate {
    //员工姓名，2-8个汉字或字母
    string name = 1 [
        (validate.rules).string = {max_len: 8, min_len: 2},
    ];
    //grpc-gateway会记录JSON中传了的字段到FieldMask中
    google.protobuf.FieldMask update_mask = 2;
}
```

“员工姓名”字段是可以选填的，不填代表不更新姓名，但是填的话需要限制最少 2 个字，用 FieldMask 是可以记录到用户是否传了 name，但是 validate 模板生成规则代码限制了用户不可以不传。

**validate 生成的规则代码**

```go
if l := utf8.RuneCountInString(m.GetName()); l < 2 || l > 8 {
    return StaffInfoValidationError{
        field:  "Name",
        reason: "value length must be between 2 and 8 runes, inclusive",
    }
}
```

这阻挡了 FieldMask 的工作，产生了矛盾，除非定制开发 protoc-gen-validate 的代码，但是会引入一些开发和维护的工作。除了与 validate 互斥的问题之外，[Google 的开发文档](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#considerations-for-http-rest)建议 HTTP 接口的部分更新应该采用 PATCH 方法，PUT 则代表全量更新，而我们当前的接口设计全部是 POST，当然我们可以坚持我们自己的规范，比较[REST 规范](https://restfulapi.net/)也没有强制采用 HTTP Method 语义，**REST and HTTP are not same !!**。
同样的例子如果用 wrapper 来实现的话则可以“蒙混过关”，因为 validate 帮忙兼容了一些 [Well-Known Types](https://github.com/envoyproxy/protoc-gen-validate#well-known-types-wkts)，不过这里并没有对枚举的支持，我们可以直接这么定义：

**Protobuffer 定义**

```go
//员工信息，更新用
message StaffInfoForUpdate {
    //2-8个汉字或字母
    google.protobuf.StringValue name = 1 [
        (validate.rules).string = {max_len: 8, min_len: 2},
    ];
}
```

**validate 生成的规则代码**

```go
if wrapper := m.GetName(); wrapper != nil {
    if l := utf8.RuneCountInString(wrapper.GetValue()); l < 2 || l > 8 {
        return StaffInfoForUpdateValidationError{
            field:  "Name",
            reason: "value length must be between 2 and 8 runes, inclusive",
        }
    }
}
```

可以看到 name 可以不传，但是如果传了就要符合参数限制。那么用 wrapper 的方案就只剩下一个问题，就是自定义枚举无法支持，现阶段性价比最高的方法就是在接口中不用自定义枚举，也可以用”字符串或整型加上 validate 限制“代替枚举，所谓性价比就是代码改动小，对现有架构影响小，达到可以接受的效果，好在这是目前了解到的唯一的牺牲。

## 等等党永不为奴

上面两种方案各有优缺点，都可以通过一些开发工作来完善，只是总觉得有点别扭，现在看到 sum types 的提案后觉得还是再等等吧。
