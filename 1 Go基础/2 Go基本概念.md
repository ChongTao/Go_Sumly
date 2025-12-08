# 1 基本结构

Go语言是静态类型，变量声明时必须指定变量的类型，以及变量一定要使用。Go 语言与其他语言显著不同的一个地方在于，Go 语言的类型在变量后面。

```go
var a int // 没有赋值默认为0
var a int = 0 // 声明时赋值，指定类型
var a = 1 // 声明时赋值，推断类型
a := 1 // 自动推断类型
```

> go语言区分大小写，不能以关键字作为标识符。

## 1.1 常见的类型

在Go语言中，数据类型主要分为两大类：

1. **基本数据类型**：
   - **布尔类型（bool）**：true、false。
   - **数字类型**
     - **整型**：int、int8、int16、int32、int64等
     - **浮点型**：float32、float64
     - **复数类型**：complex64、complex128
   - **字符串类型**：string(不可变)。
2. **派生类型**：
   - **指针类型**：引用类型，用于存储变量的内存地址。
   - **数组类型**：定长数据类型，存储固定数量的同类型元素。
   - **切片类型**：变长的数组类型，是对数组的抽象，提供了更灵活的长度和容量的操作。
   - **字典类型**：无序的键值对类型，用于存储键值对集合。
   - **通道类型**：引用类型，用于goroutine之间的通信。
   - **结构体类型**：由一组字段组成的复合数据类型，字段可以是任何数据类型。
   - **接口类型：引用类型，定义了一组方法的集合，可以由任何实现了这些方法的类型实现。
   - **函数类型**：表示函数签名的类型，可以用来定义函数变量或作为其他函数的参数和返回值。

| 类型          | 长度(字节) | 默认值 | 说明                                      |
| ------------- | ---------- | ------ | ----------------------------------------- |
| bool          | 1          | false  |                                           |
| byte          | 1          | 0      | uint8                                     |
| rune          | 4          | 0      | Unicode Code Point, int32                 |
| int, uint     | 4或8       | 0      | 32 或 64 位                               |
| int8, uint8   | 1          | 0      | -128 ~ 127, 0 ~ 255，byte是uint8 的别名   |
| int16, uint16 | 2          | 0      | -32768 ~ 32767, 0 ~ 65535                 |
| int32, uint32 | 4          | 0      | -21亿~ 21亿, 0 ~ 42亿，rune是int32 的别名 |
| int64, uint64 | 8          | 0      |                                           |
| float32       | 4          | 0.0    |                                           |
| float64       | 8          | 0.0    |                                           |
| complex64     | 8          |        |                                           |
| complex128    | 16         |        |                                           |
| uintptr       | 4或8       |        | 以存储指针的 uint32 或 uint64 整数        |
| array         |            |        | 值类型                                    |
| struct        |            |        | 值类型                                    |
| string        |            | ""     | UTF-8 字符串                              |
| slice         |            | nil    | 引用类型                                  |
| map           |            | nil    | 引用类型                                  |
| channel       |            | nil    | 引用类型                                  |
| interface     |            | nil    | 接口                                      |
| function      |            | nil    | 函数                                      |

## 1.2 字符串

在 Go 语言中，字符串被当做是字节的定长数组，字符串使用 UTF8 编码。

```go
str := "golang"
```

Go语言字符串的底层结构在`reflect.StringHeader`中定义：

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
```

字符串结构由两部分信息组成：第一个是字符串指向的底层字节数组，第二个是字符串的字节的长度。字符串其实是一个结构体，因此字符串的赋值操作也就是`reflect.StringHeader`结构体的复制过程，并不会涉及底层字节数组的复制。

## 1.3 数组

数组是一个固定长度的特定类型元素组成的序列，由零个或多个元素组成。数组的长度是数组类型的组成部分。和数组对应的类型就是切片，切片是可以动态增长和缩减的序列。

```go
var arr [5]int // 声明数组

var arr1 = [5]int{1,2,3,4,5} // 声明并初始化
arr1 := [5]int{1,2,3,4,5} // 声明并初始化
```

数组的遍历

```go
var arr = [5]int{1,2,3,4,5}
for i := 0 i < len(arr); i++ {
    fmt.Println(arr[i])
}

