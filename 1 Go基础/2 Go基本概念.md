# 1 变量

Go语言是静态类型，变量声明时必须指定变量的类型，以及变量一定要使用。Go 语言与其他语言显著不同的一个地方在于，Go 语言的类型在变量后面。

```go
var a int // 没有赋值默认为0
var a int = 0 // 声明时赋值，指定类型
var a = 1 // 声明时赋值，推断类型
a := 1 // 自动推断类型
```

> go语言区分大小写，不能以关键字作为标识符。

## 1.1 常见的类型

在Go语言中，数据类型被分为四大类型：基础类型、复合类型、引用类型和接口类型。

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

切片使用数组作为底层结构。切片包含三个组件：容量，长度和指向底层数组的指针,切片可以随时进行扩展

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

map式一种存储键值对的数据结构。

```go
m1 := make(map[string]int) // 声明 key是string类型，value是int类型
m2 := map[string]string{     // 声明并且初始化
    "name": "jack",
    "sex": "female",
}
```

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

类型转换

在Go语⾔中，可以把⼀个类型的值转换成另⼀个类型的值，这个转换过程称作类型转换。在Go语⾔类型转换中，没有隐式类型转换，所有的转换都必须显式声明

```go
v ：= T(v)
```

自定义类型

Go通过type关键字自定义类型

```go
type newType oldType
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

![image-20250927182408534](D:\chongtao\github\chongtao\Go_Sumly\1 Go基础\assets\image-20250927182408534-1758968656018-1.png)

在Go中，函数是第一类对象，可以将函数保持到变量中。函数主要有具名和匿名之分，

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

Go语言中方法时关联到类型的，在编译截断完成方法的静态绑定。方法是由函数演变而来，只是将函数的第一个对象参数移动到函数名前而已。

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
        name : "Tom",
        age : 48,
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

Go语言提供sync和channel两种方式支持协程的并发。

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

# 6 单元测试

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

# 7 包和模块

## 7.1 Package

一个文件夹可以作为package，同一个package内部变量、类型、方法等定义可见

Go 语言也有 Public 和 Private 的概念，粒度是包。如果类型/接口/方法/函数/字段的首字母大写，则是 Public 的，对其他 package 可见，如果首字母小写，则是 Private 的，对其他 package 不可见。



## 7.2 Modules





Golang map是怎么遍历的

在Golang中，遍历map主要使用`for`循环配合`range`关键字。`range`会返回两个值：键（key）和值（value）。以下是一个基本的示例：

```go
myMap := map[string]int{
    "apple":  1,
    "banana": 2,
    "orange": 3,
}

for key, value := range myMap {
    fmt.Println(key, value)
}
```

这段代码会输出：

```go
apple 1
banana 2
orange 3
```

需要注意的是，Go语言中的map并不保证遍历的顺序，每次遍历的结果可能不同。如果需要按照特定的顺序遍历map，可以先将map的键（或键值对）放入一个切片，然后对切片进行排序，最后再进行遍历。



# map数据结构

Map是一种类似于键值对集合的数据结构，但与对象相比，它提供了更大的灵活性和功能。以下是Map数据结构的主要特点和功能：

1. **键的多样性**：Map允许使用多种类型的键，包括数字、字符串、布尔值、null、undefined、数组、对象、Set、甚至另一个Map。这与对象通常只使用字符串作为键形成了鲜明对比。
2. **添加、获取和删除元素**：

- `set()`方法用于向Map实例中添加新的成员。如果新成员的键已存在，那么新成员将覆盖原有的键对应的值。
- `get()`方法用于通过键来访问其对应的值。如果键不存在，则返回undefined。
- `delete()`方法用于删除指定的键值对。
- `has()`方法用于通过键来判断成员是否存在，返回一个布尔值。

1. **遍历成员**：Map结构原生提供三个遍历器生成函数和一个遍历方法。可以使用`Map.prototype.keys()`获取键名的遍历器，`Map.prototype.values()`获取键值的遍历器，`Map.prototype.entries()`获取所有成员的遍历器。此外，还可以使用`forEach()`方法遍历Map实例的成员，该方法接收一个回调函数和一个可选的this指向参数。
2. **获取成员个数**：Map实例的`size`属性可以获取其成员的个数。

由于Map的这些特点，它在需要快速查找、插入和删除键值对的情况下非常有用，特别是在键的类型不局限于字符串时。这使得Map成为处理复杂数据结构和实现高效算法的重要工具。

Golang sync.map的用法和原理

`sync.Map` 是 Go 语言中提供的一个并发安全的映射（map）类型，它允许在多个 goroutines 之间进行安全的读写操作，而无需额外的同步机制，如互斥锁（mutex）。`sync.Map` 的设计初衷是为了在并发场景下提供比传统的 `map` 加上互斥锁更高效的读写操作。

### 使用方法

下面是 `sync.Map` 的基本使用方法：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var sm sync.Map

	// 存储键值对
	sm.Store("key1", "value1")

	// 读取键值对
	value, ok := sm.Load("key1")
	// 读取键值对
	value, ok := sm.Load("key1")
	if ok {
		fmt.Println("Loaded value:", value)
	} else {
		fmt.Println("Key not found")
	}

	// 读取或存储键值对，如果键不存在则存储它
	_, loaded := sm.LoadOrStore("key2", "value2")
	if loaded {
		fmt.Println("Key already exists")
	} else {
		fmt.Println("Key-value pair stored")
	}

	// 删除键值对
	sm.Delete("key1")

	// 遍历所有的键值对
	sm.Range(func(key, value interface{}) bool {
		fmt.Printf("Key: %v, Value: %v\n", key, value)
		return true // 返回 true 继续遍历，返回 false 停止遍历
	})
}
```

