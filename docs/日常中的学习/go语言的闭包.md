Go 语言支持匿名函数，可作为闭包。匿名函数是一个"内联"语句或表达式。匿名函数的优越性在于可以直接使用函数内的变量，不必申明。

以下实例中，我们创建了函数 getSequence() ，返回另外一个函数。该函数的目的是在闭包中递增 i 变量，代码如下：

```go
package main

import "fmt"

func getSequence() func() int {
   i:=0
   return func() int {
      i+=1
     return i  
   }
}

func main(){
   /* nextNumber 为一个函数，函数 i 为 0 */
   nextNumber := getSequence()  

   /* 调用 nextNumber 函数，i 变量自增 1 并返回 */
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
   fmt.Println(nextNumber())
   
   /* 创建新的函数 nextNumber1，并查看结果 */
   nextNumber1 := getSequence()  
   fmt.Println(nextNumber1())
   fmt.Println(nextNumber1())
}

```

以上代码执行结果为:

```
1
2
3
1
2
```

**闭包是由函数和与其相关的引用环境组合而成的实体。**

在说明闭包之前，先来了解一下什么是**函数变量**。

在 Go 语言中，函数被看作是**第一类值**，这意味着函数像变量一样，有类型、有值，其他普通变量能做的事它也可以。

```
func square(x int) {
    println(x * x)
}
```

1. 直接调用：`square(1)`
2. 把函数当成变量一样赋值：`s := square`；接着可以调用这个函数变量：`s(1)`。
   **注意：这里 `square` 后面没有圆括号，调用才有。**

- 调用 `nil` 的函数变量会导致 panic。
- 函数变量的零值是 `nil`，这意味着它可以跟 `nil` 比较，但两个函数变量之间不能比较。

## 闭包

现在开始通过例子来说明闭包：

```go
func incr() func() int {
    var x int
    return func() int {
        x++
        return x
    }
}
```

调用这个函数会返回一个函数变量。

`i := incr()`：通过把这个函数变量赋值给 `i`，`i` 就成为了一个**闭包**。

所以 `i` 保存着对 `x` 的引用，可以想象 **i 中有着一个指针指向 x** 或 **i 中有 x 的地址**。

由于 `i` 有着指向 `x` 的指针，所以可以修改 `x`，且保持着状态：

```go
println(i()) // 1
println(i()) // 2
println(i()) // 3
```

也就是说，`x` 逃逸了，它的生命周期没有随着它的作用域结束而结束。

但是这段代码却不会递增：

```go
println(incr()()) // 1
println(incr()()) // 1
println(incr()()) // 1
```

这是因为这里调用了三次 `incr()`，返回了三个闭包，这三个闭包引用着三个不同的 `x`，它们的状态是各自独立的。

## 闭包引用

现在开始通过例子来说明由闭包引用产生的问题：

```go
x := 1
f := func() {
    println(x)
}
x = 2
x = 3
f() // 3
```

因为闭包对外层词法域变量是**引用**的，所以这段代码会输出 **3**。

可以想象 `f` 中保存着 `x` 的地址，它使用 `x` 时会直接解引用，所以 `x` 的值改变了会导致 `f` 解引用得到的值也会改变。

但是，这段代码却会输出 **1**：

```go
x := 1
func() {
    println(x) // 1
}()//加了括号
x = 2
x = 3
```

把它转换成这样的形式就容易理解了：

```go
x := 1
f := func() {
    println(x)
}
f() // 1
x = 2
x = 3
```

这是因为 `f` 调用时就已经解引用取值了，这之后的修改就与它无关了。

不过如果再次调用 `f` 还是会输出 **3**，这也再一次证明了 `f` 中保存着 `x` 的地址。

可以通过在闭包内外打印所引用变量的地址来证明：

```go
x := 1
func() {
    println(&x) // 0xc0000de790
}()
println(&x) // 0xc0000de790
```

可以看到引用的是同一个地址。

## 循环闭包引用

接下来在三个例子中说明由循环内的闭包引用所产生的问题：

### 第一个例子

```go
for i := 0; i < 3; i++ {
    func() {
        println(i) // 0, 1, 2
    }()
}
```

这段代码相当于：

```go
for i := 0; i < 3; i++ {
    f := func() {
        println(i) // 0, 1, 2
    }
    f()
}
```

