# Error

## 定义

在go源码的`builtin/builtin.go`文件中，定义了`error`接口：

```
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```

通常使用的`errors`相关的代码，在`errors/errors.go`文件中：

```
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
	return &errorString{text}
}
// errorString is a trivial implementation of error.
type errorString struct {
	s string
}
func (e *errorString) Error() string {
	return e.s
}
```

**由于`errors.New`返回的是errorString的指针，因此即使传入相同的文本，每次返回的error都是不同的。**这样做的好处是，防止不同人定义的error字符串内容相同，通过值判断的话，会出现意外的相同。因此用了 struct 和 取地址 防止这种意外情况的出现。



## Error vs Exception

C++ 引入了exception，但是无法知道被调用方会抛出什么异常。

Java有checked exception，方法的所有者必须声明可能的异常，由调用者处理。但可能没被正确处理，低质量的代码没有合理区分简单或灾难性错误，只要报错，全部抛异常。而且调用方在处理exception时，可能不管。或者直接抛Exception、RuntimeException这种粗粒度的，没有记录详细信息。

语言设计者出发点是好的，但使用的人不一定按照最佳实践来。



Go 处理异常的逻辑是不引入exception，但是支持多参数返回，一般返回值的最后一个就是一个error对象。如果一个函数返回了`(value, error)`，不能直接使用这个`value`，需要先判定`error`。除非对`value`不关心，才能忽略`error`。

Go的`panic`与其他语言的exception并不相同，在抛异常的时候，认为由调用者来处理。但是Go的`panic`意味着fatal error，不能由调用者来解决，意味着代码不能继续运行。

一般不会轻易在业务代码里调用`panic`，只有在强依赖的地方出错了，才会panic，即这个地方出错会严重影响整个服务，例如在刚启动的时候、配置项不符合预期。

> 如果数据库连接不上，业务类型为读多写少，这时候不应该`panic`退出程序。读多写少的场景属于弱依赖，读还可以从缓存读，不影响可用性，对于少量的写，返回错误即可，或者降级到写队列。如果退出程序，将导致整个服务不可用。

对于真正意外的情况，那些表示不可恢复的程序错误，例如索引越界、不可恢复的环境问题、栈溢出，我们才使用panic。对于其他的错误情况，我们应该是期望使用 error 来进行判定。



## Error Type

### Sentinel Error

在基础库中有大量预定义的可导出 Error，这个名字来源于计算机编程中**使用一个特定值来表示不可能进行进一步处理的做法**，例如bufio：

```
var (
	ErrInvalidUnreadByte = errors.New("bufio: invalid use of UnreadByte")
	ErrInvalidUnreadRune = errors.New("bufio: invalid use of UnreadRune")
	ErrBufferFull        = errors.New("bufio: buffer full")
	ErrNegativeCount     = errors.New("bufio: negative count")
)
```

一般会以 `${当前package名}:` 开头，方便溯源。

这种做法是最不灵活的，因为调用方必须用`==`将结果与预先声明的值进行比较，如果要提供更多的上下文，就会返回一个不同的错误，破坏相等性检查。

并且Sentinel Error 会成为API公共部分，在两个包之间创建了依赖。



### Error  types

Error type是实现了`error`接口的自定义类型，例如下面的`MyError`类型记录了文件和行号以展示发送了什么：

```
type MyError struct {
    Line int
    File string
    Msg string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("%s:%d: %s", e.File, e.Line, e.Msg)
}
```

在使用自定义错误类型时，通过类型断言进行转换，可以获取更多的上下文信息：

```
switch err := err.(type) {
	case nil:
	  // call succeeded, nothing to do
    case *MyError:
      fmt.Println("error occurred on line ", err.Line)
	default:
}
```

`os.PathError`就是这种方式的实例。



调用者要使用类型断言和类型 `switch`，就要让自定义的 error 变为 public。这种模型会导致和调用者产生强耦合，从而导致 API 变得脆弱。

结论是尽量避免使用 error types，虽然错误类型比 sentinel errors 更好，因为它们可以捕获关于出错的更多上下文，但是 error types 共享 error values 许多相同的问题。



### Opaque errors

