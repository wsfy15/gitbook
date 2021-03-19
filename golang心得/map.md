# map

## 使用

### 初始化

初始化map时，尽可能用`make`，因为可以指定初始化容量，尽量指定 map 容量。

但如果map包含固定的元素列表，则使用 map literals\(map 初始化列表\) 初始化映射。

```text
m := map[T1]T2{
  k1: v1,
  k2: v2,
  k3: v3,
}
// 而不是make方式
m := make(map[T1]T2, 3)
m[k1] = v1
m[k2] = v2
m[k3] = v3
```



### 赋值

```
  m := make(map[string]int)
  m["foo"]++
  fmt.Println(m["foo"]) // 1
  
  // 相当于下面的代码
  
   m := make(map[string]int)
   v := m["foo"]
   v++
   m["foo"] = v
   fmt.Println(m["foo"])
```



### 修改

当value类型是值类型的结构体时（不是指针），不能直接用`m[k].xx = xx`的方式进行修改，这是因为`struct`与`int/float`同属**值类型**，值类型最重要的特点是在进行赋值时，新变量会得到一份拷贝后的值。 使用`for range` 遍历 `[]struct` `map[xx]struct` 时，得到的也是一份拷贝，因此下面的代码过不了编译，因为修改的是原有的拷贝，即使编译器允许，`map`中的`struct`值也不会改变，所以编译器直接禁止这种情况。

```text
m := map[int]student{
   1: {name: "1"},
}
m[1].name = "2" // cannot assign to struct field m[1].name in map
```

有以下两种解决方法：

- 使用指针：`map[int]*student`

- 使用临时变量：

  ```text
  m := map[int]student{1: {name: "1"}}
  tmp := m[1]
  tmp.name = "2"
  m[1] = tmp
  ```



## key的类型

`map`的键可以是任何**支持相等性操作符的类型（comparable）**，具体包括布尔值、数字、字符串、指针、通道、接口类型、结构体、只包含上述类型的数组，但切片、map、函数不行，因为它们的相等性未定义。

如果是结构体，只有 hash 后的值相等以及字面值相等，才被认为是相同的 key。即使字面值相等，hash出来的值不一定相等，比如引用。



### math.NaN 作为key

```
m := make(map[float64]int)
m[1.4] = 1
m[2.4] = 2
m[math.NaN()] = 3
m[math.NaN()] = 3
for k, v := range m {
	fmt.Printf("[%v, %d] ", k, v) // [2.4, 2] [NaN, 3] [NaN, 3] [1.4, 1] 
}
fmt.Printf("\nk: %v, v: %d\n", math.NaN(), m[math.NaN()]) // k: NaN, v: 0
fmt.Printf("k: %v, v: %d\n", 2.400000000001, m[2.400000000001]) // k: 2.400000000001, v: 0
fmt.Printf("k: %v, v: %d\n", 2.4000000000000000000000001, m[2.4000000000000000000000001]) // k: 2.4, v: 2
fmt.Println(math.NaN() == math.NaN()) // false
```

这是因为，在将`float64`类型的值插入map的key之前，会调用`Float64bits`将其以IEEE 754标准存储在一个64位的`uint64`变量中，然后再调用`Float64frombits`将前者转为`float64`类型，因此会带来精度缺失。因此 2.4 和 2.4000000000000000000000001 转换后的值一样。

```
fmt.Println(math.Float64bits(2.4)) 						// 4612586738352862003
fmt.Println(math.Float64bits(2.400000000001))			 // 4612586738352864255
fmt.Println(math.Float64bits(2.4000000000000000000000001))// 4612586738352862003
```

> ```
> // Float64bits returns the IEEE 754 binary representation of f,
> // with the sign bit of f and the result in the same bit position,
> // and Float64bits(Float64frombits(x)) == x.
> func Float64bits(f float64) uint64 { return *(*uint64)(unsafe.Pointer(&f)) }
> 
> // Float64frombits returns the floating-point number corresponding
> // to the IEEE 754 binary representation b, with the sign bit of b
> // and the result in the same bit position.
> // Float64frombits(Float64bits(x)) == x.
> func Float64frombits(b uint64) float64 { return *(*float64)(unsafe.Pointer(&b)) }
> ```



