# 代理模式

为其他对象提供一种代理以控制对这个对象的访问。

## 结构图

![1585466335419](../../../.gitbook/assets/1585466335419.png)

`Subject`类，定义了`RealSubject`和`Proxy`的共用接口，这样就可以在任何使用`RealSubject`的地方都可以使用`Proxy`。

## 实现

定义`Subject`接口：

```text
type Subject interface {
    Request()
}
```

实现该接口的真实实体：

```text
type RealSubject struct {...}
func(*RealSubject) Request() {...}
```

实现代理类，通过保存一个引用，使代理可以访问实体，并且提供一个与Subject的接口相同的接口。

```text
type Proxy struct {
    realSubject *RealSubject
}

func(p *Proxy) preRequest() {
    // 做一些初始化、锁、权限控制相关的操作
    if p.realSubject == nil {
        p.realSubject = &RealSubject{}
    }
}
func(p *Proxy) afterRequest() {...}
func(p *Proxy) Request() {
    p.preRequest()
    p.realSubject.Request()
    p.afterRequest()
}
```

客户端调用：

```text
var p Proxy = &Proxy{}
p.Request()
```

## 应用场景

* 远程代理：隐藏一个对象存在于不同地址空间的事实，例如一些WebService
* 虚拟代理：根据需要创建开销很大的对象。通过它存放实例化需要很长时间的真实对象。例如网页中为还未加载完成的图片保留图片框。
* 安全代理：控制真实对象访问时的权限
* 智能指引：当调用真实的对象时，代理处理另外一些事。例如引用计数，当引用数为0时可以释放对象，或当第一次引用一个持久对象时，将它存入内存，或在访问对象前检查锁

