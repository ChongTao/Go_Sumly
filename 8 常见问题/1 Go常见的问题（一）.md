# defer原理

在 Go 语言中，`defer` 用来延迟函数调用，直到外层函数返回之前才执行这些被延迟的语句。最典型的用途就是**资源释放**，比如关闭文件、解锁互斥锁等。

Go 编译器会在编译阶段将 `defer` 转化成一种特殊的调用结构，运行时维护一个**栈**（`defer stack`）。

**在执行到 defer 声明处时**，Go 会把这个延迟调用的参数 **立刻求值**，并将调用信息压入当前 goroutine 的 `defer 栈`。当函数要返回时（不管是正常 `return`，还是因为 panic 中途退出），运行时会从栈顶依次弹出 defer 调用并执行。如果执行 defer 的过程中发生 panic，运行时会继续执行后续的 defer，直到全部执行完毕。

# defer执行顺序

- 多个 `defer` 的执行顺序为 **后进先出（LIFO）**。

- `defer` 的参数在 **注册时求值**；但若使用闭包，会在**执行时读取变量的当前值**。

- 函数返回流程为： **计算返回值 → 执行所有 defer → 函数最终退出**。

- 使用命名返回值时，`defer` 可以修改返回值。

- 在 `panic` 展开过程中，已注册的 `defer` 仍会正常执行。

# defer和return的顺序

1. 首先执行`return`的赋值操作，即将返回值放到相应的位置。
2. 然后执行`defer`语句中的代码。
3. 最后执行`return`的返回值操作，即函数携带当前返回值退出。

这种顺序保证了`defer`语句可以在函数返回之前执行必要的操作，同时也确保了返回值的正确性。需要注意的是，如果`defer`语句中修改了有名返回值函数的返回值，那么最终返回的值将是`defer`中修改后的值。但对于无名返回值函数，`defer`无法获取到临时变量的地址，因此无法修改最终返回值。

# 循环内部执行defer会发生什么

**循环内部的 defer 会在每次迭代时注册一个新的 defer，所有 defer 都会在函数退出时逆序执行。**

- `defer` **不是在循环结束时执行**
- `defer` **是在“整个函数返回时”逆序执行**
- 每一次循环都会把一个新的`defer`入栈
- 循环次数大 → defer 堆积 → **可能导致内存泄漏或性能问题**

# defer的底层数据结构和特性

在 Go 中，`defer` 是由 **编译器 + 运行时（runtime）** 合作实现的：

- 编译器在遇到 `defer` 时，会生成专门的结构体记录延迟调用信息。
- 运行时在函数退出之前，按 **后进先出（LIFO）** 顺序执行这些延迟调用。

Go为每一个`defer`语句生成一个`_defer`结构体实例。这个结构体包含了多个字段，如函数栈指针（sp）、程序计数器（pc）、函数地址（fn）以及一个指向下一个`_defer`结构体的指针（link）。这个指针用于链接多个`defer`，形成一个单链表。每个goroutine数据结构中都有一个指向`defer`链表的指针。当声明一个新的`defer`时，它会被插入到链表的头部；当函数返回前执行`defer`语句时，会从链表的头部依次取出并执行。

# 字符串打印时，%v和%+v的区别

`%v` 和 `%+v` 都可以用来打印 struct 的值，区别在于 `%v` 仅打印各个字段的值，`%+v` 还会打印各个字段的名称。

```go
type Stu struct {
	Name string
}

func main() {
	fmt.Printf("%v\n", Stu{"Tom"}) // {Tom}
	fmt.Printf("%+v\n", Stu{"Tom"}) // {Name:Tom}
}
```

# 2个interface可以比较

Go 语言中，interface 的内部实现包含了 2 个字段，类型 `T` 和 值 `V`，interface 可以使用 `==` 或 `!=` 比较。2 个 interface 相等有以下 2 种情况：

1. 两个 interface 均等于 nil（此时 V 和 T 都处于 unset 状态）
2. 类型 T 相同，且对应的值 V 相等。

```go
type Stu struct {
	Name string
}

type StuInt interface{}

func main() {
	var stu1, stu2 StuInt = &Stu{"Tom"}, &Stu{"Tom"}
	var stu3, stu4 StuInt = Stu{"Tom"}, Stu{"Tom"}
	fmt.Println(stu1 == stu2) // false
	fmt.Println(stu3 == stu4) // true
}
```

`stu1` 和 `stu2` 对应的类型是 `*Stu`，值是 Stu 结构体的地址，两个地址不同，因此结果为 false。
`stu3` 和 `stu4` 对应的类型是 `Stu`，值是 Stu 结构体，且各字段相等，因此结果为 true。

# 两个nil可能不相等吗？

可能，接口(interface) 是对非接口值(例如指针，struct等)的封装，内部实现包含 2 个字段，类型 `T` 和 值 `V`。一个接口等于 nil，当且仅当 T 和 V 处于 unset 状态（T=nil，V is unset）。

- 两个接口值比较时，会先比较 T，再比较 V。
- 接口值与非接口值比较时，会先将非接口值尝试转换为接口值，再比较。

```go
func main() {
	var p *int = nil
	var i interface{} = p
	fmt.Println(i == p) // true
	fmt.Println(p == nil) // true
	fmt.Println(i == nil) // false
}
```

# go中make和new的区别

- **`new`**：只分配内存，返回指针，不做初始化（除了置零值）。
- **`make`**：分配并初始化内置引用类型（slice、map、chan），返回的是 **初始化后的值**，而不是指针。

| 特性                       | `new`                                        | `make`                            |
| -------------------------- | -------------------------------------------- | --------------------------------- |
| **作用**                   | 分配指定类型的零值内存，返回指向该类型的指针 | 创建并初始化指定的内部引用类型    |
| **返回类型**               | `*T` （某个类型的指针）                      | T（slice / map / chan 类型本身）  |
| **能否初始化**             | ❌ 不能（只零值化）                           | ✅ 能指定长度等初始化参数          |
| **适用类型**               | 任何类型（struct、数组、基本类型等）         | 仅限 **slice**、**map**、**chan** |
| **是否需要后续手动初始化** | 是（除了零值外的初始化需手动进行）           | 否（返回的就是可用的值）          |

# 如何在panic中恢复

- **`panic`**：让当前 goroutine 立刻中断正常执行，并沿调用栈向上传播，直到程序崩溃。
- **`recover`**：只能在 **`defer` 延迟函数** 内调用，用来捕获 `panic`，使程序恢复正常执行，避免崩溃。

在 Go 中，**发生 `panic` 时的执行顺序**：

1. 触发 `panic`，正常代码停止执行。
2. 开始逐层执行调用栈上的 `defer`。
3. 如果某个 `defer` 中调用了 `recover()` 并返回了非 `nil` 值，`panic` 被**捕获**并中止传播，程序恢复到调用 `panic` 的地方继续执行 **剩余的 defer**，再返回外层。
4. 如果没有 `recover`，最终程序崩溃退出。

# recover的执行时机

**`recover` 只有在 `defer` 延迟函数执行期间调用才生效**，并且它的生效时机是在 **发生 `panic` 后、程序开始执行 defer 栈、且调用到包含 `recover` 的 defer 时**。



# 判断一个结构是否实现某个接口

只要一个类型的方法集包含接口要求的全部方法，它就实现了该接口。

常见是编译期断言判断

```go
// 值接受
var _ Reader = File{}

// 指针接受
var _ Reader = (*File)(nil)
```
