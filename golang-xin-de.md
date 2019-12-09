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