最灵活的错误处理策略，因为它要求代码和调用者之间的耦合最少。这种风格称为**不透明错误处理**，因为虽然知道发生了错误，但没有能力看到错误的内部。作为调用者，关于操作的结果，您所知道的就是它起作用了，或者没有起作用(成功还是失败)。

这就是不透明错误处理的全部功能——**只需返回错误而不假设其内容**。

可以通过下面的方法断言错误实现了特定的行为，而不是断言错误是特定的类型或值，即Assert errors for behaviour, not type.

```
type Error interface {
    error
    Timeout() bool   // Is the error a timeout?
    Temporary() bool // Is the error temporary?
}

type temporary interface {
    Temporary() bool
}

func isTemporary(err error) bool {
    te, ok := err.(temporary)
    return ok && te.Temporary()
}
```



## Handling Error

### Wrap errors

```
func AuthenticateRequest(r *Request) error {
    return authticate(r.User)
}
```

如果 `authenticate `返回错误，则 `AuthenticateRequest`会将错误返回给调用方，调用者可能也会这样做，依此类推。在程序的顶部，程序的主体将把错误打印到屏幕或日志文件中，打印出来的只是：没有这样的文件或目录。没有任何上下文信息。

没有生成错误的`file:line` 信息，没有导致错误的调用堆栈的堆栈跟踪。因此在每个error处加上相关信息，以发现是哪个代码路径触发了文件未找到错误：

```
func AuthenticateRequest(r *Request) error {
    err := authticate(r.User)
	if err != nil {
        return fmt.Errorf("authenticate failed: %v", err)
	}
	return nil
}
```

但是这种模式与sentinel errors 或 type assertions 的使用不兼容，因为将错误值转换为字符串，将其与另一个字符串合并，然后将其转换回`fmt.Errorf` 破坏了原始错误，导致等值判定失败。



- 错误要被日志记录
- 应用程序处理错误，保证100%完整性
- 之后不再报告当前错误，即**每个错误只处理一次就可以**，记录日志属于一种处理方式



### pkg/errors

``` 
func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
        return nil, errors.Wrap(err, "open failed")
	}
	defer f.close()
	
	buf, err := ioutil.ReadAll(f)
	if err != nil {
        return nil, errors.Wrap(err, "read failed")
	}
	return buf, nil
}

func ReadConfig() ([]byte, error) {
    home := os.Getenv("HOME")
    config, err := ReadFile(filepath.Join(home, ".config"))
    return config, errors.WithMessage(err, "could not read config")
}

func main() {
	_, err := ReadConfig()
    if err != nil {
        fmt.Printf("original error: %T %v\n", errors.Cause(err), errors.Cause(err))
        fmt.Printf("stack trace:\n%+v\n", err)
        os.Exit(1)
    }
}
```

`errors.Wrap`会记录堆栈信息：

```
// Wrap returns an error annotating err with a stack trace
// at the point Wrap is called, and the supplied message.
// If err is nil, Wrap returns nil.
func Wrap(err error, message string) error {
	if err == nil {
		return nil
	}
	err = &withMessage{
		cause: err,
		msg:   message,
	}
	return &withStack{
		err,
		callers(),
	}
}
```

`errors.WithMessage`不会记录堆栈信息，但可以在已有error上附加新的信息：

```
// WithMessage annotates err with a new message.
// If err is nil, WithMessage returns nil.
func WithMessage(err error, message string) error {
	if err == nil {
		return nil
	}
	return &withMessage{
		cause: err,
		msg:   message,
	}
}
```



- `pkg/errors`库中的`New`和`Errorf`都会记录堆栈信息
- 如果调用其他  包内的函数（底层的），通常直接返回即可，不用再包装堆栈信息，否则会有两倍的堆栈信息
- 如果和其他库（包括标准库）协作，考虑用`Wrap`或`Wrapf`保存堆栈信息
- 直接返回错误，而不是每个错误产生的地方都打日志
- 在程序的顶部或者工作的goroutine顶部（请求入口），使用`%+v`打印堆栈详情
- 使用`errors.Cause`获取root error，再和sentinel error判定



- **Packages that are reusable across many projects only return root error values.**

  ​    ***选择 wrap error 是只有 applications 可以选择应用的策略。具有最高可重用性的包只能返回根错误值。此机制与 Go 标准库中使用的相同(kit*** ***库的*** ***sql.ErrNoRows)。***

