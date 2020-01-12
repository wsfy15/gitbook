# interface

## interface

实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。

```text
package main

import "fmt"

type coder interface {
    code()
    debug()
}

type Gopher struct {
    language string
}

// 会生成接收者是*Gopher类型的code方法
func (p Gopher) code() {
    fmt.Printf("I am coding %s language\n", p.language)
}

// 不会生成接收者是Gopher类型的debug方法
func (p *Gopher) debug() {
    fmt.Printf("I am debuging %s language\n", p.language)
}

func main() {
    var c coder = &Gopher{"Go"}
    // var c coder = Gopher{"Go"} // 会报错，因为没有Gopher类型的debug方法
    c.code()
    c.debug()
}
```

因为这是 **接收者是指针类型的方法，很可能在方法中会对接收者的属性进行更改操作，从而影响接收者**；**而对于接收者是值类型的方法，在方法中不会对接收者本身产生影响**。所以，当实现了一个接收者是值类型的方法，就可以自动生成一个接收者是对应指针类型的方法，因为两者都不会影响接收者。

但是，当实现了一个接收者是指针类型的方法，如果此时自动生成一个接收者是值类型的方法，原本期望对接收者的改变（通过指针实现），现在无法实现，因为值类型会产生一个拷贝，不会真正影响调用者。

语法上 `T` 能直接调 `*T` 的方法仅仅是 `Go` 的语法糖，先取到T的指针再去调用\*T的方法。

### iface VS eface

iface包含方法的接口，eface即 `interface{}` 。

```text
type iface struct {
    tab  *itab    // 接口的类型以及赋给这个接口的实体类型
    data unsafe.Pointer
}

type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

**编译器自动检测类型是否实现接口**

```text
package main

import "io"

type myWriter struct {

}

// 解除注释后，运行程序不报错。
/*func (w myWriter) Write(p []byte) (n int, err error) {
    return
}*/

func main() {
    // 检查 *myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = (*myWriter)(nil)

    // 检查 myWriter 类型是否实现了 io.Writer 接口
    var _ io.Writer = myWriter{}
}
```

上述赋值语句会发生隐式地类型转换，在转换的过程中，编译器会检测等号右边的类型是否实现了等号左边接口所规定的函数。

### 类型转换、类型断言

不支持隐式类型转换，即直接`a = b`的方式。

#### 类型转换

> &lt;结果类型&gt; := &lt;目标类型&gt; \( &lt;表达式&gt; \)

```text
var i int = 9
var f float64

f = float64(i)
a := int(f)

s := []int(i) // cannot convert i (type int) to type []int
```

#### 类型断言

空接口 `interface{}` 没有定义任何函数，因此 Go 中所有类型都实现了空接口。当一个函数的形参是 `interface{}`，那么在函数中，需要对形参进行断言，从而得到它的真实类型。

> &lt;目标类型的值&gt;，&lt;布尔参数&gt; := &lt;表达式&gt;.\( 目标类型 \) // 安全类型断言 &lt;目标类型的值&gt; := &lt;表达式&gt;.\( 目标类型 \) //非安全类型断言

```text
package main

import "fmt"

type Student struct {
    Name string
    Age int
}

func main() {
    var i interface{} = new(Student)
    s, ok := i.(Student)    // i.(*Student)
    if ok {
        fmt.Println(s)
    }
}
```

还可以使用switch判断接口类型

```text
func judge(v interface{}) {
    fmt.Printf("%p %v\n", &v, v)

    switch v := v.(type) {
    case nil:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("nil type[%T] %v\n", v, v)

    case Student:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("Student type[%T] %v\n", v, v)

    case *Student:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("*Student type[%T] %v\n", v, v)

    default:
        fmt.Printf("%p %v\n", &v, v)
        fmt.Printf("unknow\n")
    }
}

type Student struct {
    Name string
    Age int
}
```

`fmt.Println` 函数的参数是 `interface`。

对于内置类型，函数内部会用穷举法，得出它的真实类型，然后转换为字符串打印。

而对于自定义类型，首先确定该类型是否实现了 `String()` 方法（接收者最好为值类型），如果实现了，则直接打印输出 `String()` 方法的结果；否则，会通过反射来遍历对象的成员进行打印。

### 接口转换的原理

当判定一种类型是否满足某个接口时，Go 使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型实现了该接口。

例如某类型有 `m` 个方法，某接口有 `n` 个方法，则很容易知道这种判定的时间复杂度为 `O(mn)`，Go 会对方法集的函数按照函数名的字典序进行排序，所以实际的时间复杂度为 `O(m+n)`。

1. 具体类型转空接口时，\_type 字段直接复制源类型的 \_type；调用 mallocgc 获得一块新内存，把值复制进去，data 再指向这块新内存。
2. 具体类型转非空接口时，入参 tab 是编译器在编译阶段预先生成好的，新接口 tab 字段直接指向入参 tab 指向的 itab；调用 mallocgc 获得一块新内存，把值复制进去，data 再指向这块新内存。
3. 而对于接口转接口，itab 调用 getitab 函数获取。只用生成一次，之后直接从 hash 表中获取。

## reflection

### 接口的值

接口类型的值包含`(value, type)`对。

```text
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty // r 的 (value, type)对为：(tty, *os.File)

