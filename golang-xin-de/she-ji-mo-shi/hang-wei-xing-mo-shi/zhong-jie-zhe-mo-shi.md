# 中介者模式

**用一个中介对象来封装一系列的对象交互。中介使各对象不需要显式地相互引用，从而使其耦合松散，而且可以独立地改变它们之间的交互。**

> **迪米特法则**
>
> 如果两个类不必彼此直接通信，那么这两个类就不应当发生直接的相互作用。
>
> 如果其中一个类需要调用另外一个类的某一个方法，可以通过第三者转发这个调用。

## 结构图

![1585483883874](../../../.gitbook/assets/1585483883874.png)

## 实现

以同事间的交流为例，通过中介者传递消息，就不需要指定具体的交流对象了，即不需要显式的相互引用，减少了耦合。

先定义抽象中介者接口：

```text
type Mediator interface {
    Send(msg, name string)
    Register(string, Colleague)
}
```

再定义抽象同事接口：

```text
type Colleague interface {
    Sendmsg(message, colleagueName string)
    SetMediator(Mediator)
    Receivemsg(message string)
}
```

实现具体的中介者：

```text
type ConcreteMediator struct {
    maps map[string]Colleague
}
func(m *ConcreteMediator) Register(name string, c Colleague) {
    m.maps[name] = c
}
func(m *ConcreteMediator) Send(msg, name string) {
    m.maps[name].Receivemsg(msg)
}
```

实现具体的同事：

```text
type ConcreteColleague struct {
    mediator Mediator
    name string
}

func (c *ConcreteColleague) Sendmsg(message, colleagueName string) {
    c.mediator.Send(message, colleagueName)
}
func (c *ConcreteColleague)    SetMediator(m Mediator) {
    c.mediator = m
    m.Register(c.name, c)
}
func (c *ConcreteColleague)    Receivemsg(message string) {...}
```

客户端调用：

```text
var (
    m Mediator = &ConcreteMediator{maps: make(map[string]Colleague)}
     c1 Colleague = &ConcreteColleagueA{name: "c1"}
     c2 Colleague = &ConcreteColleagueB{name: "c2"}
 )
 c1.SetMediator(m)
 c2.SetMediator(m)

 c1.Sendmsg("hello", "c2")
```

## 优点

* 减少了各个`Colleague`的耦合，使得可以独立地改变和复用各个`Colleague`类和`Mediator`
* 由于把对象如何协作进行了抽象，将中介作为一个独立的概念并将其封装在一个对象中，这样关注的对象就从对象各自本身的行为转移到它们之间的交互上来，也就是站在一个更宏观的角度去看待系统

## 缺点

* 由于`mediator`控制了集中化，就把交互复杂性变为了中介者的复杂性，这就使得中介者变得比任何一个`colleague`都复杂。**一旦中介者出现问题，将影响全局。**

中介者模式一般应用于一组对象以定义良好但是复杂的方式进行通信的场合，以及想定制一个分布在多个类中的行为，而又不想生成太多的子类的场合。

