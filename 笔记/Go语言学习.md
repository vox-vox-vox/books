# helloWorld

Package： A package is a collection of source files in the same directory that are compiled together. Functions, types, variables, and constants defined in one source file are visible to all other source files within the same package.

Repository/Module：A Go repository typically contains only one module, located at the root of the repository. A file named `go.mod` there declares the module path: the import path prefix for all packages within the module

`go.mod` ：最新版本采用go.mod来配置依赖，环境变量中要开启：`GO111MODULE="on"`。如果没有`go.mod`，将不会被认为是一个module，

project--module--package



goland提供的三种启动方式，分别的原理是什么，runKind

​	-File：以指定的文件构建应用程序，相当于`go run cmd/main.go`

​	-Directory：

​	-Package：在包的层面运行，会主动找main包下的main.go执行，相当于 `go run .`



GOROOT：GOROOT is a variable that defines where your Go SDK is located. You do not need to change this variable, unless you plan to use different Go versions.

GOPATH：GOPATH is a variable that defines the root of your workspace. By default, the workspace directory is a directory that is named `go` within your user home directory (**~/go** for Linux and MacOS). GOPATH stores your code base and all the files that are necessary for your development. You can use another directory as your workspace by [configuring GOPATH for different scopes](https://www.jetbrains.com/help/go/configuring-goroot-and-gopath.html#configuring-gopath). GOPATH is the root of your workspace and contains the following folders:

- **src/**: location of Go source code (for example, **.go**, **.c**, **.g**, **.s**).
- **pkg/**: location of compiled package code (for example, **.a**).
- **bin/**: location of compiled executable programs built by Go.

但是！这些都是之前的概念。之前项目必须放在gopath下的src中，现在项目可以不放在里面了，只要设置了`go.mod`，自动下载依赖包，项目也不必放在GOPATH/src内了。

现在的`GOPATH/src`可以没有，但是最好还是把项目放在里面方便管理

所有的依赖包将会被下载至`GOPATH/pkg`中

`GOPATH/bin`目录中会存放可执行文件，同其他所有的bin目录一样。比如我现在的mac中的`GOPATH/bin`中就放了：![image-20210920203127438](/Users/huxiao/Library/Application Support/typora-user-images/image-20210920203127438.png)

这些都是可执行文件，像脚本一样直接执行就行那种, 原因是我在.local.rc文件中设置了：

![image-20210920204432721](/Users/huxiao/Library/Application Support/typora-user-images/image-20210920204432721.png)

将GOPATH添加到了bin中，这样他也可执行了。



Package 和 goland中的文件名（directory）有什么关系？为什么package不等于directory？

答：The only use of the directory is for collecting a set of files that share the same package name.在 go 中，目录与包名匹配是约定俗成的，但这不一定是这种情况。一个directory下的文件一定要是同一个package，但是directory和package的名字不一定相同



main package 和main 函数







# 切片

数组是有容量的，所以中括号中有容量，切片的动态数组，是没有容量，这是数组和切片最大的区别

```go
array := [20] int {0,1,2,3,4,5,6,7,8,9}// array
slice := [] int {0,1,2,3,4,5,6,7,8,9}// slice,会创建一个和上面相同的数组，然后构建一个引用了它的切片：
```

切片的声明

```go
var s []int
s := []int{}
s := make([]int,0)
```

capacity 表示 slice 能装多少东西, len 表示 slice 当前装了多少东西

Src:source

dst:destination



一个小例子：

```go
	s := []int{2, 3, 5, 7, 11, 13}

	s = s[1:2]
	fmt.Println(s)

	s = s[1:5]
	fmt.Println(s)

	s = s[1:]
	fmt.Println(s)


output: 
[3]
[5 7 11 13]
[7 11 13]
```



## 1. 去掉字符串的倒数37个字符

```go
func main() {
	wfID:="package-lane-match:f018e338-5300-4131-b802-46468cc563e2"
	s := []rune(wfID)
	s = s[:len(s)-37]			//切片截取
	fmt.Println(string(s))

}
```







# map

```go
/* 声明变量，默认 map 是 nil */
var map_variable map[key_data_type]value_data_type
```



```go
func main() {
  m := make(map[string]int)//key string value int
  m["Answer"] = 42
  fmt.Println("The value:", m["Answer"])

  m["Answer"] = 48
  fmt.Println("The value:", m["Answer"])

  delete(m, "Answer")
  fmt.Println("The value:", m["Answer"])

  v, ok := m["Answer"]//通过双赋值检测某个键是否存在,若 key 在 m 中，ok 为 true ；否则，ok 为 false。若 key 不在映射中，那么 elem 是该映射元素类型的零值。
  fmt.Println("The value:", v, "Present?", ok)
}
```

![image-20210911131759418](/Users/huxiao/Library/Application Support/typora-user-images/image-20210911131759418.png)

use Pointer to key & pointer to value. Unsafe pointers = pointers to everything

![image-20210911133400722](/Users/huxiao/Library/Application Support/typora-user-images/image-20210911133400722.png)

编译时：1. 先得到key的pointer 2. 调用lookup得到对应value的pointer 3. 返回value pointer对应的值

![image-20210911132238231](/Users/huxiao/Library/Application Support/typora-user-images/image-20210911132238231.png)

![image-20210911132635955](/Users/huxiao/Library/Application Support/typora-user-images/image-20210911132635955.png)

1. **map 内部的值不能寻址，not addressable**。can I take a address of a value slot? like &m[k] , the answer is no.  if I take the address of some entrys of the bucket, as the map grows bigger, the pointer may point to another bucket。自然，map["key"].name这种操作也是不被允许的，因为它的本质就是修改&map["key"]这个地址下的值

2. **a:=map["key"]并不会真正将map["key"]所代表的东西付给a**（因为指针是不可靠的！）传递的实际上是map["key"]所代表的值的拷贝。所以将a更改后，再次读取map["key"]，读出来的值并不会更改

3. ```go
   type hx struct{
     name string
   }
   a:=HX{
     name:"hx"
   }
   map=make(map[string]HX)
   map["key1"]=a
   a.name="hhhhhhhhhh"
   println(map["key1"])
   ```

4. map本身是引用类型。a:=make(map[string]int)，a是一个指向map的指针。


# 匿名函数

```go
//一般函数
func add(x int, y int) int {
	return x + y
}

//匿名函数就是没有名字的函数，可以作为值传递给变量，也可以直接执行
//1. 
add := func (x int, y int) int {
	return x + y
}

//2. 
func (x int, y int) int {
	return x + y
}()
```



# 闭包

闭包是一个匿名函数值，它引用了其函数体之外的变量。正常函数调用完后内部的变量就会销毁，但闭包却能使本该销毁的变量一直保留。**闭包=匿名函数+函数作用域外的变量**

```go
func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}
```

`adder`返回一个匿名函数，这个函数由于引用了它作用域外的`sum`变量，所以形成一个闭包。

**闭包方法的异常需要用外部来接收**，这时极有可能出现局部error覆盖问题。见Bugs的第4个



# 方法

golang语言中的方法是与对象实例绑定的特殊函数，用于维护和展示对象的自身状态。

与函数的区别是方法有前置实例接收参数(receiver)，编译器根据receiver来判断该方法属于哪个实例。

**方法具体到实例本身，和实例进行绑定，而不是通用的**

```go
//构造Animal结构体，即主人类型
type Animal struct{} 
var an Animal //声明主人
an.Run(1)     //主人调用方法