var w io.Writer
w = r.(io.Writer)    // w 的 (value, type)对为：(tty, *os.File)
```

接口的静态类型确定可以使用接口变量调用哪些方法，即使内部的具体值可能具有更大的方法集。

### 从接口值到反射对象

最基础的反射就是从一个接口变量得到其`(value, type)`对，reflect包提供了`reflect.TypeOf` 和`reflect.ValueOf`这两个方法。

```text
var x float64 = 3.4
fmt.Println("type:", reflect.TypeOf(x))    // type: float64
fmt.Println("value:", reflect.ValueOf(x)) // value: 5.4
fmt.Println("value:", reflect.ValueOf(x).String()) // value: <float64 Value>
```

`reflect.Type`和`reflect.Value`具有一系列方法。

```text
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())        // type: float64
fmt.Println("kind is float64:", v.Kind() == reflect.Float64) // kind is float64: true
fmt.Println("value:", v.Float())    // value: 3.4
```

也有`SetInt` 和`SetFloat`之类的方法，但使用前需要确认是否可以使用。

* `Kind`方法描述的是反射对象的底层类型，而不是静态类型

  ```text
  type MyInt int
  var x MyInt = 7
  v := reflect.ValueOf(x)
  v.Kind() === reflect.Int    // 虽然x的静态类型是MyInt
  ```

* `getter`和`setter`方法都是在能容纳该值的最大类型上运行。以int64（有符号整数）为例，`reflect.Value`的`Int()`返回int64、`SetInt()`接收一个int64的值（可能会转换为涉及的实际类型）

  ```text
  var x uint8 = 'x'
  v := reflect.ValueOf(x)
  fmt.Println("type:", v.Type())    // uint8.
  fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8)  // true.
  x = uint8(v.Uint())              // v.Uint returns a uint64.
  ```

### 从反射对象到接口值

`reflect.Value` 对象的`Interface` 方法，将type和value信息打包为接口形式的表示并返回，实现从反射对象到接口的转换。

```text
y := v.Interface().(float64) // y will have type float64.
```

fmt包的`Println`、`Printf`等方法以空接口的形式接收变量，然后通过`reflect.Value`得到这些接口的实际值。因此，如果我们要打印反射对象的实际值，只需要传递该对象的`Interface` 方法的结果：

```text
fmt.Println(v.Interface())
fmt.Printf("value is %7.1e\n", v.Interface()) // 知道是个float64值，格式化输出 // 3.4e+00
// 不需要类型断言
```

### 修改反射对象

修改反射对象时，必须使用settable的value。

```text
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
// panic: reflect: reflect.flag.mustBeAssignable using unaddressable value
```

问题不在于值7.1是不可寻址的，而是因为 v不是settable的。

settable是`reflect.Value`的属性，但并不是所有类型的`reflect.Value`都具有该属性。

```text
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet()) // settability of v: false
```

settability是指反射对象可以修改用于创建反射对象的实际对象的存储空间，settability由反射对象是否持有原始item确定。上例中，传入`reflect.ValueOf`方法的是变量x的copy，而不是x本身。

与函数传参的传值、传引用类似，要想修改反射对象，需要传入指针。

```text
var x float64 = 3.4
p := reflect.ValueOf(&x) 
fmt.Println("type of p:", p.Type())    // type of p: *float64
fmt.Println("settability of p:", p.CanSet()) // settability of p: false
```

虽然指针的反射对象不可以修改，我们想修改的也不是指针，而是指针指向的值。可以通过`reflect.Value`的`Elem` 方法，会将指针指向的内容保存在新的`reflect.Value`变量中。

```text
v := p.Elem()
fmt.Println("settability of v:", v.CanSet()) // settability of v: true

v.SetFloat(7.1)
fmt.Println(v.Interface())    // 7.1
fmt.Println(x)    // 7.1
```

#### 通过反射修改struct

使用反射修改结构体的字段更常见，只要有了结构的地址，就可以修改其字段。

```text
type T struct {
    A int    // 只有首字母大写的字段才可以被修改（settable）
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}

output:
0: A int = 23
1: B string = skidoo

s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t) // t is now {77 Sunset Strip}
```

