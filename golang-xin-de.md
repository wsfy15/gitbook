# golang心得

### interface

* 实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。

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