for i, v := range arr {
   fmt.Printf("b[%d]: %d\n", i, v)
}

for i := 0; i < len(arr); i++ {
   fmt.Printf("c[%d]: %d\n", i, c[i])
}
```

> 用for range方式迭代的性能更好，保证捕获出现数组越界的情况。

## 1.4 切片

切片使用数组作为底层结构。切片包含三个组件：容量，长度和指向底层数组的指针，切片可以随时进行扩展。

```go
slice := make([]int, 0) // 长度为0的切片
slice1 := make([]int, 3, 5) // 长度为3容量为5的切片

// 切片添加元素
slice1 = append(slice1, 1, 2, 3)
fmt.Println(len(slice1), cap(slice1))

// 子切片
sub := slice1[3:]
```

> 声明切片时可以指定容量大小，预分配空间。如果容量不够，切片自动扩容。

切片的结构定义`reflect.SliceHeader`:

```go
type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
}
```

可以看出切片结构体定义与字符串定义多了个Cap成员，其表示指向内存空间的最大容量。

## 1.5 map

Map是一种类似于键值对集合的数据结构，但与对象相比，它提供了更大的灵活性和功能。以下是Map数据结构的主要特点和功能：

1. **键的多样性**：Map允许使用多种类型的键，包括数字、字符串、布尔值、null、undefined、数组、对象、Set、甚至另一个Map。这与对象通常只使用字符串作为键形成了鲜明对比。
2. **添加、获取和删除元素**：

- `set()`方法用于向Map实例中添加新的成员。如果新成员的键已存在，那么新成员将覆盖原有的键对应的值。
- `get()`方法用于通过键来访问其对应的值。如果键不存在，则返回undefined。
- `delete()`方法用于删除指定的键值对。
- `has()`方法用于通过键来判断成员是否存在，返回一个布尔值。

示例：

```go
m1 := make(map[string]int) // 声明 key是string类型，value是int类型
m2 := map[string]string{     // 声明并且初始化
    "name": "jack",
    "sex": "female",
}
```

在Golang中，遍历map主要使用`for`循环配合`range`关键字。`range`会返回两个值：键（key）和值（value）。以下是一个基本的示例：

```go
for key, value := range m2 {
    fmt.Println(key, value)
}
```

> 需要注意的是，Go语言中的map并不保证遍历的顺序，每次遍历的结果可能不同。如果需要按照特定的顺序遍历map，可以先将map的键（或键值对）放入一个切片，然后对切片进行排序，最后再进行遍历。



## 1.6 指针

指针即某个值的地址，类型定义时使用符号*，对已经存在变量使用&获取该变量的地址。

```go
str := "hello"
var p *string = &str
*p = "World"
```

指针通常在函数传递参数，或者给某个类型定义新的方法时使用。在Go语言中，参数是按值传递的，如果不使用指针，函数内部将会拷贝一份参数的副本，对参数的修改并不会影响到外部变量的值。如果参数使用指针，对参数的传递将会影响到外部变量。

## 1.7 枚举值

通常使用常量来表示枚举值。

```go
tpye StuType int32

const (
	Type1 StuType = iota
    Type2
    Type3
)

func main() {
    fmt.Println(Type1, Type2, Type3)
}
```

## 1.8 类型转换

在 **Go 语言（Golang）** 中，**类型转换（Type Conversion）** 是指将一个变量从一种数据类型转换为另一种类型的过程。Go 是强类型语言，不会进行隐式类型转换，**所有类型转换必须显式声明**。

```go
v := T(v)
```

示例：

```go
var a int = 10
var b float64 = float64(a)   // int → float64
var c int = int(b)           // float64 → int（小数部分会被截断）

// 使用 类型断言（Type Assertion）：
var i interface{} = "hello"

s, ok := i.(string)
if ok {
    fmt.Println("String:", s)
}
```

# 2 流程控制

## 2.1 条件语句

```go
age := 18
if age < 18 {
    fmt.Println("kid")
} else {
    fmt.Println("Adult")
}
```

## 2.2 switch

```go
type Gender int8

