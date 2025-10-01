# 字符串和基本数据类型之间转换

1. strconv 包转换错误处理

   由于将字符串转为其他数据类型可能会出错，*strconv* 包定义了两个 *error* 类型的变量：*ErrRange* 和 *ErrSyntax*。其中，*ErrRange* 表示值超过了类型能表示的最大范围，比如将 "128" 转为 int8 就会返回这个错误；*ErrSyntax* 表示语法错误，比如将 "" 转为 int 类型会返回这个错误。

   然而，在返回错误的时候，不是直接将上面的变量值返回，而是通过构造一个 *NumError* 类型的 *error* 对象返回。*NumError* 结构的定义如下：

   ```go
   type NumError struct {
       Func string // the failing function (ParseBool, ParseInt, ParseUint, ParseFloat)
       Num  string // the input
       Err  error  // the reason the conversion failed (ErrRange, ErrSyntax)
   }
   
   // 内部实现
   func (e *NumError) Error() string {
       return "strconv." + e.Func + ": " + "parsing " + Quote(e.Num) + ": " + e.Err.Error()
   }
   
   func syntaxError(fn, str string) *NumError {
       return &NumError{fn, str, ErrSyntax}
   }
   
   func rangeError(fn, str string) *NumError {
       return &NumError{fn, str, ErrRange}
   }
   ```

2. 字符串和整数之间转换

   ```go
   // 转为有符号整型
   func ParseInt(s string, base int, bitSize int) (i int64, err error)
   // 转换无符号整型
   func ParseUint(s string, base int, bitSize int) (n uint64, err error)
   
   // atoi内部调用ParseInt(s, 10, 0)
   func Atoi(s string) (i int, err error)
   
   func FormatUint(i uint64, base int) string    // 无符号整型转字符串
   func FormatInt(i int64, base int) string    // 有符号整型转字符串
   func Itoa(i int) string
   ```

3. 字符串和布尔值之间转换

   ```go
   // 接受 1, t, T, TRUE, true, True, 0, f, F, FALSE, false, False 等字符串；
   // 其他形式的字符串会返回错误
   func ParseBool(str string) (value bool, err error)
   // 直接返回 "true" 或 "false"
   func FormatBool(b bool) string
   // 将 "true" 或 "false" append 到 dst 中
   // 这里用了一个 append 函数对于字符串的特殊形式：append(dst, "true"...)
   func AppendBool(dst []byte, b bool)
   ```

4. 字符串和浮点数之间转换

   ```go
   func ParseFloat(s string, bitSize int) (f float64, err error)
   func FormatFloat(f float64, fmt byte, prec, bitSize int) string
   func AppendFloat(dst []byte, f float64, fmt byte, prec int, bitSize int)
   ```

   