//Animal类型的主人an，有一个Run方法，
//这里是值接收，也可以使用指针接收
func (an Animal)Run(a int) int {
    fmt.Println("Run ",a)
    return 0
}
```



以指针为接收者的方法被调用时，接收者既能为值又能为指针

而以值为接收者的方法被调用时，接收者既能为值又能为指针



使用指针接收者的原因有二：

首先，方法能够修改其接收者指向的值。

其次，这样可以避免在每次调用方法时复制该值。若值的类型为大型结构体时，这样做会更加高效。





# 接口

和int，string等一样，**接口也是一种类型**，和struct类似；区别是struct中存放**各种属性**，而接口中存放**各种方法**的声明。另外有几点需要注意

- 同一个接口内所有的方法都被实现后，该接口才能被正常使用；例如下方的run和eat方法都需要被实现
- 方法实现多态



Go中的接口采用隐式实现，隐式接口从接口的实现中解耦了定义，这样接口的实现可以出现在任何包中，无需提前准备。

因此，也就无需在每一个实现上增加新的接口名称，这样同时也鼓励了明确的接口定义。

"A实现了B接口" 可以理解为，存在一种属于A的方法实现了B接口中所约束的内容



在内部，接口值可以看做包含值和具体类型的元组：

```go
(value, type)
```



即便接口内的具体值为 nil，方法仍然会被 nil 接收者调用。**保存了 nil 具体值的接口其自身并不为 nil**。



**类型断言** 提供了访问接口值底层具体值的方式。

```go
t := i.(T)
```



# 指针

Go 拥有指针。指针保存了值的内存地址。

`var point *int` 是声明指向 `int` 类型值的指针`point`。

`point = &10`，`&10` 是取`10`的地址

`*point = 20` 是取指针`point`的内容，将其改为20

```go
p := &i         // 指向 i
fmt.Println(*p) // 通过指针读取 i 的值
*p = 21         // 通过指针设置 i 的值
fmt.Println(i)  // 查看 i 的值
```



涉及到传指针想要改变内部值一般只用在函数吧，函数不传指针就会调用结构体的拷贝，但是map里面你放什么指针啊，是没有意义的。

是有意义的，减少存储，在copy的时候直接copy指针就行，性能会好。

# 全局变量

全局变量

- 函数内定义的变量称为局部变量
- 函数外定义的变量称为全局变量，全局变量可用包来访问，包名取路径上最后一个的名字

包

一个包里的东西可以相互引用

Exported and Unexported

In go, fields and variables that start with an Uppercase letter are "Exported", and are visible to other packages. Fields that start with a lowercase letter are "unexported", and are only visible inside their own package.

在go中，以大写字母开头的字段和变量是“导出的”，对其他包可见。 以小写字母开头的字段是“未导出的”，并且仅在它们自己的包内可见。

# 字符串处理记录

## 去掉UUID

```go
func main() {
	wfID:="package-lane-match:f018e338-5300-4131-b802-46468cc563e2"
	s := []rune(wfID)
	s = s[:len(s)-37]			//切片截取
	fmt.Println(string(s))

}
```



## 正则表达式匹配

需求：某一类字符串（记作loop_string）形如：“...... _123”，就是尾部由下划线和数字构成，现在要判断某一个字符串是否属于loop_string，如果属于就将其最后尾部删掉，如果不属于就很好



两种解决方案，都要依赖于正则表达式

1. 判断是否匹配，然后暴力反向遍历删掉

   ```go
   func main() {
   	wfID:="package-lane-match:f018e338-5300-4131-b802-46468cc563e2"
   	s := []rune(wfID)
   	s = s[:len(s)-37]			//切片截取
   	fmt.Println(string(s))
   
   }
   ```

   

2. 直接正则表达式匹配，匹配完以后取出一部分

   ```go
   	s:="1234_11"
   	pattern:=regexp.MustCompile(`(.*)_\d`)//构造正则表达匹配pattern
   	res := pattern.FindStringSubmatch(s)
   	fmt.Println(res[1])//res[0]取出整个字符串，res[1]取出第一个括号中的，以此类推。res=nil则匹配失败
   ```

   

# 包导入

import side effect

当导入一个包时，该包下的文件里所有init()函数都会被执行，然而，有些时候我们并不需要把整个包都导入进来，仅仅是是希望它执行init()函数而已。这个时候就可以使用 import _ " "引用该包。即使用【import _ 包路径】只是引用该包，仅仅是为了调用init()函数，所以无法通过包名来调用包中的其他函数。

```golang
import (
    _ "github.com/lib/pq"
    _ "image/png"
    ...
)
```





# Channel

4个例子：

```go
func main() {
	c:=make(chan int)
	for i:=0;i<10;i++{
    go func(){
			c<-i
		}()
	}
	
	for i:=0;i<10;i++{
		fmt.Print(" ",<-c)
	}
}
// 输出 10 10 10 10 10 10 10 10 10 10. 因为此时routine放入信道的是共享变量，共享变量的值会不断改变，最终都为10
func main() {
	c:=make(chan int)
	for i:=0;i<10;i++{
    i:=i
    go func(){
			c<-i
		}()
	}
	
	for i:=0;i<10;i++{
		fmt.Print(" ",<-c)
	}
}
// 输出 9 0 1 2 3 4 5 7 6 8，此时routine放入信道的是新的i，属于局部变量，相当于对共享变量进行了拷贝，值不会随着外部for的变化而变化，所以信道内部最终输出的就是他刚放进去的样子。输出之所以是随机的，是因为go routine的执行是随机的
func main() {
	c:=make(chan int)
	for i:=0;i<10;i++{
         go func(i int){
			c<-i
		}(i)
	}
	
	for i:=0;i<10;i++{
		fmt.Print(" ",<-c)
	}
}
// 输出 9 4 2 3 6 5 7 8 1 0 此时routine放入信道的是函数参数，这里是传值，所以传递的是拷贝而不是真正的值，本质上和2一样
func main() {
	c:=make(chan int)
	for i:=0;i<10;i++{
         go func(i *int){
			c<-*i
		}(&i)
	}
	
	for i:=0;i<10;i++{
		fmt.Print(" ",<-c)
	}
}
// 输出 10 10 10 10 10 10 10 10 10 10 此时routine放入信道的是i的指针，和1本质上是一样的
```

## channel

Channel是**带有类型**的管道。一个channel是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息。

- 创建

```go
ch := make(chan int)
```

- 关闭

Channel还支持close操作，用于关闭channel，随后对基于该channel的任何**发送操作**都将导致panic异常。但是仍然可以对关闭的channel进行读操作。当已经发送的数据都被成功接收后，后续的接收操作将不再阻塞，它们会立即返回一个零值，从而接受端会有永无休止的0值。所以接收端不建议用for 死循环来接受数据，而是采用range：

```go
for x := range naturals {
  squares <- x * x
}
```

此时channel中如果没有东西，就会自动跳出for循环。

- 无缓存channel

```go
ch := make(chan int)
```

默认情况下，发送和接收操作在另一端准备好之前**都会阻塞**。这使得 Go 程可以在没有显式的锁或竞态变量的情况下进行同步。

换句话说，对于无缓存信道，发送端发送后就进入阻塞，直到接收端接受才继续运行；接收端开启接受时也进入阻塞，直到发送端发东西来时才继续运行。这会导致两个goroutine做一次同步操作(信道里的东西传送完，两边才都开始各自运行)

形象的理解，两边goroutine的运行时序会正好被卡在channel处，然后以此为起点同时开始跑。

```go
routine1:
  ...
  channel<- 123
	funcA()

