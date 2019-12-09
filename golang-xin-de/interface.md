# interface

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

## iface VS eface

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

## 类型转换、类型断言

不支持隐式类型转换，即直接`a = b`的方式。

### 类型转换

> &lt;结果类型&gt; := &lt;目标类型&gt; \( &lt;表达式&gt; \)

```text
var i int = 9
var f float64

f = float64(i)
a := int(f)

s := []int(i) // cannot convert i (type int) to type []int
```

### 类型断言

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

## 接口转换的原理

当判定一种类型是否满足某个接口时，Go 使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型实现了该接口。

例如某类型有 `m` 个方法，某接口有 `n` 个方法，则很容易知道这种判定的时间复杂度为 `O(mn)`，Go 会对方法集的函数按照函数名的字典序进行排序，所以实际的时间复杂度为 `O(m+n)`。

1. 具体类型转空接口时，\_type 字段直接复制源类型的 \_type；调用 mallocgc 获得一块新内存，把值复制进去，data 再指向这块新内存。
2. 具体类型转非空接口时，入参 tab 是编译器在编译阶段预先生成好的，新接口 tab 字段直接指向入参 tab 指向的 itab；调用 mallocgc 获得一块新内存，把值复制进去，data 再指向这块新内存。
3. 而对于接口转接口，itab 调用 getitab 函数获取。只用生成一次，之后直接从 hash 表中获取。

