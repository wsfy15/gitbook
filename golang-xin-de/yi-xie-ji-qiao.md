# 一些技巧

* `type`（类型） VS `kind`（种类）

  ```
  type example struct {
  	field1 type1
  }
  ```
  
对于这个结构体，它的`type`是`example`，`kind`是`struct`。这里的 `kind `可以被看成一个 `type `的 `type`。
  
**在 Go 里所有 structs 都是相同的 kind，但不是相同的 Type。**
  
  对于原始类型（`int、float、string`等），它们的`kind`和`type`是相同的。

* 类型是程序员定义的关于数据和函数的元数据。种类是编译器和运行时定义的关于数据和函数的元数据。

  ```
  type VersionType string
  var Version VersionType // Version的type是VersionType，kind是string
  ```

  运行时和编译器根据 `Kind` 来分别给变量和函数分配内存或栈空间。

* `struct`与`int/float`同属**值类型**，值类型最重要的特点是在进行赋值时，新变量会得到一份拷贝后的值。 使用`for range` 遍历 `[]struct` `map[xx]struct` 时，得到的也是一份拷贝，因此下面的代码过不了编译，因为修改的是原有的拷贝，即使编译器允许，`map`中的`struct`值也不会改变，所以编译器直接禁止这种情况。

  ```text
  m := map[int]student{
     1: {name: "1"},
  }
  m[1].name = "2" // cannot assign to struct field m[1].name in map
  ```

  有以下两种解决方法：

  * 使用指针：`map[int]*student`
  * 使用临时变量：

    ```text
    m := map[int]student{1: {name: "1"}}
    tmp := m[1]
    tmp.name = "2"
    m[1] = tmp
    ```

