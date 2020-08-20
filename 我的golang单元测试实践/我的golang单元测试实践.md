# 我的 golang 单元测试实践

本文不涉及 go 的单元测试的基本结构，以及为什么我们要写单元测试，和 TDD 的好处。简单是说就是 what why how 这里只提 how，而且是 how I did。

## 测试框架选择

**无框架**

或叫原生框架

- [table driven tests](https://github.com/golang/go/wiki/TableDrivenTests)或叫[控制反转](https://zh.wikipedia.org/wiki/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC)
- [stretchr/testify](https://github.com/stretchr/testify) 好用的 assert 工具

**BDD 框架**

- [ginkgo](https://onsi.github.io/ginkgo/)
- [goconvey](http://goconvey.co/)

我个人喜欢无框架，我认为测试框架的核心价值在于 BDD 和可视化，BDD 让测试代码与业务逻辑紧密结合，非常适合敏捷开发。可视化让测试结果可以更直观，但本质上跟`go test`在控制台输出的内容和`go tool cover -html=coverage.out`输出的页面上的内容区别不大。

## 测试代码风格

控制反转, 是 go 官方推荐的测试风格，也是[gotests](https://github.com/cweill/gotests)工具默认模板的风格，由于我的开发工具 VSCode 中的 go 插件带的默认测试工具就是 gotests，所以下面以 gotests 为例，比如我有一个 MyService 类型，提供一个 Query 方法

```go
// service.go
type MyService struct {
    rdsCli  *redis.Client
    db *gorm.DB
}

func (x * MyService) Query() (int, error) {
    return 0, nil
}
```

用 gotests 工具生成测试代码

```sh
gotests -w -all service.go
```

生成测试代码如下：

```go
// service_test.go
func TestMyService_Query(t *testing.T) {
    type fields struct {
        rdsCli *redis.Client
        db     *gorm.DB
    }
    tests := []struct {
        name    string
        fields  fields
        want    int
        wantErr bool
    }{
        // TODO: Add test cases.
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            x := &MyService{
                rdsCli: tt.fields.rdsCli,
                db:     tt.fields.db,
            }
            got, err := x.Query()
            if (err != nil) != tt.wantErr {
                t.Errorf("MyService.Query() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("MyService.Query() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

得益于控制反转，我只需要在 `TODO` 处添加一些`sub test cases`即可。

但是这个模板也有一些问题

**构造冗余**

比如每写一个`sub test case`，都需要填写构造 Service 的参数，这对于有状态的结构做了很好的隔离，每次测试都使用全新的 MyService。而通常我的 Service 是无状态的，大多数`sub test case`都可以使用同一个 Service 实例，这时每次都构造一遍就会产生冗余。假如我的 Service 会有一个构造方法 New：

```go
func New(rdsCli *redis.Client, db *gorm.DB) *MyService {
	return & MyService{
		rdsCli: rdsCli,
		db: db,
	}
}
```

gotests 工具提供了自定义模板的功能，我可以把模板可以简化为：

```go
// service_test.go
func TestMyService_Query(t *testing.T) {
    x := New(rdsCli, db)
    tests := []struct {
        name    string
        want    int
        wantErr bool
    }{
        // TODO: Add test cases.
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := x.Query()
            assert.Nil(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

这里同时用 assert 包代替 go 原生**丑陋**的错误处理。

**复杂的返回值校验**

另外一个问题，对于待测试的方法的返回值，如果只检查其是否`DeepEqual`，有时候满足不了一些测试需求，当我无法预测返回值的所有字段的时候，比如有一个 Insert 方法，返回值中会带有我无法预测的 LastInsertId，或者一个依赖的其他包的接口，返回值包含一些不稳定字段。这时候有两个解决问题的方向

1. 把所有不稳定的因素改为稳定因素——用 Mock
2. 改变测试模板

第一种方式需要我们对 golang 的代码风格有一个较好的理解，才可以在做 Mock 时可以比较方便。

第一种的具体内容下面再说，我先说一下第二种，是比较简单直接的方式，可以把模板的校验返回值的部分也改为控制反转风格，比如：

```go
// service_test.go
func TestMyService_Query(t *testing.T) {
    x := New(rdsCli, db)
    tests := []struct {
        name    string
        check   func(int, error)
    }{
        // TODO: Add test cases.
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.check(x.Query())
        })
    }
}
```

每个`sub test case`的返回值由用户定义如何检测，发挥控制反转的最后余力。

## 外部依赖怎么 Mock

go 推崇 Interface，舍弃了继承，这突出了**正交(orthogonal)**的概念，隔壁 Rust 也是这样的理念。Interface 是写出 testable 代码的关键，比如依赖注入用 Interface 代替 struct。我举个例子，比如 MyService 依赖了 rand 包暴露的一个 RandSource 结构体和它的 Rand 方法，这个方法返回一个随机整数。

```go
package rand

import "math/rand"

type RandSource struct{}

func New() *RandSource {
	return &RandSource{}
}

func (x *RandSource) Rand() int {
	return rand.Int()
}
```

Service 结构体依赖 RandSource 结构体

```go
// service.go
type MyService struct {
    rs  *rand.RandSource
}

func (x * MyService) Query() (int, error) {
    return x.rs.Rand(), nil
}
```

这时在上面的测试代码中，我们没办法预测 Query 的返回值，这时我想到了写一个 MockRandSource 来代替 RandSource，返回一个稳定的整数。但由于 MyService 依赖的是 RandSource 结构体，go 语言又没有提供继承或者方法重载的语义，没法利用里氏替换原则，这时候 Interface 就派上用场了，我们把依赖由结构体改为 Interface

```go
//定义内部Interface
type randSource interface {
    Rand() int
}

// service.go
type MyService struct {
    rs  randSource //依赖内部Interface
}

func (x * MyService) Query() (int, error) {
    return x.rs.Rand(), nil
}
```

这样我们就可以在测试代码里写一个 randSourceMock 了：

```go
// service_test.go
type randSourceMock struct{}
func (randSourceMock) Rand() int {
    return 123
}
func TestMyService_Query(t *testing.T) {
    x := &randSource{
        rs: &randSourceMock{},
    }
    tests := []struct {
        name    string
        want    int
        wantErr bool
    }{
        {
            name:   "ok",
            want:   123,
        },
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := x.Query()
            assert.Nil(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

这样，所有的外部依赖都可以用 Interface 来写出 testable 的代码了。对于这块的理解上其实有一个小技巧，就是**尽可能的在你的包里用你自己定义的 Interface 做依赖注入**

也可以用[testify's mock](https://github.com/stretchr/testify#mock-package)提供的 Mock 机制做更灵活的 Mock

## 数据库相关的 CRUD 怎么测

**Using External**

CRUD 统统在一个测试方法内完成，C 为 RU 的基础，用 D 做资源回收。不过这就超出了单元测试的概念，偏向于集成测试，即你的程序与外部的数据库的集成。

也可以基于外部的临时容器，初始化一些必要测试数据

**Mock SQL**

CRUD 单独测试

[go-sqlmock](https://github.com/DATA-DOG/go-sqlmock)，完整稳定的实现了[database/sql/driver](https://godoc.org/database/sql/driver)，做到无数据库测试，符合 TDD 工作流。所有基于 go 标准库[database/sql/driver](https://godoc.org/database/sql/driver)的 orm 框架也都支持，以 GORM 为例，比如我有一个 Dao 包，提供一个 LastInertId 方法：

**Mock Redis Server**

[miniredis](https://github.com/alicebob/miniredis) 是一个实现了 Redis Server 的包，专门用于 Go 的单元测试，目前支持 Redis6 的几乎所有开发会用到的命令

Redis

```go
package dao

type Dao struct {
    db   *gorm.DB
}

func New(db *gorm.DB) *Dao {
    return &Dao{
        db:   db,
    }
}

type intT struct{
    Uint1 uint64 `gorm:"column:uint1"`
}

//LastInsertId 获取最后一条插入语句生成的自增ID
func (x *Dao) LastInsertId() (uint64, error) {
    result := &intT{}
    if err := x.db.Raw("SELECT LAST_INSERT_ID() AS `uint1`").Scan(&result).Error; err != nil {
        return 0, err
    }
    return result.Uint1, nil
}
```

测试代码可以这么写：

```go
//dao_test.go
func getMockDB(t *testing.T) (*gorm.DB, sqlmock.Sqlmock) {
    db, mock, err := sqlmock.New()
    assert.Nil(t, err)

    gdb, err := gorm.Open("mysql", db)
    assert.Nil(t, err)

    return gdb, mock
}

func TestDao_LastInsertId(t *testing.T) {
    db, mock := getMockDB(t)
    defer db.Close()
    x := New(db)

    tests := []struct {
        name    string
        want    uint64
        wantErr bool
    }{
        {
            name:    "ok",
            want:    123,
            wantErr: false,
        },
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mock.ExpectQuery(regexp.QuoteMeta("SELECT LAST_INSERT_ID() AS `uint1`")).
                WillReturnRows(sqlmock.NewRows([]string{"uint1"}).
                    AddRow(123))
            got, err := x.LastInsertId(ctx)
            assert.Nil(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

## 其他

**HTTP**

可以用 go 标准库的 httptest 包

**gRPC**

gRPC 生成的 client stub 都是 Interface，所以可以很方便的写 mock

## 总结

如果遵循上面提到的，控制反转测试风格，依赖注入接口，Mock。那么我们几乎就可以做到 Dave 在 2019 Gopher China 上分享《Testing; how, what, why》中提到的一个“信念”：

> **I hold it as an article of faith that writing tests at the same time as the code is a good thing.**
>
> 一个戛然而止的收尾。