const {
    MALE Gender = 1
    FEMALE Gender = 2
}

gender := MALE

switch gender {
    case FEMALE:
    	fmt.Println("female")
    case MALE:
    	fmt.Println("male")
    default:
	    fmt.Println("unknown")
}
```

## 2.3 for循环

```go
sum := 0
for i := 0; i < 10; i++ {
    if sum > 10 {
        break
    }
    sum += i
}
```

# 3 函数

Go语言中的函数有具名和匿名之分：具名函数一般对应于包级的函数，是匿名函数的一种特例，当匿名函数引用了外部作用域中的变量时就是闭包函数，闭包函数是函数式编程语言的核心。方法时绑定到一个具体结构的特殊函数，Go语言中的方法时依托于类型的，必须在编译时静态绑定。

Go语言的程序初始化和执行从`main.main`函数开始的，如果`main`包导入了其它的包，会按照顺序将它们包含进`main`包中，当一个包被导入时，如果它还导入了其它的包，则先将其它的包包含进来，然后创建和初始化这个包的常量和变量，再调用包里的`init`函数，如果一个包中有多个`init`函数时，调用顺序未定义，同一个文件内的多个`init`则是以出现的顺序依次调用。最后当`main`包的所有包级常量、变量被创建和初始化完成，并且`init`函数被执行后，才会进入`main.main`函数。

## 3.1 函数的定义 

```go
// 具名函数
func Add(a, b int) int {
    return a + b
}

// 匿名函数
var Add = func(a, b int) int {
    return a + b
}
```

## 3.2 参数与返回值

Go语言中的函数可以有多个参数和多个返回值，参数和返回值都是以传值的方式。

```go
func funcName(param1 Type1, param2 Type2, ...)(return1 Type3,...) {
    
}
```

示例

```go
func add(num1 int, num2 int) int {
	return num1 + num2
}

func main() {
	ans := add(1, 3)
	fmt.Println(ans)
}
```

当可变参数是一个空接口类型时，调用者是否解包可变参数会导致不同的结果：

```go
func main() {
    var a = []interface{}{1, "abc"}
    
    Print(a...)
    Print(a)
}

func Print(a ...interface{}) {
    fmt.Println(a...)
}
```

第一个 `Print` 调用时传入的参数是 `a...`，等价于直接调用 `Print(1, "abc")`。第二个 `Print` 调用传入的是未解包的 `a`，等价于直接调用 `Print([]interface{}{1, "abc"})`。

函数的返回值可以命名：

```go
func proces(m map[int]int, key int) (value int, ok bool) {
    value, ok = m[key]
    return
}
```

也可以通过`defer`语句在`return`语句之后修改返回值:

```go
func Pro() (v int) {
    defer func() { v++ } ()
    return 42
}
```

其中`defer`语句延迟执行一个匿名函数，该匿名函数捕获外部函数的局部变量v, 这种被称为闭包。闭包对外部变量不是传值方式访问，而是以引用的方式访问。

> 在 Go 语言中，`defer` 用来延迟函数调用，直到外层函数返回之前才执行这些被延迟的语句。

## 3.3 错误处理

在函数中出现不能处理的错误，返回给调用者处理，使用error记录错误信息。

```go
func main() {
    _,err := os.Open("filename.txt")
    if err != nil {
        fmt.Println(err)
    }
}
```

通过errors.New返回自定义错误

```go
func hello(name string) error {
	if name == "" {
		return errors.New("error: name is empty")
	}

	fmt.Println("Hello " + name)
	return nil
}

func main() {
	if err := hello(""); err == nil {
		fmt.Println("error")
	}
}
```

error是能够预知的错误，但是也可能出现不可预知的错误，例如数组越界，空指针等，这种错误可能导致程序非正常退出，在Go语言中称为panic。

```go
func get(index int) int {
	arr := [3]int{2, 3, 4}
	return arr[index]
}

