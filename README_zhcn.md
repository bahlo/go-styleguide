# Go Styleguide

本文是对
[Effective Go](https://golang.org/doc/effective_go.html) 的补充, 其条目来自于经年累月的会议上得到的经验和灵感。

## Table of contents

- [给错误添加上下文](#给错误添加上下文)
- [依赖管理](#依赖管理)
	- [使用 dep](#使用dep)
	- [使用 semantic 版本号](#使用semantic版本号)
	- [避免使用 gopkg.in](#避免使用gopkgin)
- [结构化的日志](#结构化的日志)
- [避免全局变量](#避免全局变量)
- [测试](#测试)
	- [使用assert库](#使用assert库)
	- [使用表驱动的测试](#使用表驱动的测试)
    - [避免 mock](#避免mock)
	- [避免使用 deepequal](#避免deepequal)
	- [不要测试非导出的函数](#不要测试非导出函数)
- [使用 linter](#使用linter)
- [使用 gofmt](#使用gofmt)
- [避免 side-effects](#避免side-effects)
- [尽量使用纯函数](#尽量使用纯函数)
- [避免接口臃肿](#避免接口臃肿)
- [Don't under-package](#dont-under-package)
- [处理信号](#处理信号)
- [分块组织import](#分块组织import)
- [避免不加修饰的 return](#避免不加修饰的return)
- [添加包的权威导入路径](#添加包的权威导入路径)
- [避免空接口](#避免空接口)
- [main 函数先行](#main函数先行)
- [使用 internal 包](#使用internal包)
- [避免使用 helper/util 的文件名、包名](#避免使用helper/util的文件名、包名)
- [将二进制内容嵌入到程序中](#内嵌二进制数据)
- [函数式的配置设置](#使用函数式的配置选项)

## 给错误添加上下文

**Don't:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

这种处理方式会导致错误信息不清晰，因为丢失了错误本来的上下文。

**Do:**
```go
import "github.com/pkg/errors" // for example

// ...

file, err := os.Open("foo.txt")
if err != nil {
	return errors.Wrap(err, "open foo.txt failed")
}
```

用自定义的 message 包装错误可以在错误从栈中向上“冒泡”的时候提供错误的上下文。
这么做并不一定总有意义。
如果你不确定一个返回的错误信息是否充分(译注：能够帮助判断问题在哪里)，
那么就对 error 进行 wrap。
确保根 error 在 wrap 之后仍然可以访问到，用于 type checking。

## 依赖管理

### 使用dep
由于 dep 已经 production ready，并且将来会成为官方的工具链之一
– [Sam Boyer at GopherCon 2017](https://youtu.be/5LtMb090AZI?t=27m57s)
因此开始使用 dep 吧。 [dep](https://github.com/golang/dep)

### 使用Semantic版本号
由于 dep 可以管理依赖版本，尽量使用 semver 对你的项目打 tag。
[Semantic Versioning](http://semver.org).

### 避免使用gopkgin
[gopkg.in](http://labix.org/gopkg.in) 是很棒很有用的工具，这个工具会将你的依赖打 tag，但其本来的设计并不是要与 dep 协作。
请直接 import，使用 dep 并在 `Gopkg.toml` 中指定版本。

## 结构化的日志

**Don't:**
```go
log.Printf("Listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**Do:**
```go
import "github.com/uber-go/zap" // for example

// ...

logger, _ := zap.NewProduction()
defer logger.Sync()
logger.Info("Server started",
	zap.Int("port", port),
	zap.String("env", env),
)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","ts":1501326297.511464,"caller":"Desktop/structured.go:17","msg":"Server started","port":80,"env":"production"}
```

这个例子并不是很有说服力，不过使用结构化的日志可以让你的日志无论是 debug 还是被日志收集 parse 都变得更容易。

## 避免全局变量

**Don't:**
```go
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```

全局变量会使测试难度增加，会使代码的可读性降低，每一个函数都能够访问这些全局变量(即使是那些根本就不需要操作全局变量的函数)。

**Do:**
```go
func main() {
	db := // ...
	http.HandleFunc("/drop", DropHandler(db))
	// ...
}

func DropHandler(db *sql.DB) http.HandleFunc {
	return func (w http.ResponseWriter, r *http.Request) {
		db.Exec("DROP DATABASE prod")
	}
}
```

使用高阶函数(high-order function)来按需注入依赖，而不是全局变量。

## 测试

### 使用assert库

**Don't:**
```go
func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	if (actual != expected) {
		t.Errorf("Expected %d, but got %d", expected, actual)
	}
}
```

**Do:**
```go
import "github.com/stretchr/testify/assert" // for example

func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	assert.Equal(t, expected, actual)
}
```

使用 assert 库使测试代码更可读，节省冗余的代码并提供稳定的错误输出。

### 使用表驱动的测试

**Don't:**
```go
func TestAdd(t *testing.T) {
	assert.Equal(t, 1+1, 2)
	assert.Equal(t, 1+-1, 0)
	assert.Equal(t, 1, 0, 1)
	assert.Equal(t, 0, 0, 0)
}
```

上面的程序看着还算简单，但是想找一个 fail 掉的 case 却非常麻烦，特别是有几百个 test case 的时候尤其如此。

**Do:**
```go
func TestAdd(t *testing.T) {
	cases := []struct {
		A, B, Expected int
	}{
		{1, 1, 2},
		{1, -1, 0},
		{1, 0, 1},
		{0, 0, 0},
	}

	for _, tc := range cases {
		t.Run(fmt.Sprintf("%d + %d", tc.A, tc.B), func(t *testing.T) {
			assert.Equal(t, t.Expected, tc.A+tc.B)
		})
	}
}
```

使用表驱动的 tests 结合子测试能够让你直接看到哪些 case 被测试，哪一个 case 失败了。
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=7m34s)

### 避免mock

**Don't:**
```go
func TestRun(t *testing.T) {
	mockConn := new(MockConn)
	run(mockConn)
}
```

**Do:**
```go
func TestRun(t *testing.T) {
	ln, err := net.Listen("tcp", "127.0.0.1:0")
	t.AssertNil(t, err)

	var server net.Conn
	go func() {
		defer ln.Close()
		server, err := ln.Accept()
		t.AssertNil(t, err)
	}()

	client, err := net.Dial("tcp", ln.Addr().String())
	t.AssertNil(err)

	run(client)
}
```

只在没有其它办法的时候才使用 mock，尽量使用真正的实现。
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=26m51s)

### 避免DeepEqual

**Don't:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	assert.True(t, reflect.DeepEqual(expected, actual))
}
```

**Do:**
```go
type myType struct {
	id         int
	name       string
	irrelevant []byte
}

func (m *myType) testString() string {
	return fmt.Sprintf("%d.%s", m.id, m.name)
}

func TestSomething(t *testing.T) {
	actual := &myType{/* ... */}
	expected := &myType{/* ... */}
	if actual.testString() != expected.testString() {
		t.Errorf("Expected '%s', got '%s'", expected.testString(), actual.testString())
	}
	// or assert.Equal(t, actual.testString(), expected.testString())
}
```

使用 `testString()` 这种方式来比较 struct，在结构体比较复杂，并且内含有逻辑上不影响相等判断的字段，那么就应该使用这种方式来进行相等判断。
这种方式只在结构体比较大，或者是“类树”的结构体比较中比较有用：
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=30m45s)

### 不要测试非导出函数

只对导出的函数进行测试，如果一个函数是 unexported 并且没有办法通过 exported 函数走到其逻辑，说明这个函数很可能会经常变动，没有必要进行测试。

## 使用linter

使用 linter， (e.g. [gometalinter](https://github.com/alecthomas/gometalinter)) 在提交你的项目之前先进行 lint 来帮助查找潜在的规范问题和代码错误。 

## 使用gofmt

在提交之前一定要对文件进行 gofmt，使用 `-s` 参数来简化代码。

## 避免side-effects

**Don't:**
```go
func init() {
	someStruct.Load()
}
```

[side-effects](https://en.wikipedia.org/wiki/Side_effect_(computer_science))
 指函数或者代码会改变其作用域外的内容或值的行为。只有在一些特定的情况下 side-effects 是允许的(比如：在命令行中解析 flags)

如果你想不出其它的办法来避免，那么就重新思考并尝试重构吧。

## 尽量使用纯函数

> 在计算机程序中，如果一个函数满足下面的几个条件，那么这个函数就是一个纯函数：
> 1. 这个函数在相同的参数下一定会产生相同的结果。即函数的返回值不依赖于任何隐藏在函数内的信息或者状态，而这些隐藏的内容在程序的运行期还可能会变化。且函数不应依赖于任何从 I/O 设备中输入的信息。
> 2. 对函数的返回结果进行操作不会引起任何语义上的副作用或者输出，比如导致可变对象的变化或者输出数据到 I/O 设备去。

– [Wikipedia](https://en.wikipedia.org/wiki/Pure_function)

**Don't:**
```go
func MarshalAndWrite(some *Thing) error {
	b, err := json.Marshal(some)
	if err != nil {
		return err
	}

	return ioutil.WriteFile("some.thing", b, 0644)
}
```

**Do:**
```go
// Marshal is a pure func (even though useless)
func Marshal(some *Thing) ([]bytes, error) {
	return json.Marshal(some)
}

// ...
```

纯函数并不一定在所有场景下都适用，但保证你用到的函数尽量都是纯函数能够让你的代码更易理解，且更容易 debug。

## 避免接口臃肿

**Don't:**
```go
type Server interface {
	Serve() error
	Some() int
	Fields() float64
	That() string
	Are([]byte) error
	Not() []string
	Necessary() error
}

func debug(srv Server) {
	fmt.Println(srv.String())
}

func run(srv Server) {
	srv.Serve()
}
```

**Do:**
```go
type Server interface {
	Serve() error
}

func debug(v fmt.Stringer) {
	fmt.Println(v.String())
}

func run(srv Server) {
	srv.Serve()
}
```

尽量使用小的 interface，并且在你的函数中只要求传入需要的 interface。

## Don't under-package

删除或者合并 package 要比将大的 package 分开容易得多。如果不确定一个包是否可以分开，那么最好去试一试。

## 处理信号

**Don't:**
```go
func main() {
	for {
		time.Sleep(1 * time.Second)
		ioutil.WriteFile("foo", []byte("bar"), 0644)
	}
}
```

**Do:**
```go
func main() {
	logger := // ...
	sc := make(chan os.Signal, 1)
	done := make(chan bool)

	go func() {
		for {
			select {
			case s := <-sc:
				logger.Info("Received signal, stopping application",
					zap.String("signal", s.String()))
				done <- true
				return
			default:
				time.Sleep(1 * time.Second)
				ioutil.WriteFile("foo", []byte("bar"), 0644)
			}
		}
	}()

	signal.Notify(sc, os.Interrupt, os.Kill)
	<-done // Wait for go-routine
}
```

对 os 的信号进行处理能够让我们 gracefully 地停止服务，关闭打开的文件和连接，并且能够防止因为服务的意外关闭而导致文件损坏或其它问题。


## 分块组织import

**Don't:**
```go
import (
	"encoding/json"
	"github.com/some/external/pkg"
	"fmt"
	"github.com/this-project/pkg/some-lib"
	"os"
)
```

**Do:**
```go
import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/some/external/pkg"

	"github.com/this-project/pkg/some-lib"
)
```

将 std，外部包和 internal 导入分开写，可以提高可读性。

## 避免不加修饰的return

**Don't:**
```go
func run() (n int, err error) {
	// ...
	return
}
```

**Do:**
```go
func run() (n int, err error) {
	// ...
	return n, err
}
```

命名返回的值对文档编写或者生成是有益的，不加任何修饰的 return 会让代码变得难读而易错。

## 添加包的权威导入路径

**Don't:**
```go
package sub
```

**Do:**
```go
package sub // import "github.com/my-package/pkg/sth/else/sub"
```

在注释中添加权威导入路径，能够给包添加上下文，也能够帮助用户更容易地导入你的包。

## 避免空接口

**Don't:**
```go
func run(foo interface{}) {
	// ...
}
```

空接口会让代码变得复杂而不清晰，只要能够不使用，就应该在任何时候避免。

## main函数先行

**Don't:**
```go
package main // import "github.com/me/my-project"

func someHelper() int {
	// ...
}

func someOtherHelper() string {
	// ...
}

func Handler(w http.ResponseWriter, r *http.Reqeust) {
	// ...
}

func main() {
	// ...
}
```

**Do:**
```go
package main // import "github.com/me/my-project"

func main() {
	// ...
}

func Handler(w http.ResponseWriter, r *http.Reqeust) {
	// ...
}

func someHelper() int {
	// ...
}

func someOtherHelper() string {
	// ...
}
```

将 `main()` 函数放在你文件的最开始，能够让阅读这个文件变得更加轻松。如果有 `init()` 函数的话，应该再放在 `main()` 之前。

## 使用internal包

如果你想创建一个 cmd，考虑将 libraries 移动到 `internal/` 包中，而避免这些不稳定可能经常会变化的库被其它项目引用。

## 避免使用helper/util的文件名、包名

使用清晰的命名，避免创建形如：`helper.go`，`util.go` 这样的文件名或者 package。

## 内嵌二进制数据

为了在部署阶段只有一个二进制文件，使用工具来将 templates 和其它静态内容嵌入到你的二进制文件中
(e.g. [github.com/jteeuwen/go-bindata](https://github.com/jteeuwen/go-bindata)).

## 使用函数式的配置选项

```go

func main() {
	// ...
	startServer(
		WithPort(8080),
		WithTimeout(1 * time.Second),
	)
}

type Config struct {
	port    int
	timeout time.Duration
}

type ServerOpt func(*Config)

func WithPort(port int) ServerOpt {
	return func(cfg *Config) {
		cfg.port = port
	}
}

func WithTimeout(timeout time.Duration) ServerOpt {
	return func(cfg *Config) {
		cfg.timeout = timeout
	}
}

func startServer(opts ...ServerOpt) {
	cfg := new(Config)
	for _, fn := range opts {
		fn(cfg)
	}

	// ...
}


```
