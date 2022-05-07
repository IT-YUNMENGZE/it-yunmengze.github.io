## Golang知识点

### 变量
1. 短变量声明（:=）不能在函数外使用，函数外的每个语句都必须以关键字开始（var, func 等等）；
2. int, unit 和 uintptr 在32系统上通常为32位宽，在64位系统上则为64位宽；

### 流程控制语句
1. 同 for 一样，if 语句可以在条件表达式前执行一个简单的语句，变量的作用域仅在 if 和对应的 else 块之内；
    ```go
    func pow(x, n, lim float64) float64 {
        if v := math.Pow(x, n); v<lim {
            return v
        }
        return lim
    }
    ```








2. Go 的 switch 语句类似于 C、C++、Java、JavaScript 和 PHP 中的，不过 Go 只运行选定的 case，而非之后所有的 case。 实际上，Go 自动提供了在这些语言中每个 case 后面所需的 `break` 语句。 除非以 `fallthrough` 语句结束，否则分支会自动终止。 Go 的另一点重要的不同在于 switch 的 case 无需为常量，且取值不必为整数；

    ```go
    func main() {
        fmt.Print("Go runs on ")
        switch os := runtime.GOOS; os {
        case "darwin":
            fmt.Println("OS X.")
        case "linux":
            fmt.Println("Linux.")
        default:
            // freebsd, openbsd,
            // plan9, windows...
            fmt.Printf("%s.\n", os)
        }
    }
    ```

3. 无条件的 switch 同 switch true，能简化 if-else 语句；

    ```go
    func main() {
        t := time.Now()
        switch {
        case t.Hour() < 12:
            fmt.Println("Good morning!")
        case t.Hour() < 17:
            fmt.Println("Good afternoon.")
        default:
            fmt.Println("Good evening.")
        }
    }
    ```

4. defer 语句会将函数推迟到外层函数返回后执行。推迟调用的函数其参数会立即求值，但直到外层函数返回前该函数都不会被调用。推迟的函数调用会被压入一个栈中。当外层函数返回时，被推迟的函数会按照后进先出的顺序调用；

    ```go
    func main() {
        fmt.Println("counting")

        for i := 0; i < 10; i++ {
            defer fmt.Println(i)
        }

        fmt.Println("done")
    }
    ```
    Result:

    ```go
    counting
    done
    9
    8
    7
    6
    5
    4
    3
    2
    1
    0
    ```

### 结构体struct

1. 一个结构体（struct）就是一组字段（field）；
2. 结构体中的字段可以通过指针来访问，即```(*p).X```来访问指针p所指向结构体的字段X。简单起见，Go允许我们使用**隐式间接引用**，直接写成```p.X```即可；

### 数组和切片
1. ```[n]T```表示拥有n个T类型的值的数组。数组的长度是其类型的一部分，因此数组不能改变大小；

    ```go
    var a [10]int
    ```

2. 每个数组的大小都是固定的，而切片（slice）则为数组元素提供动态大小的、灵活的视角。类型 `[]T` 表示一个元素类型为 `T` 的切片。

   切片通过两个下标来界定，即一个上界和一个下界，二者以冒号分隔：

   ```
   a[low : high]
   ```

   它会选择一个半开区间，包括第一个元素，**但排除最后一个元素**。

   ```go
   package main
   
   import "fmt"
   
   func main() {
   	primes := [6]int{2, 3, 5, 7, 11, 13}
   
   	var s []int = primes[1:4]
   	fmt.Println(s)
   }
   ```

3. 切片并不存储任何数据，它只是描述了底层数组中的一段。更改切片的元素会修改其底层数组中对应的元素；

