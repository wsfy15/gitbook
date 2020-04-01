# 命令模式

**将一个请求封装为一个对象，从而使你可用不同的请求对客户进行参数化；对请求排队或记录请求日志，以及支持可撤销的操作。**

以烧烤摊和烧烤店为例：

* 烧烤摊：客户直接与烤肉者交互，**行为请求者 与 行为实现者 的紧耦合**。
* 烧烤店：客户通过服务员下单，服务员告知烤肉者执行烧烤。可以实现对请求的排队或日志记录，以及撤销操作

## 结构图

![1585488172035](../../../.gitbook/assets/1585488172035.png)

## 实现

以一个具有两个按钮的机器为例，两个按钮分别提供启动和重启的功能。

首先定义`Command`接口，封装了实际要执行的操作：

```text
type Command interface {
    execute()
}
```

然后实现`Invoker`，即要求执行命令的类：

```text
type Machine struct {
    buttion1 Command
    buttion2 Command
}

func NewMachine(buttion1, buttion2 Command) *Machine {
    return &Box{
        buttion1: buttion1,
        buttion2: buttion2,
    }
}

func (m *Machine) PressButtion1() {
    b.buttion1.Execute()
}

func (m *Machine) PressButtion2() {
    m.buttion2.Execute()
}
```

实现启动和重启两个命令：

```text
type StartCommand struct {
    mb *MotherBoard
}

func NewStartCommand(mb *MotherBoard) *StartCommand {
    return &StartCommand{mb: mb}
}

func (c *StartCommand) Execute() {
    c.mb.Start()
}

type RebootCommand struct {
    mb *MotherBoard
}

func NewRebootCommand(mb *MotherBoard) *RebootCommand {
    return &RebootCommand{mb: mb}
}

func (c *RebootCommand) Execute() {
    c.mb.Reboot()
}
```

实现`Receiver`类，知道如何实施与执行一个与请求相关的操作，在这里，即知道如何启动和重启。**任何类都可能作为一个`Receiver`。**

```text
type MotherBoard struct{}

func (*MotherBoard) Start() {
    fmt.Print("system starting\n")
}

func (*MotherBoard) Reboot() {
    fmt.Print("system rebooting\n")
}
```

客户端调用：

```text
mb := &MotherBoard{}
startCommand := NewStartCommand(mb)
rebootCommand := NewRebootCommand(mb)
m := NewMachine(startCommand, rebootCommand)
m.PressButtion1()
m.PressButtion2()
```

## 好处

* 能较容易地设计一个命令队列（将Invoker的command设计为一个切片，依次记录每个命令）
* 在需要的情况下，可以较容易地将命令写入日志
* 允许接收请求的一方决定是否要否决请求
* 可以容易地实现对请求的撤销和重做
* 由于加入新的具体命令类不影响其他类，所以增加新的具体命令类很容易
* 把请求一个操作的对象 与 知道如何执行该操作的对象 分割开

**只有在真正需要如 撤销 / 恢复操作等功能时，把原来的代码重构为命令模式才有意义。**

