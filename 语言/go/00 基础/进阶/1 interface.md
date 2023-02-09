
# interface 转具体类型


golang interface convert to other type

1、string int float64
```go
func interface2String(inter interface{}) {

    switch inter.(type) {

    case string:
        fmt.Println("string", inter.(string))
        break
    case int:
        fmt.Println("int", inter.(int))
        break
    case float64:
        fmt.Println("float64", inter.(float64))
        break
    }

}

func main() {
    interface2String("jack")
    interface2String(1)
    interface2String(12.223)
}
```

2、to struct

```go
// use assert
p, ok := (Value).(user)

// use json
resByre,resByteErr:=json.Marshal(ResponseData)

var newData MnConfig
jsonRes:=json.Unmarshal(resByre,&newData)
```


鸭子类型，是动态编程语言的一种对象推断策略，它更关注对象如何被使用，而不是对象的类型本身。


# Go and duck 

**静态语言和动态语言的对比**


静态语言：java C++只能发，必须要显示地声明实现了某个接口，之后， 才能用在任何需要这个接口的地方。

Go语言既有动态语言的便利，同时又会进行静态语言的类型检查。不要求类型显示地声明实现了某个接口，只要实现了相关的方法即可，编译器就能检测到。

顺带再提一下动态语言的特点：

> 变量绑定的类型是不确定的，在运行期间才能确定 函数和方法可以接收任何类型的参数，且调用时不检查参数类型 不需要实现接口


# 值接受者和指针接收者的区别

## 方法

方法能给用户自定义的类型添加新的行为。它和函数的区别在于方法有一个接收者，给一个函数添加一个接受者，它就变成了方法。接受者可以是值接受者，也可以是指针接受者。

在调用方法的时候，值类型既可以调用值接受者的方法，也可以调用指针接受者的方法；指针类型既可以调用`指针接收者`的方法，也可以调用`值接收者`的方法。

也就是说，不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。

实际上，当类型和方法的接受者类型不同时，其实是编译器在备受做了一些工作，

![[Pasted image 20230108170014.png]]


# 值接收者和指针接收者


前面说过，不管接受者类型是值类型还是指针类型，都可以通过值类型或者指针类型调用，这里面实际上通过语法糖起作用的。

先说结论：实现了接受者是值类型的方法，相当于自动实现了接受者是指针类型的方法，而实现了接受者是指针类型的方法，不会自动生成对应接受者是值类型的方法。


## 两者分别在何时使用

如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者;如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。

使用指针作为方法的接收者的理由：

•方法能够修改接收者指向的值。

•避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做会更加高效。


如果类型具备“原始的本质”，也就是说他的成员都是由Go语言内置的原始类型，如字符串、整型值等，那就定义值接收者类型的方法，像内置的引用类型，如 slice map interface channel 这些类型比较特殊，声明他们的时候，实际上是创建了一个header，对于他们也是直接定义值接收者类型的方法，这样，调用函数的时候，是直接copy了这些类型的header，而header本身就是为了复制设计的。


# iface和eface的区别是什么

iface和eface都是Go中描述接口的底层结构体，区别在于iface描述的接口包含方法，而eface则是不包含任何方法的空接口：interface{}


## iface

```
type iface struct {  
   tab  *itab // 表示接口的类型以及赋值给这个接口的实体类型  
   data unsafe.Pointer // 指向堆内存的指针 分配数据空间  
}

type itab struct {  
   inter *interfacetype  // 描述的是接口类型
   _type *_type  // 包含内存对齐方式，大小等
   hash  uint32 // copy of _type.hash. Used for type switches.  
   _     [4]byte  
   fun   [1]uintptr // variable sized.fun[0]==0 means _type does not implement inter.  // 放置和接口方法对应的具体数据类型的方法类型，实现接口调用方法的动态分配
}

```

func数组的大小为1，要是接口定义了多个方法？
实际上，这里存储的是第一个方法的函数指针，如果有更多的方法，在它之后的内存空间继续存储。从汇编角度来看，通过增加地址就能获取到这些函数指针，没什么影响。这些方法是按照函数名称的字典序进行排列的。

interfacetype 类型，他描述的是接口的类型
```
type interfacetype struct {  
   typ     _type  
   pkgpath name  
   mhdr    []imethod  
}
```


可以看到，它包装了 `_type` 类型，`_type` 实际上是描述 Go 语言中各种数据类型的结构体。我们注意到，这里还包含一个 `mhdr` 字段，表示接口所定义的函数列表， `pkgpath` 记录定义了接口的包名。

通过一张图来看iface的结构体全貌：

![[Pasted image 20230115135044.png]]

## eface

```
type eface struct {  
   _type *_type  
   data  unsafe.Pointer  
}
```