func main() {
	fmt.Println(get(5))
	fmt.Println("finished")
}
```

Go 语言使用defer和recover进行捕获和恢复。

```go
func process(index int) (ret int) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println("arr index out")
			ret = -1
		}
	}()

	nums := [3]int{1, 2, 3}
	return nums[index]
}

func main() {
	fmt.Println(process(42))
}
```

> go中无异常类型，只有错误类型。

# 4 结构体和接口

## 4.1 结构体和方法

Go语言中方法是关联到类型的，在编译截断完成方法的静态绑定。方法是由函数演变而来，只是将函数的第一个对象参数移动到函数名前而已。

go的结构体类似Java的class，在结构体中可以定义多个字段，方法等。

```go
type Student struct {
	name string
	age  int
}

func (stu *Student) hello(person string) string {
	return fmt.Sprintf("hello %s, I am %s", person, stu.name)
}

func main() {
	stu := &Student{
		name: "Jack",
	}

	msg := stu.hello("Rose")
	fmt.Println(msg)
}
```

此外也可以使用new实例化：

```go
stu := new(Student)
```

## 4.2 接口

在go语言中，接口定义一组方法的集合，接口不能被实例化，一个类型实现所有的方法，接口对应的方法是在运行时动态绑定。

```go
type Person interface {
	getName() string
}

type Teacher struct {
	name string
	age  int
}

func (tea *Teacher) getName() string {
	return tea.name
}

type Worker struct {
	name string
	age  int
}

func (work *Worker) getName() string {
	return work.name
}

func main() {
	var p Person = &Teacher{
		name: "tom",
		age:  28,
	}

	fmt.Println(p.getName())
}
```

- Go语言中，不需要显示声明实现哪一个接口，只需要直接实现该接口的所有的方法。

实例可以强制类型转换为接口，接口也可以强制类型转换成实例。

```go
func main() {
    var p Person = &Worker {
        name: "Tom",
        age: 48,
    }
    
    worker := p.(*Worker) // 接口转换为实例
    fmt.Println(worker.getAge)
}
```

### 4.2.1 空接口

空接口是没有任何方法的接口，可以用该接口表示任意类型。

```go
func main() {
	m := make(map[string]interface{})

	m["name"] = "Go"
	m["age"] = 18
	fmt.Println(m)
}
```

# 5 并发

## 5.1 sync

Go 的 **`sync`** 包是标准库中用于**并发控制和同步**的核心组件之一，提供了一组原语（primitives）来保证多协程（goroutine）之间的安全访问和协作。它解决了并发中的常见问题，如**共享资源竞争、等待任务完成、数据一致性**等。

| 类型             | 作用                                         | 常用方法                                     |
| ---------------- | -------------------------------------------- | -------------------------------------------- |
| `sync.Mutex`     | 互斥锁，保证同一时刻只有一个协程访问共享资源 | `Lock()`, `Unlock()`                         |
| `sync.RWMutex`   | 读写锁，允许并发读但写时独占                 | `RLock()`, `RUnlock()`, `Lock()`, `Unlock()` |
| `sync.WaitGroup` | 等待一组协程完成任务                         | `Add()`, `Done()`, `Wait()`                  |
| `sync.Once`      | 确保某个操作只执行一次（常用于单例）         | `Do(func)`                                   |
| `sync.Cond`      | 条件变量，用于协程之间的事件通知             | `Wait()`, `Signal()`, `Broadcast()`          |
| `sync.Map`       | 并发安全的 map，替代传统 map + 锁            | `Store()`, `Load()`, `Delete()`, `Range()`   |
| `sync.Pool`      | 对象池，用于重复使用临时对象减少 GC 压力     | `Get()`, `Put()`                             |

示例：

```go
var wg sync.WaitGroup

func download(url string) {
	fmt.Println("start download...")
	time.Sleep(time.Second * 3)
	wg.Done()
}

