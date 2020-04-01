# 迭代器模式

**提供一种方法顺序访问一个聚合对象中各个元素，而又不暴露该对象的内部表示。**

例如部分高级语言里的`foreach`、`for in`、`for range`等语句。

## 结构图

![1585489934959](../../../.gitbook/assets/1585489934959.png)

## 实现

定义聚集接口：

```text
type Aggregate interface {
    Iterator() Iterator
}
```

定义迭代器接口：

```text
type Iterator interface {
    First()
    Next() interface{}
    IsDone() bool
}
```

模拟一个聚集及其迭代器：

```text
type Numbers struct {
    start, end int
}

func NewNumbers(start, end int) *Numbers {
    return &Numbers{
        start: start,
        end:   end,
    }
}

func(n *Numbers) Iterator() Iterator {
    return &NumbersIterator{
        numbers: n,
        next:    n.start,
    }
}

type NumbersIterator struct {
    numbers *Numbers
    next    int
}

func (i *NumbersIterator) First() {
    i.next = i.numbers.start
}

func (i *NumbersIterator) IsDone() bool {
    return i.next > i.numbers.end
}

func (i *NumbersIterator) Next() interface{} {
    if !i.IsDone() {
        next := i.next
        i.next++
        return next
    }
    return nil
}
```

客户端调用：

```text
var nums Aggregate = NewNumbers(1, 10)
iterator := nums.Iterator()
for !iterator.IsDone() {
    next := iterator.Next()
    num := next.(int)
    fmt.Println(num)
}
```

同样可以实现从后向前遍历的迭代器。

迭代器模式就是**分离了集合对象的遍历行为**，抽象出一个迭代器类来负责，这样既可以不暴露集合的内部结构，又可让外部代码透明地访问集合内部的数据。