相对于iface，eface就比较简单了，只维护了一个_type字段，表示空接口所承载的具体的实体类型，data描述了具体的值。

![[Pasted image 20230115142757.png]]

## example

```
package main  
  
import "fmt"  
  
func main() {  
  
   a := 100  
  
   // eface  
   var any interface{} = a  
   fmt.Println(any)  

   // iface
   g := Gopher{"Go"}  
   var c coder = g  
   fmt.Println(c)  
}  
  
type coder interface {  
   code()  
   debug()  
}  
  
type Gopher struct {  
   language string  
}  
  
func (p Gopher) code() {  
   fmt.Printf("I am coding %s language\n", p.language)  
}  
  
func (p Gopher) debug() {  
   fmt.Printf("I am debuging %s language\n", p.language)  
}

```

执行命令，打印出汇编语言：

```
go tool compile -S ./src/main.go
```

可以看到，main 函数里调用了两个函数：

```
func convT2E64(t *_type, elem unsafe.Pointer) (e eface)
func convT3I(tab *itab,elem unsafe,Pointer) (i iface)

```

上面两个函数的参数和 `iface` 及 `eface` 结构体的字段是可以联系起来的：两个函数都是将参数`组装`一下，形成最终的接口。

## _type 结构体

```
type _type struct {  
   // 类型大小  
   size       uintptr  
   ptrdata    uintptr // size of memory prefix holding all pointers  
    // 类型的hash值   
   hash       uint32  
   // 类型的flag和反射有关系  
   tflag      tflag  
   // 和内存对齐相关  
   align      uint8  
   fieldAlign uint8  
   //  类型的编号，有bool, slice, struct 等等等等  
   kind       uint8  
   equal func(unsafe.Pointer, unsafe.Pointer) bool  
   // gc相关  
   gcdata    *byte  
   str       nameOff  
   ptrToThis typeOff  
}
```

Go语言各种数据类型都是在_type字段的基础上，增加一些额外的字段来进行管理的

```
type arraytype struct {  
   typ   _type  
   elem  *_type  
   slice *_type  
   len   uintptr  
}  
  
type chantype struct {  
   typ  _type  
   elem *_type  
   dir  uintptr  
}  
  
type slicetype struct {  
   typ  _type  
   elem *_type  
}  
  
type functype struct {  
   typ      _type  
   inCount  uint16  
   outCount uint16  
}  
  
type ptrtype struct {  
   typ  _type  
   elem *_type  
}  
  
type structfield struct {  
   name       name  
   typ        *_type  
   offsetAnon uintptr  
}
```


这些数据类型的结构定义，是反射实现的基础

# 接口的动态类型和动态值

iface包含两个字段：tab是接口表指针，指向类型信息；data是数据指针，则指向具体的数据。
他们分别被称为动态类型和动态值。接口值包含动态类型和动态值


##  接口类型和 `nil` 作比较

### demo1


接口值的零值是指`动态类型`和`动态值`都为 `nil`。当仅且当这两部分的值都为 `nil` 的情况下，这个接口值就才会被认为 `接口值 == nil`。

### demo2 

```
  
package main  
  
import "fmt"  
  
type MyError struct {}  
  
func (i MyError) Error() string {  
   return "MyError"  
}  
  
func main() {  
   err := Process()  
   fmt.Println(err)  
  
   // 相当于 所以，虽然它的值是 nil，其实它的类型是 *MyError，最后和 nil 比较的时候，结果为 false。  
   fmt.Println(err == nil)  
}  
  
// 这里返回的error 接口  
func Process() error {  
   var err *MyError = nil  
   // 这里判断的err是具体的MyError 结构体了  
   fmt.Println(err==nil)  
   return err  
}
```

函数运行结果：

```
<nil>
false
```

这里先定义了一个 `MyError` 结构体，实现了 `Error` 函数，也就实现了 `error` 接口。`Process` 函数返回了一个 `error` 接口，这块隐含了类型转换。所以，虽然它的值是 `nil`，其实它的类型是 `*MyError`，最后和 `nil` 比较的时候，结果为 `false`。

### demo3

如何打印出接口的动态类型和值？



# 编译器自动检测类型是否实现接口

```
var _ io.Write=(*myWriter)(nil)

var _ io.Writer = myWriter{}
```


编译器由此会检查*myWriter类型是否实现了io.Writer接口，没有实现的情况下会报错。


# 接口的构造过程是怎么样的

# 类型转换和断言的区别

Go语言中不允许隐式类型转换，也就是说=两边，不允许出现类型不相同的变量。

类型转换、类型断言本质上都是把一个类型转换成另外一个类型，不同在于类型断言是对接口变量进行的操作