对于`math.NaN()`，其函数如下：

```
uvnan    = 0x7FF8000000000001

// NaN returns an IEEE 754 ``not-a-number'' value.
func NaN() float64 { return Float64frombits(uvnan) }

func f64hash(p unsafe.Pointer, h uintptr) uintptr {
    f := *(*float64)(p)
    switch {
    case f == 0:
        return c1 * (c0 ^ h) // +0, -0
    case f != f:
        return c1 * (c0 ^ h ^ uintptr(fastrand())) // any kind of NaN
    default:
        return memhash(p, h, 8)
    }
}
```

由于 `NAN != NAN`，在求`NaN`的哈希时，会进入第二个case，导致加一个随机数，导致`hash(NAN) != hash(NAN)`。



## 结构

`map`结构体定义在`hmap`中，go的`map`是通过哈希表实现的，`hmap`中又通过`bmap`存储键值对，一个`bmap`就是一个bucket，bucket的容量是8。通过key的哈希值定位存放在哪个bucket中，有两种方法可以用于获取bucket编号：

1. 取模法：$hash \% m$，m是桶的数量
2. 与运算法：$hash \& (m - 1)$，m取值需要是2的幂次才能保证`[0, m - 1]`区间内的每个桶都有可能被选择

```
type hmap struct {
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

type mapextra struct {
	overflow    *[]*bmap  // 记录所有已经被使用的溢出桶的位置
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	// 哈希值的高八位
	// If tophash[0] < minTopHash, tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

虽然`bmap`结构体定义里只有`tophash`一个字段，但实际上在其后面，还会存放八个key和八个value，这样子存放内存会更紧凑，最后还会有一个`bmap`指针`overflow`指向溢出桶。

> `map`会预分配$2^{B - 4}$个溢出桶，即当$B > 4$，桶数量大于16个时。

溢出桶与常规桶在内存中是连续的，只是前$2^B$个用于常规桶，后面的用作溢出桶。



## 非线程安全

在查找、赋值、遍历、删除的过程中都会检测写标志，一旦发现写标志置位（等于1），则直接 panic。赋值和删除函数在检测完写标志是复位之后，先将写标志位置位，才会进行之后的操作。

检测写标志：

```
if h.flags&hashWriting != 0 {
	throw("concurrent map writes")
}
```

设置写标志：

```
h.flags ^= hashWriting
```

在赋值、删除、清空完成后，也会检查写标志是否为1，如果不为1，说明其他地方也在同时写map，并且完成了写入：

```
if h.flags&hashWriting == 0 {
	throw("concurrent map writes")
}
```

恢复写标志：

```
h.flags &^= hashWriting
```



## 扩容

### 触发时机

- $loadFactor = count / m, m = 2^B$，当负载因子大于阈值时：翻倍扩容
- 负载因子不大，但溢出桶数量大于等于$min(常规桶数量, 2^{15})$：这种场景对应有大量键值对被删除的场景，采用等量扩容

采用渐进式扩容，每次访问map时迁移几个桶。



### 无序遍历

map 在扩容后，会发生 key 的搬迁，原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了（bucket 序号加上了 `2^B`）。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。

在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。



### 元素取址

不能对map的key或者value取址，即使通过`unsafe.Pointer`等获取到了 key 或 value 的地址，也不能长期持有，因为一旦发生扩容，key 和 value 的位置就会改变，之前保存的地址也就失效了。



### map指针不支持索引

```
type Param map[string]interface{}

 type Show struct {
     *Param
 }

 func main() {
     s := new(Show)
     p := make(Param)
     s.Param = &p
     s.Param["day"] = 2 // invalid operation: s.Param["day"] (type *Param does not support indexing)
}
```

访问内嵌指针类型的元素的正确方法：

```
p := make(Param)
p["day"] = 2
s.Param = &p
tmp := *s.Param
fmt.Println(tmp["day"])
```



