# 开放-封闭原则

**软件实体（类、模块、函数等等）应该可以扩展，但不可修改。**

这样的设计才可以在面对需求改变时，保持相对稳定。

在最初编写代码时，假设变化不会发生。当变化发生时，就创建抽象来隔离以后发生的同类变化。

**面对需求，对程序的改动是提高增加新代码进行的，而不是更改现有的代码。**

在开放工作展开不久就知道可能发生的变化，查明可能发生的变化所等待的时间越长，要创建正确的抽象就越困难。
