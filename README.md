# Go Styleguide

This serves as a supplement to
[Effective Go](https://golang.org/doc/effective_go.html), based on years of
experience and inspiration/ideas from conference talks.

## Table of contents

- [Add context to errors](#add-context-to-errors)
- [Consistent error and log messages](#consistent-error-and-log-messages)
- [Dependency management](#dependency-management)
	- [Use dep](#use-dep)
	- [Use Semantic Versioning](#use-semantic-versioning)
	- [Avoid unnessary version lockdown](#avoid-unnessary-version-lockdown)
	- [Avoid gopkg.in](#avoid-gopkgin)
- [Structured logging](#structured-logging)
- [Avoid global variables](#avoid-global-variables)
- [Testing](#testing)
	- [Use testify as assert libary](#use-testify-as-assert-libary)
	- [Use subtests to structure functional tests](#use-sub-tests-to-structure-functional-tests)
	- [Use table-driven tests](#use-table-driven-tests)
	- [Avoid mocks](#avoid-mocks)
	- [Avoid DeepEqual](#avoid-deepequal)
	- [Avoid testing unexported funcs](#avoid-testing-unexported-funcs)
	- [Add examples to your test files to demonstrate usage](#add-examples-to-your-test-files-to-demonstrate-usage)
- [Use linters](#use-linters)
- [Use goimports](#use-goimports)
- [Use meaningful variable names](#use-meaningful-variable-names)
- [Avoid side effects](#avoid-side-effects)
- [Favour pure functions](#favour-pure-functions)
- [Don't over-interface](#dont-over-interface)
- [Don't under-package](#dont-under-package)
- [Handle signals](#handle-signals)
- [Divide imports](#divide-imports)
- [Avoid unadorned return](#avoid-unadorned-return)
- [Use canonical import path](#use-canonical-import-path)
- [Avoid empty interface](#avoid-empty-interface)
- [Main first](#main-first)
- [Use internal packages](#use-internal-packages)
- [Avoid helper/util](#avoid-helperutil)
- [Embed binary data](#embed-binary-data)
- [Use io.WriteString](#use-iowritestring)
- [Use functional options](#use-functional-options)
- [Structs](#structs)
	- [Use named structs](#use-named-structs)
	- [Avoid new keyword](#avoid-new-keyword)
- [Consistent header naming](#consistent-header-naming)

## Add context to errors

**Don't:**
```go
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```

Using the approach above can lead to unclear error messages because of missing
context.

**Do:**
```go
import "github.com/pkg/errors" // for example

// ...

file, err := os.Open("foo.txt")
if err != nil {
	return errors.Wrap(err, "open foo.txt failed")
}
```

Wrapping errors with a custom message provides context as it gets propagated up
the stack.
This does not always make sense.
If you're unsure if the context of a returned error is at all times sufficient,
wrap it.
Make sure the root error is still accessible somehow for type checking.

## Consistent error and log messages
Error messages should start with a lowercase letter and should not end with a `.`. See [Wiki page on Errors](https://github.com/golang/go/wiki/Errors) for reference. For consistency the same logic should be applied for log messages.

**Don't:**
```go
logger.Print("Something went wrong.")
ReadFailError := errors.New("Could not read file")
```

**Do:**
```go
logger.Print("something went wrong")
ErrReadFailed := errors.New("could not read file")
```


## Dependency management

### Use dep
Use [dep](https://github.com/golang/dep), since it's production ready and will
soon become part of the toolchain.
– [Sam Boyer at GopherCon 2017](https://youtu.be/5LtMb090AZI?t=27m57s)

### Use Semantic Versioning
Since `dep` can handle versions, tag your packages using
[Semantic Versioning](http://semver.org).  
The git tag for your go package should have the format `v<major>.<minor>.<patch>`, e.g., `v1.0.1`.

### Avoid unnessary version lockdown
If possible only lock the major version in the `Gopkg.toml` file.

**Don't:**
```toml
[[constraint]]
  name = "github.com/stretchr/testify"
  version = "1.1.4"
```

**Do:**
```toml
[[constraint]]
  name = "github.com/stretchr/testify"
  version = "^1.1.4"
```

### Avoid gopkg.in
While [gopkg.in](http://labix.org/gopkg.in) is a great tool and was really
useful, it tags one version and is not meant to work with `dep`.
Prefer direct import and specify version in `Gopkg.toml`.

## Structured logging

**Don't:**
```go
log.Printf("Listening on :%d", port)
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// 2017/07/29 13:05:50 Listening on :80
```

**Do:**
```go
import "github.com/sirupsen/logrus"
// ...

logger.WithField("port", port).Info("Server is listening")
http.ListenAndServe(fmt.Sprintf(":%d", port), nil)
// {"level":"info","msg":"Server is listening","port":"7000","time":"2017-12-24T13:25:31+01:00"}
```

This is a harmless example, but using structured logging makes debugging and log
parsing easier.

## Avoid global variables

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

Global variables make testing and readability hard and every method has access
to them (even those, that don't need it).

**Do:**
```go
func main() {
	db := // ...
	handlers := Handlers{DB: db}
	http.HandleFunc("/drop", handlers.DropHandler)
	// ...
}

type Handlers struct {
	DB *sql.DB
}

func (h *Handlers) DropHandler(w http.ResponseWriter, r *http.Request) {
	h.DB.Exec("DROP DATABASE prod")
}
```
Use structs to encapsulate the variables and make them available only to those functions that actually need them by making them methods implemented for that struct.

Alternatively higher-order functions can be used to inject dependencies via closures.
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

If you really need global variables or constants, e.g., for defining errors or string constants, put them at the top of your file.

**Don't:**
```go
import "xyz"

func someFunc() {
	//...
}

route = "/some-route"

func someOtherFunc() {
	// usage of route
}

var NotFoundErr = errors.New("not found")

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```

**Do:**
```go
import "xyz"

route = "/some-route"

var NotFoundErr = errors.New("not found")

func someFunc() {
	//...
}

func someOtherFunc() {
	// usage of route
}

func yetAnotherFunc() {
	// usage of NotFoundErr
}
```


## Testing

### Use testify as assert libary

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
import "github.com/stretchr/testify/assert"

func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	assert.Equal(t, expected, actual)
}
```

Using assert libraries makes your tests more readable, requires less code and
provides consistent error output.

### Use sub-tests to structure functional tests
**Don't:**
```go
func TestSomeFunctionSuccess(t *testing.T) {
	// ...
}

func TestSomeFunctionWrongInput(t *testing.T) {
	// ...
}
```

**Do:**
```go
func TestSomeFunction(t *testing.T) {
	t.Run("success", func(t *testing.T){
		//...
	})

	t.Run("wrong input", func(t *testing.T){
		//...
	})
}
```

### Use table driven tests

**Don't:**
```go
func TestAdd(t *testing.T) {
	assert.Equal(t, 1+1, 2)
	assert.Equal(t, 1+-1, 0)
	assert.Equal(t, 1, 0, 1)
	assert.Equal(t, 0, 0, 0)
}
```

The above approach looks simpler, but it's much harder to find a failing case,
especially when having hundreds of cases.

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
			t.Parallel()
			assert.Equal(t, t.Expected, tc.A+tc.B)
		})
	}
}
```

Using table driven tests in combination with subtests gives you direct insight
about which case is failing and which cases are tested.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=7m34s)

Running subtests in parallel allows you to have a lot more test cases and still get those awesomely fast go build times.
– [The Go Blog](https://blog.golang.org/subtests)

### Avoid mocks

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

Only use mocks if not otherwise possible, favor real implementations.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=26m51s)

### Avoid DeepEqual

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

Using `testString()` for comparing structs helps on complex structs with many
fields that are not relevant for the equality check.
This approach only makes sense for very big or tree-like structs.
– [Mitchell Hashimoto at GopherCon 2017](https://youtu.be/8hQG7QlcLBk?t=30m45s)

Google open sourced their [go-cmp](http://github.com/google/go-cmp) package as a more powerful and safer alternative to `reflect.DeepEqual`.
– [Joe Tsai](https://twitter.com/francesc/status/885630175668346880).

### Avoid testing unexported funcs

Only test unexported funcs if you can't access a path via exported funcs.
Since they are unexported, they are prone to change.

### Add examples to your test files to demonstrate usage
```go
func ExamleSomeInterface_SomeMethod(){
	instance := New()
	result, err := instance.SomeMethod()
	fmt.Println(result, err)
	// Output: someResult, <nil>
}
```

## Use linters

Use all the linters included in [gometalinter](https://github.com/alecthomas/gometalinter) to lint your projects before committing.
```bash
# Installation
go get -u gopkg.in/alecthomas/gometalinter.v2
gometalinter.v2 --install

# Usage in the project workspace
gometalinter.v2 --vendor ./...
```

## Use goimports

Only commit gofmt'd files. Use `goimports` for this to format/update the import statements as well.

## Use meaningful variable names
Avoid single-letter variable names. They may seem more readable to you at the moment of writing but they make the code hard to understand for your colleagues and your future self.

**Don't:**
```go
func findMax(l []int) int {
	m := l[0]
	for _, n := range l {
		if n > m {
			m = n
		}
	}
	return m
}
```

**Do:**
```go
func findMax(inputs []int) int {
	max := inputs[0]
	for _, value := range inputs {
		if value > max {
			max = value
		}
	}
	return max
}
```
Single-letter variable names are fine in the following cases.
* They are absolut standard like ...
	* `t` in tests
	* `r` and `w` in http request handlers
	* `i` for the index in a loop
* They name the receiver of a method, e.g., `func (s *someStruct) myFunction(){}`

Of course also too long variables names like `createInstanceOfMyStructFromString` should be avoided.

## Avoid side-effects

**Don't:**
```go
func init() {
	someStruct.Load()
}
```

Side effects are only okay in special cases (e.g. parsing flags in a cmd).
If you find no other way, rethink and refactor.

## Favour pure functions

> In computer programming, a function may be considered a pure function if both of the following statements about the function hold:
> 1. The function always evaluates the same result value given the same argument value(s). The function result value cannot depend on any hidden information or state that may change while program execution proceeds or between different executions of the program, nor can it depend on any external input from I/O devices.
> 2. Evaluation of the result does not cause any semantically observable side effect or output, such as mutation of mutable objects or output to I/O devices.

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

This is obviously not possible at all times, but trying to make every possible
func pure makes code more understandable and improves debugging.

## Don't over-interface

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

Favour small interfaces and only expect the interfaces you need in your funcs.

## Don't under-package

Deleting or merging packages is far more easier than splitting big ones up.
When unsure if a package can be split, do it.

## Handle signals

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

Handling signals allows us to gracefully stop our server, close open files and
connections and therefore prevent file corruption among other things.

## Divide imports

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

	"git.fastbill.com/this-project/pkg/some-lib"

	"git.fastbill.com/another-project/pkg/some-lib"
	"git.fastbill.com/yet-another-project/pkg/some-lib"

	"github.com/some/external/pkg"
	"github.com/some-other/external/pkg"
)
```

Divide imports into four groups sorted from internal to external for readability:
1. Standard library
2. Project internal packages
3. Company internal packages
4. External packages

## Avoid unadorned return

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

Named returns are good for documentation, unadorned returns are bad for
readability and error-prone.

## Use canonical import path

**Don't:**
```go
package sub
```

**Do:**
```go
package sub // import "github.com/my-package/pkg/sth/else/sub"
```

Adding the canonical import path adds context to the package and makes
importing easy.

## Avoid empty interface

**Don't:**
```go
func run(foo interface{}) {
	// ...
}
```

Empty interfaces make code more complex and unclear, avoid them where you can.

## Main first

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

Putting `main()` first makes reading the file a lot more easier. Only the
`init()` function should be above it.

## Use internal packages

If you're creating a cmd, consider moving libraries to `internal/` to prevent
import of unstable, changing packages.

## Avoid helper/util

Use clear names and try to avoid creating a `helper.go`, `utils.go` or even
package.

## Embed binary data

To enable single-binary deployments, use tools to add templates and other static
assets to your binary
(e.g. [github.com/jteeuwen/go-bindata](https://github.com/jteeuwen/go-bindata)).

## Use `io.WriteString`
A number of important types that satisfy `io.Writer` also have a `WriteString`
method, including `*bytes.Buffer`, `*os.File` and `*bufio.Writer`. `WriteString`
is behavioral contract with implicit assent that passed string will be written
in efficient way, without a temporary allocation. Therefore using
`io.WriteString` may improve performance at most, and at least string will be
written in any way.

**Don't:**
```go
var w io.Writer = new(bytes.Buffer)
str := "some string"
w.Write([]byte(str))
```

**Do:**
```go
var w io.Writer = new(bytes.Buffer)
str := "some string"
io.WriteString(w, str)
```

## Use functional options

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

## Structs
### Use named structs
If a struct has more than one field include field names when instantiating it.

**Don't:**
```go
params := request.Params{
	"http://example.com",
	"POST",
	map[string]string{"Authentication": "someToken"}
	someStruct,
}
```

**Do:**
```go
params := request.Params{
	URL: "http://example.com",
	Method: "POST",
	Headers: map[string]string{"Authentication": "someToken"}
	Body: someStruct,
}
```

### Avoid new keyword
Using the normal syntax instead of the `new` keyword makes it more clear what is happening: a new instance of the struct is created `MyStruct{}` and we get the pointer for it with `&`.

**Don't:**
```go
s := new(MyStruct)
```

**Do:**
```go
s := &MyStruct{}
```

## Consistent header naming
**Don't:**
```go
r.Header.Get("authorization")
w.Header.Set("Content-type")
w.Header.Set("content-type")
w.Header.Set("content-Type")
```

**Do:**
```go
r.Header.Get("Authorization")
w.Header.Set("Content-Type")
```