`sync.Map` 的内部实现相对于传统的 `map` 来说更为复杂，它旨在减少锁争用（lock contention）并优化读写性能。其设计考虑到了读多写少的场景，其中读操作通常是无锁的，而写操作则通过精细的锁粒度来减少锁争用。

以下是 `sync.Map` 的主要组件及其作用：

1. **read**：每个 `sync.Map` 实例包含一个 `read` 结构体，用于存储最近读取的键值对和指向相关桶（bucket）的指针。这有助于优化无锁读操作。
2. **dirty**：一个标准的 `map`，用于存储新写入的键值对以及由于并发修改而需要更新的键值对。当 `dirty` 不为空时，它包含了 `sync.Map` 的最新状态。
3. **misses**：一个计数器，记录了在 `read` 中未找到键的次数。当 `misses` 达到一定阈值时，会将 `dirty` 的内容提升（promote）到 `read` 中，并清空 `dirty`，以减少未来读操作的开销。
4. **mu**：一个互斥锁，保护 `dirty` 的读写以及 `read` 和 `dirty` 之间的提升操作。

在 `sync.Map` 的实现中，读操作通常不需要获取锁，它会先尝试从 `read` 中读取键值对。如果 `read` 中没有所需的键值对，或者 `read` 和 `dirty` 之间的状态不一致（由于并发写操作），则会尝试获取 `mu` 锁并从 `dirty` 中读取或更新键值对。写操作则需要获取 `mu` 锁来修改 `dirty`。

这种设计使得 `sync.Map` 在高并发读、低并发写的场景中表现良好，但在写操作非常频繁的情况下，性能可能不如使用单个互斥锁保护的 `map`。因此，在选择使用 `sync.Map` 还是传统的 `map` 加锁时，需要根据实际的使用场景进行权衡。

# golang并发机制和csp并发模型

Golang的并发机制主要基于goroutine和channel，这是一种轻量级的线程实现方式，与操作系统的线程相比，goroutine的创建和销毁成本更低，可以创建成千上万个而不会给系统带来太大负担。

goroutine是Go语言中并发的执行单位，可以理解为“线程”。它们由Go运行时（runtime）管理，并自动在可用的逻辑处理器上进行调度。这使得开发者可以轻松地编写并发代码，而无需关心底层的线程管理细节。

而channel则是Go语言中各个并发结构体（goroutine）之间的通信机制，可以类比为Linux中的管道。通过channel，goroutine之间可以安全地传递数据，从而实现协同工作。这种通信方式避免了直接使用共享内存可能带来的数据竞争和同步问题。

CSP（Communicating Sequential Processes）并发模型是Go语言推荐使用的并发模型。它强调通过发送消息的方式来同步信息，而不是直接共享内存。在CSP模型中，每个goroutine都可以看作是一个顺序进程，它们通过channel进行通信和协作。这种模型有助于简化并发编程的复杂性，提高代码的可读性和可维护性。

总之，Golang通过goroutine和channel实现了一种高效且易用的并发机制，而CSP并发模型则为这种机制提供了理论支持和指导。这使得Golang成为处理大量并发任务的理想选择。

# slice的底层数据结构和特性

在Go语言中，切片（slice）的底层数据结构主要包括三个关键属性：指针、长度和容量。

1. **指针**：指向底层数组的第一个元素。这个指针使得切片能够引用数组的一段连续区域，从而实现动态数组的功能。
2. **长度**：表示当前切片中元素的数量。这是切片的一个重要属性，它决定了切片当前包含多少个元素。
3. **容量**：从切片的开始位置到底层数组的结尾位置的元素数量。容量总是大于等于长度，它表示切片在不重新分配底层数组的情况下可以容纳的元素数量。

切片的特性主要有以下几点：

1. **动态长度**：与数组不同，切片的长度是动态的，可以在运行时增长或缩小。这使得切片非常适合处理不确定大小的数据集合。
2. **引用类型**：切片是引用类型，它们不存储任何数据，只描述底层数组的一段。因此，对切片的修改会影响到其底层数组。这种特性使得切片在函数传参和共享内存方面非常高效。
3. **自动扩容**：当向切片添加元素时，如果切片的当前容量不足以容纳新元素，Go语言会自动进行扩容操作。这通常涉及到创建一个新的底层数组，并将原切片的数据复制到新数组中。扩容操作确保了切片能够灵活地适应数据的变化。

总的来说，切片的底层数据结构使得它能够在Go语言中实现动态数组的功能，并且具有高效、灵活和易于使用的特性。

# 讲一下golang协程池, 作用是什么