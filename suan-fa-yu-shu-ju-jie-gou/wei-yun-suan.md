# 位运算

## 类别

| 运算符 | 含义 |  |
| :--- | :--- | :--- |
| \` | \` | 或 |
| `&` | 与 |  |
| `~` | 取反 |  |
| `^` | 异或 |  |
| `<<` | 左移 |  |
| `>>` | 右移 |  |

> 在golang中，`^`既可以表示异或，也可以表示取反。
>
> 但作为二元运算符时，就是异或；作为一元运算符时，是取反。

## 作用

* **判断奇偶**：`x & 1`
* **判断是否为2的幂次**：`x & (x - 1) == 0`
* **清除最低位的1**：`x & (x - 1)`
* **得到最低位的1**：`x & -x`
* **将最右边n位清零**：`x & (~0 << n)` 相当于对`11……1`左移n位，因此最低n位都是0
* **获取第n位**：`(x >> n) & 1`
* **获取第n位的幂值**：`x & (1 << (n - 1))`
* **将第n位 置为0**：`x & (~(1 << n))`
* **将第n位 置为1**：`x | (1 << n)`
* **将最高位 至 第n位（含）清零**：`x & ((1 << n)- 1)`
* **将第n位 至 第0位（含） 清零**：`x & ~(((1 << (n+1)) - 1)`