- **If the error is not going to be handled, wrap and return up the call stack.**
  ***这是关于函数/方法调用返回的每个错误的基本问题。如果函数/方法不打算处理错误，那么用足够的上下文 wrap errors 并将其返回到调用堆栈中。例如，额外的上下文可以是使用的******输入参数******或失败的查询******语句******。确定您记录的上下文是足够多还是太多的一个好方法是检查日志并验证它们在开发期间是否为您工作。***

- **Once an error is handled, it is not allowed to be passed up the call stack any longer.**

  ​    ***一旦确定函数/方法将处理错误，错误就不再是错误。如果函数/方法仍然需要发出返回，则它不能返回错误值。它应该只返回零(******比如降级处理中，你返回了降级数据，然后需要*** ***return nil)。***




## go1.13

为 *errors* 和 *fmt* 标准库包引入了新特性，以简化处理包含其他错误的错误。

- 包含另一个错误的error 可以实现返回底层错误的 *Unwrap* 方法。如果 *e1.Unwrap()* 返回 *e2*，那么我们说 *e1* 包装 *e2*，您可以展开 *e1* 以获得 *e2*。

- `errors`包还提供了两个用于检查错误的新函数：`Is`和`As`。

  `Is`会对传入的error调用`Unwrap`，递归获取根error，然后再和target error进行比较。

  `As`实现的功能类似于对error struct进行类型断言

  ```
  var e *QueryError
  if errors.As(err, &e) {
      // err is a *QueryError, and e is set to the error's value
  }
  ```

- 支持`%w`谓词，使用`%v`谓词会丢失原始error。`%w`包装的错误可用于`Is`和`As`

  ```
  err := fmt.Errorf("access denied: %w", ErrPermission)
  if errors.Is(err, ErrPermission) {...}
  ```

  

```
type wrapError struct {
	msg string
	err error
}

func (e *wrapError) Error() string {
	return e.msg
}

func (e *wrapError) Unwrap() error {
	return e.err
}
```



### 使用自定义error

`errors.Is`和`Unwrap`定义如下：

```
func Is(err, target error) bool {
	if target == nil {
		return err == target
	}

	isComparable := reflectlite.TypeOf(target).Comparable()
	for {
		if isComparable && err == target {
			return true
		}
		if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
			return true
		}
		// TODO: consider supporing target.Is(err). This would allow
		// user-definable predicates, but also may allow for coping with sloppy
		// APIs, thereby making it easier to get away with them.
		if err = Unwrap(err); err == nil {
			return false
		}
	}
}

func Unwrap(err error) error {
	u, ok := err.(interface {
		Unwrap() error
	})
	if !ok {
		return nil
	}
	return u.Unwrap()
}
```

要使用`Is`，只需要实现`Is(error) bool`这个方法

```
type Error struct {
    Path string
    User string
}

func (e *Error) Is(target error) bool {
    t, ok := target.(*Error)
    if !ok {
        return false
    }
    
    return (e.Path == t.Path || t.Path == "") && (e.User == t.User || t.User == "")
}
```



在对外提供接口的时候，通过`%w`wrap错误，这样可以避免判断错误时依赖于指定的错误值，可以对错误进行更多的封装。

```
var ErrPermission = errors.New("permission denied")

func DoSomething() error {
    if !userHasPermission() {
        // 如果直接返回 ErrPermission，调用者判断error时，就依赖了一个指定值
        //  if err := pkg.DoSomething(); err == pkg.ErrPermission
        return fmt.Errorf("%w", ErrPermission)
        // 调用者这样调用：
        // if err := pkg.DoSomething(); errors.Is(err, pkg.ErrPermission)
    }
}
```



### pkg/errors

虽然1.13 对error做了很多改进，但error仍不支持携带堆栈信息。不过`pkg/errors`对`Is`和`Unwrap`接口也进行了封装，内部的对象也实现了相关的接口：

```
// errors/go113.go
import (
	stderrors "errors"
)

func Is(err, target error) bool { return stderrors.Is(err, target) }
func Unwrap(err error) error {
	return stderrors.Unwrap(err)
}
```





## go2

[GO2对error的提案](https://go.googlesource.com/proposal/+/master/design/29934-error-values.md)