routine2:
  ...
  <-channel
	funcB()

此时funcA() funcB() 会同时开始跑，这就是基于阻塞的同步机制
```

- Goroutines 泄露

如果我们使用了无缓存的channel，那么两个慢的goroutines将会因为没有人接收而被永远卡住。这种情况，称为**goroutines泄漏**，这将是一个BUG。和垃圾变量不同，泄漏的goroutines并不会被自动回收，因此确保每个不再需要的goroutine能正常退出是重要的。

- 消息事件

```go
基于channels发送消息有两个重要方面。首先每个消息都有一个值，但是有时候通讯的事实和发生的时刻也同样重要。当我们更希望强调通讯发生的时刻时，我们将它称为消息事件。有些消息事件并不携带额外的信息，它仅仅是用作两个goroutine之间的同步，这时候我们可以用struct{}空结构体作为channels元素的类型，虽然也可以使用bool或int类型实现同样的功能，done <- 1语句也比done <- struct{}{}更短。
```

## select

selector可以在操作一个case的时候对另一个case屏蔽，达到一种加锁的效果

一个例子，我们想统计channelA和channelB发来的信号的次数，这就需要一个共享变量。但是对这个共享变量进行操作的时候要加锁，否则会出现并发问题。

```golang
counter := 0
go func(){
   for{
      _ := <- chA
      counter += 1
   }
}()