每次迭代后都对 `i` 进行了解引用并使用得到的值且不再使用，所以这段代码会正常输出。

### 第二个例子

正常代码：输出 ***0, 1, 2\***：

```go
var dummy [3]int
for i := 0; i < len(dummy); i++ {
    println(i) // 0, 1, 2
}
```

然而这段代码会输出 **3**：

```go
var dummy [3]int
var f func()
for i := 0; i < len(dummy); i++ {
    f = func() {
        println(i)
    }
}
f() // 3
```

前面讲到闭包取引用，所以这段代码应该输出 **i 最后的值** **2** 对吧？

不对。这是因为 `i` 最后的值并不是 **2**。

把循环转换成这样的形式就容易理解了：

```go
var dummy [3]int
var f func()
for i := 0; i < len(dummy); {
    f = func() {
        println(i)
    }
    i++
}
f() // 3
```

`i` 自加到 **3** 才会跳出循环，所以循环结束后 `i` 最后的值为 **3**。

所以用 `for range` 来实现这个例子就不会这样：

```go
var dummy [3]int
var f func()
for i := range dummy {
    f = func() {
        println(i)
    }
}
f() // 2
```

这是因为 `for range` 和 `for` 底层实现上的不同。

### 第三个例子

```go
var funcSlice []func()
for i := 0; i < 3; i++ {
    funcSlice = append(funcSlice, func() {
        println(i)
    })

}
for j := 0; j < 3; j++ {
    funcSlice[j]() // 3, 3, 3
}
```

输出序列为 **3, 3, 3**。

看了前面的例子之后这里就容易理解了：
这三个函数引用的都是同一个变量（`i`）的地址，所以之后 `i` 递增，解引用得到的值也会递增，所以这三个函数都会输出 **3**。

添加输出地址的代码可以证明：

```go
var funcSlice []func()
for i := 0; i < 3; i++ {
    println(&i) // 0xc0000ac1d0 0xc0000ac1d0 0xc0000ac1d0
    funcSlice = append(funcSlice, func() {
        println(&i)
    })

}
for j := 0; j < 3; j++ {
    funcSlice[j]() // 0xc0000ac1d0 0xc0000ac1d0 0xc0000ac1d0
}
```

可以看到三个函数引用的都是 `i` 的地址。

### 解决方法

#### 1. 声明新变量：

- 声明新变量：`j := i`，且把之后对 `i` 的操作改为对 `j` 操作。
- 声明新同名变量：`i := i`。**注意：这里短声明右边是外层作用域的 `i`，左边是新声明的作用域在这一层的 `i`**。原理同上。

这相当于为这三个函数各声明一个变量，一共三个，这三个变量初始值分别对应循环中的 `i` 并且之后不会再改变。

#### 2. 声明新匿名函数并传参：

```go
var funcSlice []func()
for i := 0; i < 3; i++ {
    func(i int) {
        funcSlice = append(funcSlice, func() {
            println(i)
        })
    }(i)

}
for j := 0; j < 3; j++ {
    funcSlice[j]() // 0, 1, 2
}
```

现在 `println(i)` 使用的 `i` 是通过函数参数传递进来的，并且 Go 语言的函数参数是按值传递的。

```go
	var funcSlice []func()
	for i := 0; i < 3; i++ {
		func(value int) {
			funcSlice = append(funcSlice, func() {
				println(value)
			})
		}(i)

	}
	for j := 0; j < 3; j++ {
		funcSlice[j]() // 0, 1, 2
	}
```

就像这样

所以相当于在这个新的匿名函数内声明了三个变量，被三个闭包函数独立引用。原理跟第一种方法是一样的。

这里的解决方法可以用在大多数跟闭包引用有关的问题上，不局限于第三个例子。

## 参考链接

[Go 语言圣经 - 匿名函数](https://link.segmentfault.com/?enc=3xfY%2B9iMmVKygVsFOSWN5Q%3D%3D.%2BdADFhgGqDP6L%2BUv%2FwaQQCbpxrn%2FiRpk6bSh4oDNwZ9StuvXH3K%2FSmfCLJHyKZpf)

https://segmentfault.com/a/1190000022798222

https://www.runoob.com/go/go-function-closures.html