4. 切片拥有 **长度** 和 **容量**。切片的长度就是它所包含的元素个数。切片的容量是从它的**第一个元素**开始数，到其**底层数组元素末尾**的个数。切片 `s` 的长度和容量可通过表达式 `len(s)` 和 `cap(s)` 来获取。类似于一个滑动窗口，下限只能往前移动，不能回退，上限可以前后移动；

    ```go
    func main() {
        s := []int{2, 3, 5, 7, 11, 13}
        printSlice(s)

        // 截取切片使其长度为 len=0 cap=6 []
        s = s[:0]
        printSlice(s)

        // 拓展其长度 len=4 cap=6 [2 3 5 7]
        s = s[:4]
        printSlice(s)

        // 舍弃前两个值 len=2 cap=4 [5 7]
        s = s[2:]
        printSlice(s)
    }

    func printSlice(s []int) {
        fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
    }
    ```

5. `make` 函数会分配一个元素为零值的**数组**并返回一个引用了它的**切片**，用来创建动态数组：

    ```go
    a := make([]int, 5)  // len(a)=5 cap(a)=5 [0 0 0 0 0]
    ```
    
    要指定它的容量，需向 `make` 传入第三个参数：

    ```go
    b := make([]int, 0, 5) // len(b)=0, cap(b)=5
    
    b = b[:cap(b)] // len(b)=5, cap(b)=5, [0 0 0 0 0]
    b = b[1:]      // len(b)=4, cap(b)=4
    ```

6. `append`为切片追加新元素，当切片底层数组太小，无法容纳所有给定的值时，它会分配一个更大的数组；

### 包

1. 当标识符（包括常量、变量、类型、函数名、结构体字段等等）以一个大写字母开头，那么这个标识符的对象就是可导出的（类似于面向对象语言中的public）；标识符如果以小写字母开头，是不可导出的，但在整个包的内部是可见并且可用的（类似于面向对象语言中的protected）；

### 方法

1. Go没有类，但可以为结构体类型定义方法。方法就是一个带 **接收者** 参数的**函数**；

    ```go
    type Vertex struct {
        X,Y float64
    }

    func (v Vertex) Abs() float64 {
        return math.Sqrt(v.X*v.X + v.Y*v.Y)
    }
    ```

2. Go中可以为非结构体类型声明方法，但只能为在同一包内定义的类型的接收者声明方法。而不能为其他包内定义的类型（包括 int 之类的内建类型）的接收者声明方法；
3. **值接收者与指针接收者**：指针接收者的方法可以修改接收者指向的值，若使用值接收者，那么方法会对原始值的副本进行操作。由于方法经常需要更改它的接收者，指针接收者比值接收者更常用。

    ```go
    type Vertex struct {
        X,Y float64
    }

    func (v Vertex) Abs() float64 {
        return math.Sqrt(v.X*v.X + v.Y*v.Y)
    }

    func (v *Vertex) Scale(f float64) {
        v.X = v.X * f
        v.Y = v.Y * f
    }
    ```

4. 以指针为接收者的方法被调用时，接收者既能为值又能为指针：

    ```go
    var v Vertex
    v.Scale(5)	//OK
    p := &v
    p.Scale(10)	//OK
    ```

Go会将语句```v.Scale(5)```解释为```（&v）.Scale(5)```。

5. 而以值为接收者的方法被调用时，接收者既能为值又能为指针：

    ```go
    var v Vertex
    fmt.Println(v.Abs())	//OK
    p := &v
    fmt.Println(p.Abs())	//OK
    ```
    
    这种情况下，方法调用```p.Abs()```会被解释为```(*p).Abs()```，注意，此时v的值不会因为接收者是指针而改变，方法是以值为接收者的，所以Abs()方法修改的是v的副本；
    
6. 使用指针接收者的原因有二：1）方法能够修改其接收者指向的值；2）其次，这样可以避免在每次调用方法时复制该值。若值的类型为大型结构体时，这样做会更加高效。

### 接口

1. 接口类型是由一组**方法签名**（注意是方法不是函数）定义的集合，接口类型的变量可以保存任何实现了这些方法的类型值。

    ```go
    type Abser interface{
        Abs() float64
    }

    type MyFloat float64

    func (f MyFloat) Abs() float64 {
        if f < 0 {
            return float64(-f)
        }
        return float64(f)
    }

    type Vertex struct {
        X, Y float64
    }

    func (v *Vertex) Abs() float64 {
        return math.Sqrt(v.X*v.X + v.Y*v.Y)
    }

    func main() {
        var a Abser
        f := MyFloat(-math.Sqrt2)
        v := Vertex{3, 4}
        a = f //MyFloat实现了Abser
        a = &v //*Vertex实现了Abser
        // a = v，由于Abs方法只为*Vertex(指针类型)定义，因此Vertex（值类型）并未实现Abser，所以错误，
    }
    ```

