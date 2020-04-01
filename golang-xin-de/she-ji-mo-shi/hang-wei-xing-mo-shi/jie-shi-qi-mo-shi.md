# 解释器模式

**给定一个语言，定义它的文法的一种表示，并定义一个解释器，这个解释器使用该表示来解释语言中的句子。**

如果一种特定类型的问题发生的频率足够高，那么可能就值得将该问题的各个实例表述为一个简单语言中的句子。这样就可以构建一个解释器，该解释器提供解释这些句子来解决该问题。例如 正则表达式。

## 结构图

![1585533126285](../../../.gitbook/assets/1585533126285.png)

## 实现

首先定义表达式接口：

```text
type Expression interface {
    Interpret() int
}
```

接下来以算式计算为例，每个具体的数字都是叶子节点，非叶子节点为具体的算符。

定义终结符表达式，实现与文法中的终结符相关联的解释操作。文法中的每一个终结符都有一个具体终结表达式与之相对应。

```text
type ValExpression struct {
    val int
}

func (e *ValExpression) Interpret() int {
    return n.val
}
```

定义非终结符表达式，为文法中的非终结符实现解释操作。对文法中每一条规则都需要一个具体的非终结符表达式类。

```text
type AddExpression struct {
    left, right Expression
}

func (n *AddExpression) Interpret() int {
    return n.left.Interpret() + n.right.Interpret()
}

type MinExpression struct {
    left, right Expression
}

func (n *MinExpression) Interpret() int {
    return n.left.Interpret() - n.right.Interpret()
}
```

Context，包含解释器之外的一些全局信息。

```text
type Context struct {
    exp []string
    index int
    prev Expression
}

func (c *Context) Parse(exp string) {
    c.exp = strings.Split(exp, " ")

    for {
        if c.index >= len(c.exp) {
            return
        }
        switch c.exp[c.index] {
        case "+":
            c.prev = c.newAddExpression()
        case "-":
            c.prev = c.newMinExpression()
        default:
            c.prev = c.newValExpression()
        }
    }
}

func (c *Context) newAddExpression() Node {
    c.index++
    return &AddExpression{
        left:  c.prev,
        right: c.newValExpression(),
    }
}

func (c *Context) newMinExpression() Expression {
    c.index++
    return &MinExpression{
        left:  c.prev,
        right: c.newValExpression(),
    }
}

func (c *Context) newValExpression() Expression {
    v, _ := strconv.Atoi(c.exp[c.index])
    c.index++
    return &ValExpression{
        val: v,
    }
}

func (c *Context) Result() Expression {
    return c.prev
}
```

客户端调用：

```text
c := &Context{}
c.Parse("1 + 2 + 4 - 5")
c.Result().Interpret()
```

**当有一个语言需要解释执行，并且可将该语言中的句子表示为一个抽象语法树时，可使用解释器模式。**

## 好处

* 很容易地改变和扩展文法，因为该模式使用类来表示文法规则，可使用继承来改变或扩展文法。
* 比较容易实现文法，因为定义抽象语法树中各个节点的类的实现大体类似，这些类都易于直接编写

## 不足

* 为文法中的每一条规则至少定义了一个类，因此包含许多规则的文法可能难以管理和维护

