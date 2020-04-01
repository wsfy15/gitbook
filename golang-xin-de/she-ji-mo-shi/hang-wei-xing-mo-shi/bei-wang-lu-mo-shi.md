# 备忘录模式

**在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可将该对象恢复到原先保存的状态。**

## 结构图

![1585545211945](../../../.gitbook/assets/1585545211945.png)

## 实现

先定义备忘录接口，用于存储其他对象的内部状态：

```text
type Momento interface {}

type MomentoA struct {
    state string
}
```

定义使用备忘录的Originator，负责创建一个备忘录Memento，用以记录当前时刻它的内部状态，并可使用备忘录恢复内部状态。Originator可根据需要决定Memento存储Originator的哪些内部状态。

```text
type Originator struct {
    state string 
}

func(o *Originator) CreateMomento() Momento {
    return &MomentoA{state: o.state}
}

func(o *Originator) RestoreFromMomento(m Momento) {
    o.state = m.state
}
```

备忘录由Caretaker负责保存，Caretaker（管理者）不能对备忘录的内容进行操作或检查。

```text
type Caretaker struct {
    m Momento // 可以考虑用切片存储更多的状态
}

func(c *Caretaker) Set(m Momento) { c.m = m }
func(c *Caretaker) Get() Momento { return c.m }
```

客户端调用：

```text
o := &Originator{"start"}
c := &Caretaker{}
c.Set(o.CreateMomento())

o.state = "end"

o.RestoreFromMomento(c.Get())
```

备忘录模式适用于**功能比较复杂的，但需要维护或记录属性历史**的类，或者需要保存的属性只是众多属性中的一小部分时，Originator可以根据保存的Memento信息还原到前一状态。