* 使用[原始字符串字面值](https://golang.org/ref/spec#raw_string_lit)，可以跨越多行并包含引号

  ```text
  wantError := `unknown error:"test"
  123`
  // 相当于 wantError := "unknown name:\"test\"\n123"
  ```

* 初始化Struct引用时，使用`&T{}`代替`new(T)`，以使其与结构体初始化一致

  ```text
  sval := T{name: "foo"}
  sptr := &T{Name: "bar"}
  ```

* 初始化map时，尽可能用`make`，因为还可以指定初始化容量，尽量指定 map 容量。 但如果map包含固定的元素列表，则使用 map literals\(map 初始化列表\) 初始化映射。

  ```text
  m := map[T1]T2{
    k1: v1,
    k2: v2,
    k3: v3,
  }
  // 而不是make方式
  m := make(map[T1]T2, 3)
  m[k1] = v1
  m[k2] = v2
  m[k3] = v3
  ```

* getter：若字段名为小写开头，getter方法名字命名使用同名但大写开头，而不用`getXXX`；setter：使用`setXXX`命名

  ```text
  type Obj struct {
      owner string
  }

  func(this Obj) Owner() string {
      return this.owner
  }

  owner := obj.Owner()
  if owner != xxx {
      obj.SetOwner(user)
  }
  ```

* 只包含一个方法的接口应当以该方法的名称加上 `- er` 后缀来命名，如 Reader、Writer、 Formatter、CloseNotifier 等

* Go使用分号`;`结束语句，但并不需要编写代码时写分号，而是由词法分析器自动插入分号。

  插入规则：如果新行前的标记为语句的末尾，则插入分号。末尾包括标识符（包括 `int`和 `float64`这类的单词）、数值或字符串常量之类的基本字面或以下标记之一

  `break continue fallthrough return ++ -- ) }`

  这也是为什么不要将`if、for、switch、select`这些语句的左大括号放在下一行的原因。

* `range`可以解析UTF-8字符串 将每个独立的 Unicode 码点分离出来。错误的编码将占用一个字节，并以符文 `U+FFFD` 来代替。

  ```text
  for pos, char := range "日本 \x80 語" { // \x80 is an illegal UTF-8 encoding
    fmt.Printf("character %#U starts at byte position %d\n", char, pos)
  }
  character U+65E5 '日' starts at byte position 0
  character U+672C '本' starts at byte position 3
  character U+0020 ' ' starts at byte position 6
  character U+FFFD '�' starts at byte position 7
  character U+0020 ' ' starts at byte position 8
  character U+8A9E '語' starts at byte position 9
  ```

* `++`和`--`是语句，而不是表达式。

  ```text
  a[i++] = b[j--] // wrong
  a[i] = b[j]
  i++
  b--
  ```

* `switch`的`case`语句除了可以是常量或整数，还可以是条件表达式\(当`switch`后面没有表达式时，将匹配为`true`的`case`语句，可以看作多个`if-else`\)

  ```text
  func unhex(c byte) byte {
    switch {
    case '0' <= c && c <= '9':
      return c - '0'
    case 'a' <= c && c <= 'f':
      return c - 'a' + 10
    case 'A' <= c && c <= 'F':
      return c - 'A' + 10
    }
    return 0
  }
  ```

* `switch`的`case`可以通过逗号分隔 列举相同的处理条件

  ```text
  switch c {
    case ' ', '?', '&', '=', '#', '+', '%':
      return true
  }
  ```

* `switch`的每个`case`最后不需要`break`，即它不会自动下溯。可以通过`break`结束`switch`，以及跳出包含该`switch的for`循环

  ```text
  Loop:
  for i := 0; i < 10; i++ {
    switch {
    case i < 2:
      fmt.Println(i)
    case i >2 && i < 4:
      break Loop // 会跳出for循环
    case i >= 4:
      fmt.Println(i)
      break // 结束当前的switch
      fmt.Println(i)
    }
  }
  ```

* `continue`只能在循环中使用

* `switch`可以判断接口变量的动态类型

  ```text
  var t interface{}
  t = functionOfSomeType()
  switch t := t.(type) { // 通过圆括号中的关键字 type 使用类型断言语法
  default:
    fmt.Printf("unexpected type %T", t) // %T 输出 t 是什么类型
  case bool:
    fmt.Printf("boolean %t\n", t) // t 是 bool 类型
  case int:
    fmt.Printf("integer %d\n", t) // t 是 int 类型
  case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t 是 *bool 类型
  case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t 是 *int 类型
  }
  ```

* `defer`的函数若有参数、接收者，在推迟执行时就会求值，而不是在调用执行时才求值。被推迟的函数按照后进先出的顺序执行。

  ```text
  for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i) // 4 3 2 1 0
  }
  ```

* Go 提供了两种分配原语，即内建函数 `new`和 `make`。

  * `new`不会初始化内存，只会将内存置零。也就是说，`new(T)`会为类型为 `T` 的新项分配已置零的内存空间， 并返回它的地址，也就是一个类型为 `*T` 的值，即`T`的指针。既然 `new`返回的内存已置零，那么当你设计数据结构时， 每种类型的零值就不必进一步初始化了，这意味着该数据结构的使用者只需用 `new`创建一个新的对象就能正常工作。例如，对于`bytes.Buffer`， 零值的 Buffer 就是已准备就绪的缓冲区；零值的 `sync.Mutex` 就已经被定义为已解锁的互斥锁。
  * `make`只用于创建slice、map和channel，并返回类型为 `T`（而非 `*T` ）的一个已初始化 （而非置零）的值。因为这三种类型本质上为引用数据类型，它们在使用前必须初始化。

    `make`不会返回指针，如果需要指针，就得使用`new`。

  ```text
  var p *[]int = new([]int) // *p == nil  基本没用
  *p = make([]int, 100, 100)

  var v []int = make([]int, 100) // 切片 v 现在引用了一个具有 100 个 int 元素的新数组
  ```

* 数组的大小是其类型的一部分。类型 `[10]int` 和`[20]int` 是不同的。

* 二维切片的构造方式：
  * 独立分配每一个切片：

    ```text
    picture := make([][]uint8, rows) 
    for i := range picture {
      picture[i] = make([]uint8, cols)
    }
    ```

  * 一次分配，若切片会增长或收缩，则不要使用这种方法：

    ```text
    picture := make([][]uint8, rows) 
    pixels := make([]uint8, rows*cols) 

    for i := range picture {
      picture[i], pixels = pixels[:cols], pixels[cols:]
    }
    ```
  
* `map`的键可以是任何**支持相等性操作符的类型**， 如整数、浮点数、复数、字符串、指针、接口、结构以及数组，但切片不行，因为切片的相等性未定义。

* `print`中的`%v`占位符

  ```text
  type T struct {
    a int
    b float64
    c string
  }
  t := &T{ 7, -2.35, "abc\tdef" }
  fmt.Printf("%v\n", t) 
  // &{7 -2.35 abc def}

  fmt.Printf("%+v\n", t) // 会为结构体的每个字段添上字段名
  // &{a:7 b:-2.35 c:abc def}

  fmt.Printf("%#v\n", t) // 完全按照 Go 的语法打印
  // &main.T{a:7, b:-2.35, c:"abc\tdef"}

  fmt.Printf("%#v\n", timeZone)
  // map[string] int{"CST":-21600, "PST":-28800, "EST":-18000, "UTC":0, "MST":-25200}
  ```

* `%T`可以打印变量的**类型**

* 对于`string`或 `[]byte` 值， 可使用 `%q` 产生**带引号的字符串**；而格式 `%#q` 会尽可能使用**反引号**。 `%q`也可用于整数和`rune`，会产生一个带单引号的`rune`常量

* `%x` 可用于字符串、字节数组以及整数，并生成一个十六进制字符串， 而带空格的格式`% x`还会在字节之间插入空格

* 不要在结构体的`String`方法内直接调用`Sprintf`方法，因为会导致无限递归。可以通过将`Sprintf`的实参转换为基本的字符串类型后再调用。

  ```text
  type MyString string
  func (m MyString) String() string {
    // return fmt.Sprintf("MyString=%s", m) // 会无限递归
    return fmt.Sprintf("MyString=%s", string(m)) // ok
  }
  ```

* 常量只能是数字、字符（符文）、字符串或布尔值。由于编译时的限制， 定义它们的表达式必须也是可被编译器求值的常量表达式。例如 `1<<3` 就是一个常量表达式，而`math.Sin(math.Pi/4)`则不是，因为对 `math.Sin` 的函数调用在运行时才会发生。 `iota`可为表达式的一部分，而表达式可以被隐式地重复，这样也就更容易构建复杂的值的集合了。

  ```text
  type ByteSize float64
  const (
      _ = iota // 忽略第一个值
      KB ByteSize = 1 << (10 * iota) // 2^10 
      MB // 2^20
      GB
      TB
      PB
      EB
      ZB
      YB
  )
  ```

* 几乎所有类型都可以定义方法

  ```text
  type HandlerFunc func(ResponseWriter, *Request)
  func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
    f(w, req)
  }

  type Chan chan *http.Request
  func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    ch <- req
    fmt.Fprint(w, "notification sent")
  }

  type Counter int
  func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    *ctr++
    fmt.Fprintf(w, "counter = %d\n", *ctr)
  }
  ```

* 有时导入某个包只是为了其副作用， 而没有任何明确的使用。例如，在`net/http/pprof`包的 `init`函数中记录了 HTTP 处理程序的调试信息。它有个可导出的 API， 但大部分客户端只需要该处理程序的记录和通过 Web 网页访问数据。只为了其副作用来导入该包， 只需将包重命名为空白标识符：

  ```text
  import _ "net/http/pprof"
  ```

* 利用无缓冲channel同步交换数据

  ```text
  c := make(chan int) // Allocate a channel.
  go func() {
    list.Sort()
    c <- 1 // Send a signal; value does not matter.
  }()
  doSomethingForAWhile()
  <-c // Wait for sort to finish; discard sent value
  ```

  接收者在收到数据前会一直阻塞。若信道是不带缓冲的，那么在接收者收到值前， 发送者会一直阻塞；若信道是带缓冲的，则发送者仅在值被复制到缓冲区前阻塞； 若缓冲区已满，发送者会一直等待直到某个接收者取出一个值为止。

* 带缓冲的channel可被用作信号量，例如限制吞吐量。

  ```text
  var sem = make(chan int, MaxOutstanding)

  func handle(r *Request) {
    sem <- 1 // sem需要有位置才能继续往下执行
    process(r) // May take a long time.
    <-sem // Done; enable next request to run.
  }

  func Serve(queue chan *Request) {
    for {
      req := <-queue
      go handle(req) // Don't wait for handle to finish.
    }
  }
  ```

  在`Serve`函数中，存在资源消耗问题，如果请求来得很快，就会一直创建go协程。因此，可以把信号量提前到`Serve`函数中。

  ```text
  func Serve(queue chan *Request) {
    for req := range queue {
      sem <- 1
      go func(req *Request) {
        process(req)
        <-sem
      }(req)
    }
  }
  ```

* 另一种管理资源的好方法就是启动固定数量的 `handle`Go协程，一起从请求信道中读取数据。Go 协程的数量限制了同时调用 `process`的数量。`Serve`同样会接收一个通知退出的信道， 在启动所有 Go 程后，它将阻塞并暂停从`clientRequests`信道中接收消息。

  ```text
  func handle(queue chan *Request) {
    for r := range queue {
      process(r)
    }
  }

  func Serve(clientRequests chan *Request, quit chan bool) {
    // Start handlers
    for i := 0; i < MaxOutstanding; i++ {
      go handle(clientRequests)
    }
    <-quit // Wait to be told to exit.
  }
  ```

* 实现一个速率有限、并行、非阻塞 RPC 系统的 框架

  ```text
  type Request struct {
    args []int
    f func([]int) int
    resultChan chan int
  }

  func sum(a []int) (s int) {
    for _, v := range a {
      s += v
    }
    return
  }

  // 客户端提供一个函数及其实参，此外在Request对象中还有个接收应答的channel
  request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
  // Send request
  clientRequests <- request
  // Wait for response.
  fmt.Printf("answer: %d\n", <-request.resultChan)

  // server side
  func handle(queue chan *Request) {
    for req := range queue {
      req.resultChan <- req.f(req.args)
    }
  }
  ```

* 在多 CPU 核心上实现并行计算。如果计算过程能够被分为几块可独立执行的过程，它就可以在每块计算结束时向信道发送信号，从而实现并行处理。

  ```text
  type Vector []float64
  // Apply the operation to v[i], v[i+1] ... up to v[n-1].
  func (v Vector) DoSome(i, n int, u Vector, c chan int) {
    for ; i < n; i++ {
      v[i] += u.Op(v[i]) // 每个item的计算都是独立的
    }
    c <- 1 // signal that this piece is done
  }
  ```

  并行化：

  ```text
  const NCPU = 4 // number of CPU cores

  func (v Vector) DoAll(u Vector) {
    c := make(chan int, NCPU) // Buffering optional but sensible.
    for i := 0; i < NCPU; i++ {
      go v.DoSome(i*len(v)/NCPU, (i+1)*len(v)/NCPU, u, c)
    }

    // Drain the channel.
    for i := 0; i < NCPU; i++ {
      <-c // wait for one task to complete
    }
    // All done.
  }
  ```

* 将channel当作缓冲区使用。为避免分配和释放缓冲区， 使用一个带缓冲channel表示一个空闲链表。

  ```text
  var freeList = make(chan *Buffer, 100)
  var serverChan = make(chan *Buffer)

  func client() {
    for {
      var b *Buffer
      // Grab a buffer if available; allocate if not.
      select {
      case b = <-freeList:
        // Got one; nothing more to do.
      default:
        // None free, so allocate a new one.
        b = new(Buffer)
      }
      load(b) // Read next message from the net.
      serverChan <- b // Send to server.
    }
  }

  func server() {
    for {
      b := <-serverChan // Wait for work.
      process(b)
      // Reuse buffer if there's room.
      select {
      case freeList <- b:
        // 将缓冲区返回给空闲列表 nothing more to do.
      default:
       // Free list full, just carry on.
      }
    }
  }
  ```

* 当 `panic`被调用后（包括不明确的运行时错误，例如切片检索越界或类型断言失败）， 程序将立刻终止当前函数的执行，并开始回溯 Go 协程的栈，运行任何`defer`函数。 调用 `recover`将停止回溯过程，并返回传入 `panic`的实参。 由于在回溯时只有`defer`函数中的代码在运行，因此 `recover`只能在`defer`函数中才有效。

* `recover`还可以终止失败的 Go 协程而无需杀死其它正在执行的 Go 协程。

  ```text
  func server(workChan <-chan *Work) {
    for work := range workChan {
      go safelyDo(work)
    }
  }

  func safelyDo(work *Work) {
    defer func() {
      if err := recover(); err != nil {
        log.Println("work failed:", err)
      }
    }()
    do(work) // 即使触发了panic，该协程会被干净利落地结束，不会干扰到其它Go协程
  }
  ```

* 在`defer`中可以修改函数的返回值

  ```text
  // Error is the type of a parse error; it satisfies the error interface.
  type Error string
  func (e Error) Error() string {
    return string(e)
  }

  // error is a method of *Regexp that reports parsing errors by panicking with an Error.
  func (regexp *Regexp) error(err string) {
    panic(Error(err))
  }

  // Compile returns a parsed representation of the regular expression.
  func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
      if e := recover(); e != nil {
        regexp = nil // Clear return value.
        // 如果不是parse error，就会触发新的panic，继续栈的回溯（因为是其他类型的错误）
        // 虽然会在产生实际错误时改变Panic的值，不过在clash report中可以展示出原始和新的错误，因此问题根源还是可见的
        err = e.(Error) 
      }
    }()
    return regexp.doParse(str), nil
  }
  ```



