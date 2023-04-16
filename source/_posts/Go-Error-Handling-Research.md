---
title: 如何在 Golang 项目中处理好错误
date: 2020-10-05 16:20:00
category: 
- 编程
---

造一辆能跑在路上的车并非难事，但要这辆车能在各种路况、气候和突发事件下安全行驶，事情就不再简单。如果把写程序比喻成造车，构建程序的主要功能就是让车跑起来，而处理好错误就是让车安全地跑。**错误是程序的重要组成部分，能否在程序中处理好错误决定了软件的质量上限**。在这篇博客中，我将介绍个人在 Golang 项目中错误处理的思考。

<!--more-->

# 谁在消费错误

> The tricky part about errors is that they need to be different things to different consumers of them。 --- Ben Johnson

要妥善地处理好程序中的错误，首先应想清楚这些错误的消费者是谁。boltdb 的作者 Ben Johnson 在[这篇博客](https://middlemost.com/failure-is-your-domain/)里总结了他的思考：**有三种角色在消费错误，它们分别是用户 (end user)、程序 (application) 和运维 (operator)**。

## 消费者 1：用户

当服务遇到错误，无法完成用户的请求时，我们需要告诉用户「是什么」和「怎么做」，比如：

* 您的权限不足，请联系 xxx 开启
* 系统临时故障，请稍后重试

这里的「是什么」并非越具体越好，一般告诉用户错误的大类即可：是参数错误、还是权限问题、亦或是服务端超载。许多工程师会本能地把原本应该打印在日志里的信息告诉用户，这么做背后隐藏着有两个目的：

* 如果用户有技术背景知识足够，能理解根因
* 用户反馈时会把错误信息带上，能加速定位

但仔细一想，这些目的经不起推敲。首先，用户根本不关心背后的细节和实现，即便这些用户是软件工程师这个断言也没问题；其次，暴露过多的信息可能给恶意攻击者留下线索，降低系统的安全性；最后，问题定位是低频场景，通过日志查询详细的信息即便速度慢一些，但并非无法接受。

## 消费者 2：程序

许多时候，程序需要根据错误的类型来精细地控制逻辑。比如，当 X 服务发送请求给 Y 服务，Y 服务无法满足该请求，便返回错误。此时，X 服务是否应该重试？这取决于返回的错误是临时性的还是不可恢复的；当 X 服务的 DAL (Data Access Layer) 发送请求到数据库，后者返回错误时，X 服务应该给用户返回什么信息？这取决于数据库返回的错误是什么类型，是数据找不到？还是数据库表满了？还是别的原因？

这错误的类型定义方面，业界已经有许多成型的实践：

* [HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)：400、401、403、404、429、500、502、503...
* [gRPC](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)：INVALID_ARGUMENT, DEADLINE_EXCEEDED, NOT_FOUND, ALREADY_EXISTS, ...
* [MySQL](https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html)：1040、1045、1046、1064、1114...

它们都经过了无数项目的考验，非必要不重复造轮子。

## 消费者 3：运维

用户遇到无法解决的问题时，最终会来到运维的手上。服务日志是运维定位问题的利器。将错误及错误发生的背景信息打印到日志里，将极大地方便故障排查。要想定位快，细节就要越丰富，这些细节可能包括错误发生时的：

* 一句话描述
* 函数调用栈
* 请求上下文 (request_id、user_id、device_id)

其中「一句话描述」与「函数调用栈」就可能来自于错误。

# "Errors are values"

> Values can be programmed, and since errors are values, errors can be programmed...The key lesson, however, is that errors are values and the full power of the Go programming language is available for processing them. --- Rob Pike

2015 年 1 月，[Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike) 在 The Go Blog 上发表了题为 "Errors are values" 的[文章](https://go.dev/blog/errors-are-values)，并在当年的 Gopherfest [演讲](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=973s) "Go Proverbs" 中将这句话列在 19 个 proverbs 之中。它是每位 Golang 工程师应该铭记的一句话。

## 错误只是一个普通值

### The error interface

在 Golang 中，任意实现了 error interface 的数据类型都被认为是错误：

```go
type error interface {
	Error() string
}
```

它甚至可以只是一个字符串：

```go
type err string

func (e err) Error() string {
	return e
}
```

> 💁‍♂️ Golang 中没有 implement 关键词，只要实现了 interface，就等价于 implement。

### 作为返回值

既然错误只是一个普通值，这个值就可以被作为函数的入参和出参。如果一个函数的执行过程中可能出现错误，那么 error 约定俗成地会作为最后一个返回值，举例如下：

```go
// example 1:
res, err := http.Get("http://localhost:8080")
// example 2:
if u, err := url.Parse("invalid-url"); err != nil {
	// handle sad path
} else {
	// handle happy path
}
```

## 与使用 Exception 的区别

在许多当下流行的编程语言中，基于 Exception 的错误处理占主流地位，比如 C++、Java 和 Python。对于从这些语言转到 Golang 的工程师而言，"errors are values" 的观点相当激进，难以适应。Stackoverflow 的前 CEO Joel Spolsky 在 2003 年发表过一篇[博客](https://www.joelonsoftware.com/2003/10/13/13/)，在其中他讨论了用 Exceptions 处理错误带来的问题：

> I consider exceptions to be no better than "goto's", [considered harmful](http://www.acm.org/classics/oct95/) since the 1960s, in that they create an abrupt jump from one point of code to another. --- Joel Spolsky

Joel 认为更好的方式是将错误当作普通的返回值，而程序应该在拿到返回值时立即处理它，尽管这会让程序变得更啰嗦，但啰嗦总比牺牲软件的质量好一些。

> 💁‍♂️ 本节并非想说明语言设计的优劣，只是想介绍一下 Golang 的错误处理设计理念的由来。

## 若干 error interface 的实现

既然 "errors are values"，我们就可以利用 Golang 赋予的所有逻辑表达能力处理错误，为不同项目、场景定制化设计。无论是标准库还是社区中都有许多相关实践，这里分别举几个例子：

### 标准库

**1. errorString**

```go
// src/errors/errors.go
func New(text string) error {
	return &errorString{text}
}

type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

利用 `errors.New` 创建的错误实际上就是这里的 `errorString`。

**2. joinError**

```go
// src/errors/join.go
type joinError struct {
	errs []error
}

func (e *joinError) Error() string {
	var b []byte
	for i, err := range e.errs {
		if i > 0 {
			b = append(b, '\n')
		}
		b = append(b, err.Error()...)
	}
	return string(b)
}
```

一些场景里，我们希望合并多个错误，同时保留这些错误的原始信息，这时可以用 `errors.Join`，后者就会创建一个 `joinError`。

**3. os.PathError**

```go
// src/io/fs/fs.go
// PathError records an error and the operation and file path that caused it.
type PathError struct {
	Op   string
	Path string
	Err  error
}

func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
func (e *PathError) Unwrap() error { return e.Err }
```

在执行文件操作遇到错误时，除了记录错误本身，保留操作类型、文件路径信息能帮助我们更快地定位问题，这里的 `PathError` 就干了这么一件事。

### 社区

许多团队为了方便在自己的项目中处理错误，定制化开发了许多 Golang packages，然后开源出来造福社区。以下列举一些项目供读者进一步了解，这里不再赘述：

* [GitHub - uber-go/multierr](https://github.com/uber-go/multierr)
* [GitHub - juju/errors](https://github.com/juju/errors)
* [GitHub - go-errors/errors](https://github.com/go-errors/errors)
* [GitHub - cockroachdb/errors](https://github.com/cockroachdb/errors)
* [GitHub - pkg/errors](https://github.com/pkg/errors)
* [GitHub - pingcap/errors](https://github.com/pingcap/errors)

# 标准库的演进

Golang 对于语法和功能的添加十分克制，因此 errors 标准库迭代之路可谓是小心翼翼。

## <1.13: 点

在 Go1.13 之前，每个错误都是一个「点」，错误之间无法建立联系。我们可以用 `errors.New` 和 `fmt.Errorf` 这两种方法创建一个新的错误：

```go
// create an error
var RecordNotFoundErr = errors.New("DB: record not found")
var UserNotFoundErr = fmt.Errorf("user not found: %v", RecordNotFoundErr)
```

如果要在程序中消费它，可以通过检查值或类型是否相等来控制程序逻辑：

```go
// check identity
if err == RecordNotFoundErr {
	// case 1
} else {
	// case 2
}
// check type
if nerr, ok := err.(net.Error) {
	// case 1
} else {
	// case 2
}
```

这时有一个常见的问题：当我们想给错误补充一些信息时，错误之间的血缘关系会消失，比如：

```go
var RecordNotFoundErr := errors.New("DB: record not found")
var UserNotFoundErr = fmt.Errorf("user not found: %v", RecordNotFoundErr)
```

程序拿到 `UserNotFoundErr` 时，它已经和 `RecordNotFoundErr` 没有任何关系，我们无法针对它做任何的值或类型的判断。

## 1.13-1.19: 链表

Go1.13 支持了错误的包装 (wrap)，于是错误之间可以形成「链表」。Golang 官方为此发布了一篇[博客](https://go.dev/blog/go1.13-errors)，介绍相关的最佳实践。具体地说，`fmt.Errorf` 新增了一个格式标记「%w」，开发者可以用它包装错误：

```go
var RecordNotFoundErr = errors.New("DB: record not found")
var UserNotFoundErr = fmt.Errorf("user not found: %w", RecordNotFoundErr)
```

与「%v」不同，「%w」会在创建新错误的同时，保留对下层错误的引用。这时开发者可以通过 errors package 新增的两个方法来检查错误值或错误类型：

```go
// check error identity: errors.Is
if errors.Is(err, RecordNotFoundErr) {}
// check error type: errors.As
var nerr *net.Error
if errors.As(err, &nerr) {}
```

`errors.Is` 和 `errors.As` 都会递归地遍历整条错误链表，确认链表上是否存在相等的值或类型。除此以外，为了将这种递归的能力开放，Go1.13 还提供了 `errors.Unwrap` 方法，方便开发者获取链表上下一个错误节点：

```go
var recordNotFoundErr = errors.Unwrap(UserNotFoundErr)
```

## 1.20: 树

Go1.20 在 Go1.13 的基础上更进一步，支持一次包装多个错误，于是错误之间可以建立「树」状关系。在使用层面的体现就是 `fmt.Errorf` 方法支持指定多个「%w」标记，即同时包装多个错误：

```go
var RecordNotFoundErr = errors.New("DB: record not found")
var NotFoundErr = errors.New("NotFound")
var UserNotFoundErr = fmt.Errorf("user not found: %w (%w)", RecordNotFoundErr, NotFoundErr)
```

相应地， `errors.Is` 与 `errors.As` 也从对链表遍历升级成了对树的遍历。

# 业务服务中的错误处理实战

> 📢 本小节为个人开发经验总结，存在一些观点倾向，请按需摄取。

那么我们应该如何利用上述的思路和工具，在业务服务开发中合理地处理错误？我将解决方案概括成了四句话：

* 定义通用错误
* 底层转换标识
* 中间填充信息
* 上层统一判断

下面就来分别解释它们的含义。

## 定义通用错误

如果你足够幸运能在标准化做得很强的公司工作，那么公司内部应该会有一套稳定通用错误标准定义，比如 [Google Cloud](https://cloud.google.com/apis/design/errors)，直接使用这些标准错误来驱动服务内部的错误处理即可，你可以直接跳过此步骤；如果你的公司与我工作过的大多数公司一样，缺乏人人遵守的工程化标准，就需要定义服务内部或团队内部的通用错误。

定义通用错误并不难，一般根据需要选择 HTTP 或 gRPC 的错误定义即可，比如：

```go
// pkg/errors.go
var (
	NotFound   = errors.New("NotFound")
	BadRequest = errors.New("BadRequest")
	Internal   = errors.New("InternalServerError")
	//...
)
```

这里的通用错误主要是提供给程序和运维消费，并非面向用户，粒度不必定义地特别细致。

## 底层转换标识

在跨进程调用处，无论是访问数据库、消息队列、配置中心，还是请求上游的微服务，一旦发生错误就立即包装成定义好的通用错误：

```go
// DAO
func (mud *MySQLUserDAO) GetUser(ctx context.Context, id int64) (*User, error) {
	user, err := mud.getUser(ctx, id)
	return user, mud.wrapMySQLError(err)
}

func (mud *MySQLUserDAO) getUser(ctx context.Context, id int64) (*User, error) {
	// call mysql driver
}

func (mud *MySQLUserDAO) wrapMySQLError(err error) error {
	if err == nil {
		return nil
	}

	switch err {
		case MySQLAccessDenied:
			return fmt.Errorf("MySQL access denied: %w", PermissionDenied)
		// ... other cases
		default:
		return fmt.Errorf("MySQL error: %w", Internal)
	}
}
```

## 中间填充信息

在上层与底层之间，难免会有一些中间层。业务越复杂，划分的层级越多，同层之间还可能存在相互依赖。结果就是函数调用栈变深。这里会出现两个问题：

1. 到达同一个底层方法的路径可能有多个，光看底层错误信息无法回溯问题触发过程
2. 不同层关心的内容不同，拥有的信息也不同，光看底层错误信息无法拿到完整信息

因此需要在中间层填充必要的信息，比如在下面的例子中：

```go
func (dus *DefaultUserService) GetUser(ctx context.Context, id int64) (*User, error) {
	user, err := dus.user.GetUser(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("UserService gets user %d: %w", id, err)
	}
	return user, nil
}
```

既明确了当前函数为 `UserService.GetUser`，也补充了查询的目标用户 `id`。

## 上层统一判断

当这些错误来到上层后，我们可以利用一个工具函数或 HTTP/gRPC middleware 来统一决定：

* 返回的错误码
* 返回的错误消息
* 打印的日志

```go
func (rt *Router) handleError(w http.ResponseWriter, err error) {
	if err == nil {
		return
	}

	var status int
	var message string

	switch {
	case errors.Is(err, InvalidArgument):
		// ...
	case errors.Is(err, NotFound):
		// ...
	case errors.Is(err, Internal):
		// ...
	default:
		// ...
	}

	w.WriteHeader(status)
	w.Write([]byte(message))
}
```

# 小结

* 错误的消费者：用户、程序、运维
* 错误就是值：错误可以被编程
* 标准库的演进：点 → 链表 → 树
* 业务服务中的错误处理实战：定义通用错误、底层转换标识、中间填充信息、上层统一判断

# 参考

* [MiddleMost: Failure is your Domain](https://middlemost.com/failure-is-your-domain/)
* [The Go Blog: Error handling and Go](https://blog.golang.org/error-handling-and-go)
* [The Go Blog: Errors are values](https://blog.golang.org/errors-are-values)
* [Joel on Software: Exceptions](https://www.joelonsoftware.com/2003/10/13/13/)
* [Working with Errors in Go 1.13](https://blog.golang.org/go1.13-errors)
* [New in Go 1.20: wrapping multiple errors](https://lukas.zapletalovi.com/posts/2022/wrapping-multiple-errors/)