2. 类型通过实现一个接口的**所有**方法来实现该接口，无需专门的显式声明，隐式接口从接口的实现中解耦了定义，这样接口的实现可以出现在任何包中；

3. 指定了零个方法的接口值被称为 **空接口**：

    ```go
    interface{}
    ```

   接口可保存任何类型的值。（因为每个类型都至少实现了零个方法。）

   空接口被用来处理未知类型的值。例如，`fmt.Print` 可接受类型为 `interface{}` 的任意数量的参数。

    ```go
    package main

    import "fmt"

    func main() {
        var i interface{}
        describe(i)

        i = 42
        describe(i)

        i = "hello"
        describe(i)
    }

    func describe(i interface{}) {
        fmt.Printf("(%v, %T)\n", i, i)
    }

    ```

4. 由于空接口可保存任何类型的值，**类型断言** 能提供访问接口指定底层具体值的方式。

   ```
   t := i.(T)
   ```

   该语句断言接口值 `i` 保存了具体类型 `T`，并将其底层类型为 `T` 的值赋予变量 `t`。

   若 `i` 并未保存 `T` 类型的值，该语句就会触发一个恐慌。

   为了 **判断** 一个接口值是否保存了一个特定的类型，类型断言可返回两个值：其底层值以及一个报告断言是否成功的布尔值。

   ```
   t, ok := i.(T)
   ```

5. **类型选择** 是一种按顺序从几个类型断言中选择分支的结构。

   类型选择与 switch 语句相似，不过类型选择中的 case 为类型（而非值）， 它们针对给定接口值所存储的值的类型进行比较。

   ```
   switch v := i.(type) {
   case T:
       // v 的类型为 T
   case S:
       // v 的类型为 S
   default:
       // 没有匹配，v 与 i 的类型相同
   }
   ```

   类型选择中的声明与类型断言 `i.(T)` 的语法相同，只是具体类型 `T` 被替换成了关键字 `type`。

   此选择语句判断接口值 `i` 保存的值类型是 `T` 还是 `S`。在 `T` 或 `S` 的情况下，变量 `v` 会分别按 `T` 或 `S` 类型保存 `i` 拥有的值。在默认（即没有匹配）的情况下，变量 `v` 与 `i` 的接口类型和值相同；

6. [`fmt`](https://go-zh.org/pkg/fmt/) 包中定义的 [`Stringer`](https://go-zh.org/pkg/fmt/#Stringer) 是最普遍的接口之一。

   ```go
   package main
   
   import "fmt"
   
   type Person struct {
   	Name string
   	Age  int
   }
   
   func (p Person) String() string {
   	return fmt.Sprintf("%v (%v years)\n", p.Name, p.Age)
   }
   
   func main() {
   	a := Person{"Arthur Dent", 42}
   	z := Person{"Zaphod Beeblebrox", 9001}
   	fmt.Println(a, z)
   }
   ```

7. Go 程序使用 `error` 值来表示错误状态。

   与 `fmt.Stringer` 类似，`error` 类型是一个内建接口：

   ```
   type error interface {
       Error() string
   }
   ```

   （与 `fmt.Stringer` 类似，`fmt` 包在打印值时也会满足 `error`。）

   通常函数会返回一个 `error` 值，调用的它的代码应当判断这个错误是否等于 `nil` 来进行错误处理。

   ```
   i, err := strconv.Atoi("42")
   if err != nil {
       fmt.Printf("couldn't convert number: %v\n", err)
       return
   }
   fmt.Println("Converted integer:", i)
   ```

   `error` 为 nil 时表示成功；非 nil 的 `error` 表示失败。