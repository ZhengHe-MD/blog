---
title: Go Error Handling 方案调研
date: 2020-10-05 16:20:00
tags:
- Go
categories:
- engineering
---

自从 2018 年底用 Go 搭建第一个项目以来，已经过去接近 2 年时间，我发现自己从未系统地思考过 Go 的 error handling 方案。最近在阅读 [1] 时，逐渐发现个人和团队都应该花更多的精力建立更加扎实的工程实践方法论，进一步提升交付项目质量。而本篇博客算是向这个方向迈出的第一步。

## 0. 术语说明

为了避免翻译造成的歧义，文中涉及的没有通用翻译中文的术语都会直接使用原英文单词：

| 英文                      | 中文                                     |
| ------------------------- | ---------------------------------------- |
| error                     | 错误                                     |
| exception                 | 异常                                     |
| error-code-based          | 基于错误码                               |
| exception-based           | 基于异常                                 |
| package                   | 包 (Go 中 module 由多个 package 构成)    |
| error wrapping/unwrapping | 包装错误/解包装                          |
| error inspection          | 错误检查                                 |
| error formatting          | 错误格式化                               |
| error chain               | 错误链表，即通过包装将错误组织成链表结构 |
| error class               | 错误类别、类型                           |

下文中，errors package 指代我们定制化的 error handling 方案。

## 1. 文献综述