go func(){
   for{
      _ := <- chB
      counter += 1
   }
}()
```

用select就能解决这个问题，因为select在进入一个case的时候对另一个case是直接跳过的

```golang
counter := 0
for {
  select {
    case _ := <- chA:
    counter += 1
    case _ := <- chB:
    counter += 1
  }
}
```



# context

![preview](https://pic4.zhimg.com/v2-8e70419cb07e99bda656f23f3eb75dcb_r.jpg)



一种推荐的 ctx.Done()的方法。

```
//  func Stream(ctx context.Context, out chan<- Value) error {
//     for {
//        v, err := DoSomething(ctx)
//        if err != nil {
//           return err
//        }
//        select {
//        case <-ctx.Done():
//           return ctx.Err()
//        case out <- v:
//        }
//     }
//  }
```



# 值类型和引用类型

- 值类型分别有：int系列、float系列、bool、string、数组和结构体
- 引用类型有：指针、slice切片、管道channel、接口interface、map、函数等

值类型的特点是：变量直接存储值，内存通常在栈中分配

引用类型的特点是：变量存储的是一个地址，这个地址对应的空间里才是真正存储的值，内存通常在堆中分配

# Error

你可以在[这里](https://golang.org/src/errors/errors.go) 看到它的简单实现。它做的事情就是保存一个 `string`，同时，这个字符串是由 `Error` 方法返回的。

```go
err.Error()
```

# WaitGroup

geerpc中有提及

# go GC

# reflect

为什么是type和value，因为一个变量唯二的标识就是type和value，我们声明一个对象， `int a = 1`此时`int`和`1`就是两个重要的标识。



```go
argv = reflect.New(m.ArgType.Elem())
for i := 0; i < s.typ.NumMethod(); i++ 
method := s.typ.Method(i)
argType, replyType := mType.In(1), mType.In(2)
f := m.method.Func
returnValues := f.Call([]reflect.Value{s.rcvr, argv, replyv})

```









golang为什么不用锁 (也是有锁的)

锁会避免什么问题（修改共享资源）

用信道可以怎样解决锁解决的问题

Golang并发问题（lock）

带缓冲的信道

select