func main() {
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go download("url" + string(rune(i+'0')))
	}
	wg.Wait()
	fmt.Println("finish download")
}
```

- wg.Add(1)：为 wg 添加一个计数，wg.Done()，减去一个计数。
- go download()：启动新的协程并发执行 download 函数。
- wg.Wait()：等待所有的协程执行结束。

## 5.2 channel

在 Go 语言中，**`channel`（通道）** 是 goroutine 之间**通信与同步的核心机制**，用于在不同协程之间安全地传递数据（更多信息见[Channel](./5 Channel.md)）。

- goroutine 通过 channel **发送数据（send）** 或 **接收数据（receive）**。

- channel 本身会负责同步 —— 当一个 goroutine 发送时，会阻塞直到另一个 goroutine 接收。

类型分类：

| 类型           | 定义                                        | 特点                           |
| -------------- | ------------------------------------------- | ------------------------------ |
| **无缓冲通道** | `make(chan int)`                            | 发送和接收必须同步进行（阻塞） |
| **有缓冲通道** | `make(chan int, N)`                         | 可存放 N 个元素，满/空时阻塞   |
| **单向通道**   | `chan<- int`（只写） / `<-chan int`（只读） | 用于限制函数参数的使用方向     |

示例：

```go
var ch = make(chan string, 10)

func download(url string) {
	fmt.Println("start to download", url)
	time.Sleep(time.Second * 3)
	ch <- url // 将URL发送信道
}

func main() {
	for i := 0; i < 3; i++ {
		go download("a.com" + string(rune(i+'0')))
	}

	for i := 0; i < 3; i++ {
		msg := <-ch
		fmt.Println("finish", msg)
	}
	fmt.Println("Done!")
}
```

使用channel信道，可以在协程之间传递消息。

> 对于无缓冲的 channel，发送方将阻塞该信道，直到接收方从该信道接收到数据为止，而接收方也将阻塞该信道，直到发送方将数据发送到该信道中为止。对于有缓存的 channel，发送方在没有空插槽（缓冲区使用完）的情况下阻塞，而接收方在信道为空的情况下阻塞。

# 6 单元测试

在 Go 中，**单元测试（Unit Test）** 是语言级别内置支持的功能， 由标准库 **`testing`** 包提供，用于自动化验证代码逻辑是否正确。

假设我们希望测试 package main 下 `calc.go` 中的函数，要只需要新建 `calc_test.go` 文件，在`calc_test.go`中新建测试用例即可。

```go
package main

func add(num1 int, num2 int) int {
	return num1 + num2
}
```

```go
// calc_test.go
package main

import "testing"

func TestAdd(t *testing.T) {
	if ans := add(1, 2); ans != 3 {
		t.Error("add(1, 2) should be equal to 3")
	}
}
```

## 6.1 TestMain

在 Go 中，`TestMain` 是一个特殊命名的函数：

```go
func TestMain(m *testing.M)
```

如果某个测试包中定义了这个函数，当你执行 `go test` 时，**Go 不会直接运行 `TestXxx` 测试函数**，而是：

1. 自动检测并调用 `TestMain`；
2. 由 `TestMain` 调用 `m.Run()` 来执行所有测试；
3. 最终由你自己决定如何退出（通常通过 `os.Exit(code)`）。

```go
func TestMain(m *testing.M) {
    // 1️⃣ 全局初始化
    fmt.Println(">>> 初始化测试环境")
    setup()

    // 2️⃣ 运行所有测试
    code := m.Run()

    // 3️⃣ 全局清理
    fmt.Println(">>> 清理测试环境")
    teardown()

    // 4️⃣ 手动退出并返回状态码
    os.Exit(code)
}
```

> 单独执行某个测试用例，TestMain也会执行。

# 7 包和模块

## 7.1 Package

代码组织的**基本单元**，由同一目录下的 Go 文件组成，共同形成一个命名空间。一个文件夹可以作为package，同一个package内部变量、类型、方法等定义可见

Go 语言也有 Public 和 Private 的概念，粒度是包。如果类型/接口/方法/函数/字段的首字母大写，则是 Public 的，对其他 package 可见，如果首字母小写，则是 Private 的，对其他 package 不可见。

## 7.2 Modules

代码分发与依赖管理的**最小单元**，由 `go.mod` 文件定义。

```html
myapp/
 ├── go.mod
 ├── main.go
 └── mathutil/
      └── add.go
```

`go.mod`：

```yaml
module example.com/myapp

go 1.23
```