不同程序语言的 error handling 方案大致可以分为两种：error-code-based 和 exception-based。Raymond 在博客 [2] [3] 中指出 exception-based 错误处理更不利于写出优质的代码，也更难辨别优质和劣质的代码；Go 在设计时选择了 error-code-based error handling 方案，鼓励开发者显式地在 error 出现的地方直接处理 [4]；并在官博 [5] 中提出了 **errors are values** 的理念，只要实现 `Error` 接口的结构体就可以作为 error，不同的项目就能够按需定制 error handling 实现方案，并提出在一些特殊场景下可以利用非通用的代码重构技巧避免冗长、啰嗦的表达，如errWriter；许多来自 Java、Python 等语言的工程师习惯了 exception-based 的方案，遇到 Go 时感到十分不习惯 [6]，但如果我们总是希望在一门新语言中尝试套用自己熟悉语言的语法，就无法充分理解其它语言在这方面的设计理念。Go 核心工程师 Rob Pike 在 [7] 中描述了他如何在 [Upspin](https://upspin.io/) 项目中定制 error 信息和处理方案，使得项目对程序、用户及开发者更加友好；许多 error handling 项目都关注到了多层嵌套调用场景下的上下文注入问题，即所谓的 error wrapping，其中 Dave Cheney 的项目 pkg/errors [8] 被广泛使用，Go 在 1.13 后也提供类似的原生解决方案 [9]；受 [7] [8] 的启发，Ben Johnson，boltDB 的作者，结合自己多年的编码经验，在 [10] 中提出 **Failure is your Domain** 的观点，认为每个项目应当构建特有的 error handling package，并提出逻辑调用栈 (logical stack) 的概念，在 GopherCon 2019，还有工程师在推广类似的方案 [11]。

error handling 可以细分为 checking、inspection 和 formatting 三部分，分别指判断 error 发生与否、检查 error 类型、打印 error 上下文。在发现 Go 社区的开发者们因为语言本身对 error handling 的支持不足，频繁创造各种各样的轮子之后，Russ Cox 在 2018 年末发布了两个新提议 [12] [13]，前者尝试解决 checking 代码冗长的问题；后者尝试解决 inspection 的信息丢失以及 formatting 的上下文信息不足问题。目前仅 inspection 的方案被整合到了 1.13 中，直到最近的 1.15 版本没有新的解决方案出现。

## 2. 项目综述

发布之初，Go (<1.13) 仅提供 `Error` 接口及 `errors.New`、`fmt.Errorf` 两个构建 error 的方法 [4]；Go 1.13 支持利用 `%w` 格式化符号实现 error wrapping，并提供 `Unwrap`、`errors.Is` 以及 `errors.As` 来解决 error wrapping 过程中上下文缺失的问题 [9]；spacemonkeygo 为了将大型 Python 仓库迁移到 Go 上，开发了 [14] ，模拟 Python 中 error class 的继承，支持自动记录日志、调用栈以及任意键值数据，支持 error inspection；juju errors [15] 因 juju 项目而诞生，在 wrap error 时，你可以选择保留或隐藏 error 产生的原因 (cause)，但它的 `Cause` 方法仅 unwrap 一层，而 [8] 会递归地遍历 error chain，[16] 中的概念与 [15] 类似，仅在 API 上有所不同；hashicorp 开源的 errwrap [16]，支持将 errors 组织成树状结构，并提供 `Walk` 方法遍历这棵树；pkg/errors [8] 提供 wrapping 和调用栈捕获的功能，并利用 `%+v` 格式化 error，展示更多的细节，它认为只有整个 error chain 最末端的 error 最有价值，pingcap/errors [18] 基于 [8] 二次开发，并且在 [19] 中增加了 error 类 (域) 的概念；upspin.io/errors [20] 是定制化 error 的实践范本，同时引入了 `errors.Is` 和 `errors.Match` 用于辅助检查 error 类型；[21] 考虑了 error 在进程间传递的场景，让 error handling 具备网络传播兼容能力。

## 3. 现状

### 3.1 举例

目前，公司内部生产环境使用 Go 1.12，仅有最基本的 error handling 工具，此外服务器研发团队没有统一 error handling 方案。一段 Controller 中典型的代码如下所示：

```go
func (m *GRPCADServiceImpl) DelAD(ctx context.Context, req *ad.DelADReq) (res *ad.DelADRes, err error) {
  // (1)
  fun := "GRPCADServiceImpl.DelAD -->" 
  res = &ad.DelADRes{}
  
  passed, err := !auth.CheckAuth(ctx, req.Uid, auth.AccessCodeAdDelete)
  if err != nil {
    // (2)
    err = fmt.Errorf("%s check auth failed err %v", fun, err)
    // (5)
    xlog.Error(err)
    return
  }
  
  if !passed {
    // (3)
    res.ErrInfo = &grpcutil.ErrInfo{Code: -1, Msg: "not authorized"}
    return
  }

  conds := map[string]interface{}{"id": req.Id}

  adToDelete, err := model.ADDao.GetOneAD(ctx, conds)
  // (4)
  if err == scanner.ErrEmptyRow {
    res.ErrInfo = &grpcutil.ErrInfo{Code: -1, Msg: "ad not found"}
    return
  }
  
  if err != nil {
    return
  }

  _, err = model.ADDao.DeleteAD(ctx, conds)
  if err != nil {
    return
  }

  res.Data = &ad.DelADRes_Data{
    Id: req.Id,
  }

  return
}
```

先抛开代码逻辑的正确性，只谈 error handling 逻辑，我们可以大致总结出如下几点 (以下序号与上述代码中的序号一一对应)：

1. 每个函数的起始处设置一个 `fun := "FunctionName -->"`，便于生成日志和 error 信息
2. 如果遇到服务端 error，则使用 `fmt.Errorf` 将被调函数返回的 error 信息包装一层，并记录 ERROR 级别日志，继续向上返回
3. 如果遇到客户端 error，则设置响应结构体的 ErrInfo 字段，可以设定 error code 和用户可见的 error msg
4. 如果需要针对不同 error 类型执行不同逻辑分支，则利用等价判断。

### 3.2 讨论

以上几点并未形成任何跨项目的规范，仅仅是路径依赖的结果。其中有一些小问题，我在下面列举出来：

##### fun 的约定值

`fun` 的约定值是 `${结构体名称}.${函数名称} -->`。首先，后面的箭头符号 `-->` 需要占用 3 个字符，日志量大时会占用可观的存储空间。除此之外，每位工程师都需要手写 `-->`，即便它可以增加日志可读性，至少我们应当让它自动生成。

##### error wrapping

前面已经提到过，使用 `fmt.Errorf` 包装被调函数返回的 error 会丢失一些上下文信息 (如下层错误类型)，但打印到日志中的 error 信息一般足以追溯线上问题。抛开上下文信息丢失问题，以上面的代码为例：

```go
fmt.Errorf("%s check auth failed err %v", fun, err)
```

的打印结果是："GRPCADServiceImpl.DelAD --> check auth failed err auth.CheckAuth --> xxx"，可以发现 "check auth failed" 与 "auth.CheckAuth" 的内容相同，增加了消息的冗余度。如果保证每个被调函数也会在错误消息中增加 `fun` 的信息，我们可以将其简化为： 

```go
fmt.Errorf("%s %v", fun, err)
```

其打印结果就是："GRPCADServiceImpl.DelAD --> auth.CheckAuth --> xxx"，这里已经可以看到**逻辑调用栈**的影子。但因为 `fun` 的存在主要是为了打日志方便，当不需要打日志时就不会声明这个局部变量，因此没有形成统一的规范，打印结果中的逻辑调用栈信息实际上并无法保证完整性。

##### 客户端 error

遇到客户端 error，如鉴权失败、请求参数校验失败等情况，需要在每个可能出问题的地方书写同样的代码逻辑来构建 `ErrInfo` 字段。目前，error code 的使用刚刚起步，采用不同业务线预留号段的策略；error msg 还处在蛮荒状态，尚未有进入讨论阶段的方案。

##### error inspection

error inspection 采用的是比较原始的等价判断 (==) 或类型断言来实现，因为没有 wrapping 方案，类型断言也基本不用，或至少没有在实践上达成共识。

## 4. error handling 的世界观和方法论

在上文中，我们阐述了 error handling 现状，并讨论其中的潜在问题。需要肯定的是，即便当前的方案并不完美，它事实上已经满足开发者平时的服务日志观测及问题排查的需求。但我们缺失的是更系统的跨项目 error handling 共识及其方案。经过调研和分析后，我将在本节介绍我认同的 Go error handling 的世界观，并介绍相应的实践方法论。

### 4.1 世界观

####4.1.1 "happy path" 与 "sad path" 地位相同

如果我们将函数的正常逻辑路径称为 "happy path"，异常逻辑路径称为 "sad path"。在使用 exception-based error handling 的编程语言时，工程师认为 "sad path" 是一种需要额外考虑的特殊情况，需要特殊对待；而在 Go 开发者眼里，"happy path" 和 "sad path" 都是一般的情况，二者应该同样重要，被同等对待。

#### 4.1.2 面向应用程序、用户及运维

> The tricky part about errors is that they need to be different things to different consumers of them.
>
> — Ben Johnson

当我们在代码中处理 error 时，需要思考这样一个问题："是谁在消费这些 errors？" 在任意一个服务的生命周期中，通常至少有 3 个角色关心 error：应用程序 (application) 本身、服务的用户 (end user)、服务的维护者 (operator)。在刚才的例子中，error 类型检查 (4) 面向的是应用程序；创建 res.ErrInfo (3) 面向的是服务的用户；打印日志 (5) 面向的是维护者。因此一个设计精良的 errors package 要能够让工程师自如地处理 error 与各个角色之间的信息传递。

##### 应用程序与 error

应用程序可能拥有各种各样的外部依赖，比如第三方服务、内部 RPC 服务、数据库服务、消息队列服务，甚至磁盘、网卡、CPU 等等。这些依赖本身随时可能出现这样或那样的问题，但这类问题本身通常不会导致应用程序的进程崩溃，只要问题是临时的、非致命的、在定义范围内的，应用程序就可以从容地根据 error 的特点处理。因此应用程序需要能够准确、方便、健壮地获取 error 特征。

##### 用户与 error

当服务运行遇到 error 时，需要向普通的 C 端用户提供友好、明确的消息提示，让他明白系统正处于异常状态，可以稍后重试或联系客服、技术人员。因此消息应该是对人类友好的自然语言。除此之外，系统内部的细节，如错误栈信息，不应当直接暴露给 C 端用户，对于未明确定义的 error 更应如此。主要原因在于：

1. 用户不应该关心服务的实现细节
2. 暴露不必要的细节可能会降低系统安全性

##### 维护者与 error

遇到线上问题时，服务的维护者接到报警后，需要根据详细的 error 信息做根源分析，这时信息越多越好，当然更高的可读性能够帮助维护人员更快地定位问题，解决问题。这里的维护者一般是服务的开发者，而非 devop 团队成员。

*备注：本节的观点主要源自于 [7] [10]*

### 5. 方法论

在进入方法论之前，需要先明确适用范围：本节提出的方法针对的是单个网络服务 (如 HTTP/RPC) 内部的 error，不涉及 error 在进程间的传递的部分，针对后者可以考虑 cockroachdb 团队提出的解决方案 [21]。

#### 5.1 errors are values

任何实现了 `Error` 接口的数据类型都是 error，它们与字符串、整数、结构体相比并没有特别之处。

##### 5.1.1 将 "happy path" 留在控制流的最外层

Go 鼓励工程师将逻辑的 "happy path" 留在函数缩进的最外层，而把 "sad path" 放到第二级缩进中，如：

```go
func Do() (ret interface{}, err error) {
  // happy path
  v1, err := A()
  if err != nil {
    // sad path 1
  }
  
  v2, err := B(v1)
  if err != nil {
    // sad path 2
  }
  
  ret = process(v1, v2)
  return
}
```

「现状」一节中的例子也遵守了这个规则。

##### 5.1.2 error 是否为空反映调用成功与否

> Never use nil to indicate failure
>
> — Dave Cheney

所有可能产生 error 的函数都使用 Go 的多值返回特性，其中最后一个返回值默认为 `error` 类型，即：

```go
func A (args ...interface{}) (ret interface{}, err error)
```

调用方默认先检查 err 是否为空，为空则认为调用成功，非空则认为调用失败，如：

```go
func main() {
  ret, err := A(1, 2, 3)
  if err != nil {
    // 调用失败，即 sad path
  }
  // 调用成功，即 happy path
}
```

一旦调用失败，调用方不应该使用其它任意返回值，或对其它返回值有任何假设，而是采用 fail-fast 的策略结束执行。**不要使用其它返回值的特征作为调用成功与否的依据**，这样做既不符合 Go 的设计理念，也会使代码的可读性大大下降。

#### 5.2 从 error 的消费者角度出发

本节我们要定制自己的 errors package，并利用它来解决现状中的问题，同时践行我们的世界观。既然关心 error 的角色有很多，我们就索性分开管理面向不同消费者的信息，因此定义 `Error` 结构体如下：

```go
// Error defines a standard application error.
type Error struct {
  // For application/machine
  Class Class
  // For both users & operators, see methods ErrMsg (users) and Error (operators)
  Msg string
  // For operators
  Op    Op    // logical operation
  Code  int   // error code, which identifies an user-defined error
  Cause error // error from lower level
}

type Op string
type Class string
```

其中 Class 表示 error 类型，面向应用程序；Msg 表示 error 消息，既面向外部用户，也面向维护者，分别体现在 `ErrMsg` 和 `Error` 两个方法上；Op 指 Error 生成时所处的函数，Code 是 error 的标识，Cause 存储下层 error，三者面向维护者。为了方便创建 Error，errors package 还提供一个 constructor：

```go
func E(args ...interface{}) error {
  // ignore preprocessing
  e := &Error{}
  for _, arg := range args {
    switch arg := arg.(type) {
    case Class:
      e.Class = arg
    case string:
      e.Msg = arg
    case Op:
      e.Op = arg
    case int:
      e.Code = arg
    case *Error:
      cp := *arg
      e.Cause = &cp
    case error:
      e.Cause = arg
    default:
      _, file, line, _ := runtime.Caller(1)
      log.Printf("errors.E: bad call from %s:%d: %v", file, line, args)
      return fmt.Errorf("unknown type %T, value %v in error call", arg, arg)
    }
  }
  // ignore postprocessing
  return e
}
```

这样开发者就可以很方便地构建 Error，并填充任意需要的字段，包括 wrap 下层 error。

##### 5.2.1 应用程序

应用程序主要关注 error 的两个方面：「是否为空」及「所属类型」。前者我们在上文中已经讨论，本节只关注后者。有的开发者可能会定义许多 error 类型，涉及各种分支情形，以便应用程序可以根据各种场景做判断，如：

```go
const (
  ErrCodeInvalidUserName    Class = "invalid username"
  ErrCodeInvalidPassword    Class = "invalid password"
  ErrCodeUserNotFound       Class = "user not found"
  ErrCodeDepartmentNotFound Class = "department not found"
  ErrCodeTokenNotFound      Class = "token not found" 
  //...
)
```

但如果从应用程序的角度出发，一般情况下，上层代码逻辑仅关心少数的几种 error 类型，甚至大多数情况下仅关心 error 是临时性还是永久性的即可。error 类型过多，则上层调用方需要处理的场景就越多，这既给调用方添加负担，又将实现细节暴露到上层。经过权衡，我认同 Ben Johnson 的主张，在 errors package 中定义少量几个含义宽泛的 error 类型即可，就先从以下几个开始：

```go
const (
	Conflict   Class = "conflict"          // Action cannot be performed
	Internal   Class = "internal"          // Internal error, error from DB, RPC, and other external services
	Invalid    Class = "invalid"           // Validation failed
	NotFound   Class = "not_found"         // Entity does not exist
	PermDenied Class = "permission_denied" // Does not have permission
	Other      Class = "other"             // Unclassified error
)
```

也许你已经察觉到这与 HTTP 协议的响应码有些相似。最后的 Other 类型可以容纳未定义的 error，在必要时可以将其它类型从中抽出。判断 error 类型时，可以使用工具函数 Is，如：

```go
if errors.Is(err, errors.Invalid) {
  // sad path
}
```

不同抽象层中的 error 可以通过多层 wrapping，形成 error chain。Is 会从头到尾遍历 error chain，以第一个 Class 取值不为 zero value 的 `*Error` 为判定依据。实践中，我们约定：**error chain 上最后一个数据类型为 `*Error` 的节点，它必须有一个定义好的的 Class，而它前面的所有节点 Class 都为空**。背后的理由是：整个 error chain 应该只有一个类型，且开发者关心的是根因。

##### 5.2.2 用户

直接返回给普通用户的消息存放在 Msg 字段中，它的内容通常在最外层 (Controller) 设置。errors package 也提供了 ErrMsg 工具函数：

```go
func ErrMsg(err error) string {
  code := firstCode(err)
  // if err != nil, msg will be set to a default msg
  msg := firstMsg(err)

  if msg != "" && code != 0 {
    return fmt.Sprintf("[%d] %s", code, msg)
  }

  return msg
}
```

ErrMsg 与 Is 类似，其中 code 取值为 error chain 上第一个非 0 的 error code；msg 取值为 error chain 上第一个非空字符串的 error msg，如果参数 err 不为空，且找不到合法的 msg，则返回默认消息。

##### 5.2.3 维护者

服务出现线上问题时，对服务的维护者而言最重要的就是日志信息。有效的日志信息需要包括至少两部分：(逻辑) 调用栈和 error 详情，前者帮助开发者追溯引起 error 的调用链；后者为开发者提供造成 error 的现场信息，如位置和参数。日志信息的展示对应 error 的 formatting 功能。

在 Go 程序中，发生 panic 时就可以看到类似如下的调用栈信息：

```go
goroutine 11 [running]:
testing.tRunner.func1(0xc420092690)
    /usr/local/go/src/testing/testing.go:711 +0x2d2
panic(0x53f820, 0x594da0)
    /usr/local/go/src/runtime/panic.go:491 +0x283
github.com/yourbasic/bit.(*Set).Max(0xc42000a940, 0x0)
    ../src/github.com/bit/set_math_bits.go:137 +0x89
github.com/yourbasic/bit.TestMax(0xc420092690)
    ../src/github.com/bit/set_test.go:165 +0x337
testing.tRunner(0xc420092690, 0x57f5e8)
    /usr/local/go/src/testing/testing.go:746 +0xd0
created by testing.(*T).Run
    /usr/local/go/src/testing/testing.go:789 +0x2de
```

这里的调用栈信息非常完整，完全满足开发者的问题排查需求。但在日志采集的过程中，由于出现了换行，会造成检索难的问题；如果将其合并成一行，则会严重影响可读性。errors package 使用逻辑调用栈，可以打印出轻量的、可读性强的 error 信息，以下便是对应 `Error` 接口的实现：

```go
func (e *Error) Error() string {
  b := bytes.NewBuffer(nil)
  if e.Op != "" {
    _, _ = fmt.Fprintf(b, "%s: ", e.Op)
  }
  // print operation info of the tail error
  if e.Cause == nil {
    e.writeOpInfo(b)
    return b.String()
  }

  // if the inner error is of type *Error, only print the Op,
  // otherwise, print operation info and the inner error
  if _, isError := e.Cause.(*Error); isError {
    b.WriteString(e.Cause.Error())
  } else {
    e.writeOpInfo(b)
    b.WriteString(e.Cause.Error())
  }

  return b.String()
}

func (e *Error) writeOpInfo(b *bytes.Buffer) {
  if e.Code != 0 && len(e.Msg) > 0 {
     _, _ = fmt.Fprintf(b, "[%d] %s", e.Code, e.Msg)
  } else if e.Code != 0 {
    _, _ = fmt.Fprintf(b, "[%d]", e.Code)
  } else if len(e.Msg) > 0 {
    _, _ = fmt.Fprintf(b, "%s", e.Msg)
  }
}
```

假设存在以下场景：

```go
func GetUser(ctx context.Context, id int) (user *User, err error) {
  op := errors.Op("GetUser")
  if user, err = db.GetUser(ctx, id); err != nil {
    return errors.E(op, errors.Internal, 10001, fmt.Sprintf("get user %d from db", id), err)  
  } else {
    // happy path (ignored)
  }
}

func HandleGetUser(ctx context.Context, req GetUserReq) (res GetUserRes) {
  op := errors.Op("HandleGetUser")
  if user, err := GetUser(ctx, req.Id); err != nil {
    err = errors.E(op, err)
    log.Error(err) // (1)
    res = GetUserRes{Msg: errors.ErrMsg(err), Code: errors.ErrCode(err)}
    return
  }
  // happy path (ignored)
}
```

在 (1) 处打印出来的日志就是 `HandleGetUser: GetUser: [10001] get user 10873521 from db`，前半段是服务内部的逻辑调用栈，后半段是出错细节，合并在同一行中，对日志收集和查看也比较友好。在打错误日志时，需要遵守一个原则：**每个 error 只被打印一次**，且通常这一次打印发生在调用链的头部，因为头部拥有最完备的信息。这种做法的不方便之处在于，调用链上的每个函数都需要定义一个局部变量 `op`，并且当下层返回的 error 不为空时需要包装一层，增加了许多人工成本，但鉴于在「现状」一节中的方案也做了这样的事情，引入 errors package 实际并未增加额外的工作。

如果我们构建的是开放 API 服务，为了方便快速定位问题，可以使用 `Code` 标识具体的某个 error，其数值范围，每个取值的含义由使用方自行决定。

*备注：本节的观点主要源自于 [1] [7] [10]*

### 6. 完整示例

现在我们用上节介绍的世界观和方法论，尝试改善「现状」：

```go
func (m *GRPCADServiceImpl) delAD(ctx context.Context, req *ad.DelADReq) (res *ad.DelADRes, err error) {
  op := errors.Op("GRPCADServiceImpl.DelAD")
  res = &ad.DelADRes{}
  
  passed, err := !auth.CheckAuth(ctx, req.Uid, auth.AccessCodeAdDelete)
  if err != nil {
    return errors.E(op, err)
  }
  
  if !passed {
    res.ErrInfo = &grpcutil.ErrInfo{Code: ErrCodeNotAuthorized, Msg: ErrMsgNotAuthorized}
    return
  }

  conds := map[string]interface{}{"id": req.Id}

  adToDelete, err := model.ADDao.GetOneAD(ctx, conds)
  if errors.Is(err, errors.NotFound) {
    res.ErrInfo = &grpcutil.ErrInfo{Code: errors.ErrCode(err), Msg: fmt.Sprintf("未找到广告 %d", req.Id)}
    err = nil
    return
  }
  
  if err != nil {
    return errors.E(op, err)
  }

  _, err = model.ADDao.DeleteAD(ctx, conds)
  if err != nil {
    return errors.E(op, err)
  }

  res.Data = &ad.DelADRes_Data{Id: req.Id}
  return
}

func (m *GRPCADServiceImpl) DelAD(ctx context.Context, req *ad.DelADReq) (res *ad.DelADRes, err error) {
  res, err := delAD(ctx, req)
  if err != nil {
    xlog.Error(err)
    res.ErrInfo = &grpcutil.ErrInfo{Code: errors.ErrCode(err), Msg: errors.ErrMsg(err)}
  }
  return
}
```

为了代码的简洁，打 ERROR 级别日志和根据 err 构建 ErrInfo 的逻辑单独抽离到 `DelAD` 函数中，当然我们也可以通过 Interceptor 来达到同样的目的。

## 参考

[1]: [practical-go: gophercon-singapore-2019#error_handling](https://dave.cheney.net/practical-go/presentations/gophercon-singapore-2019.html#_error_handling)

[2]: [Cleaner, more elegant, and wrong](https://devblogs.microsoft.com/oldnewthing/20040422-00/?p=39683)

[3]: [Cleaner, more elegant, and harder to recognize](https://devblogs.microsoft.com/oldnewthing/20050114-00/?p=36693)

[4]: [The Go Blog: Error handling and Go](https://blog.golang.org/error-handling-and-go)

[5]: [The Go Blog: Errors are values](https://blog.golang.org/errors-are-values)

[6]: [Exploring Error Handling Patterns in Go](https://8thlight.com/blog/kyle-krull/2018/08/13/exploring-error-handling-patterns-in-go.html)

[7]: [Error handling in Upspin](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html)

[8]: [pkg errors](https://github.com/pkg/errors)

[9]: [Working with Errors in Go 1.13](https://blog.golang.org/go1.13-errors)

[10]: [Failure is your Domain --- Ben Johnson](https://middlemost.com/failure-is-your-domain/)

[11]: GopherCon 2019: Marwan Sulaiman - Handling Go Errors, [video](https://www.youtube.com/watch?v=4WIhhzTTd0Y), [summary](https://about.sourcegraph.com/go/gophercon-2019-handling-go-errors/)

[12]: [Error Handling — Problem Overview](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md)

[13]: [Error Values — Problem Overview](https://go.googlesource.com/proposal/+/master/design/go2draft-error-values-overview.md)

[14]: [spacemonkeygo errors](https://github.com/spacemonkeygo/errors)

[15]: [juju errors](https://github.com/juju/errors)

[16]: gopkg errgo [v1](https://github.com/go-errgo/errgo/tree/v1.0.1), [v2](https://github.com/go-errgo/errgo/tree/v2)

[17]: [hashicorp errwrap](github.com/hashicorp/errwrap)

[18]: [pingcap errors](https://github.com/pingcap/errors)

[19]: [pingcap parser terror](https://github.com/pingcap/parser/blob/release-3.0/terror/terror.go)

[20]: [upspin.io errors](https://github.com/upspin/upspin/tree/master/errors)

[21]: [cockroachdb errors](https://github.com/cockroachdb/errors)

