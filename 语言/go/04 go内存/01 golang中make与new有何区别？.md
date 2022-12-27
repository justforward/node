


对于基础类型（值类型），我们通过关键字var声明之后，就可以在程序中使用。基础类型通过关键字声明之后，存在默认值。比如int类型的零值是0，string类型的零值是""。

但是对于引用类型，我们不仅要通过关键字声明，还需要为它分配空间，否则我们将无法对这个变量进行赋值操作。

要分配内存，就引出来今天的`new`和`make`。

## new

```
// 返回一个指向该类型的内存地址指针
func new(Type) *Type
```

他接受一个参数，这个参数就是一个类型，分配好内存之后，返回一个指向该类型的内存地址的指针，同时请注意把分配的内存置为零，也就是类型的零值。

返回的是指向这个新分配零值的指针。

例子：

```
func main() {
  
   var i *int
   i=new(int)
   *i=10
   fmt.Println(*i)
  
}
```

我们的例子中，如果没有`*i=10`，那么打印的就是0(int 的零值)。这里体现不出来new函数这种内存置为零的好处，我们再看一个例子。

struct里面

```
import (
    "fmt"
    "sync"
)

type user struct {
    lock sync.Mutex
    name string
    age int
}

func main() {

    u := new(user) //默认给u分配到内存全部为0

    u.lock.Lock()  //可以直接使用，因为lock为0,是开锁状态
    u.name = "张三"
    u.lock.Unlock()

    fmt.Println(u)
}
```

运行

```
$ go run test2.go 
&{{0 0} 张三 0}
```


示例中的user类型中的lock字段我不用初始化，直接可以拿来用，不会有无效内存引用异常，因为它已经被零值了。

这就是new，它返回的永远是类型的指针，指向分配类型的内存地址。


## make


make也是用于内存分配的，但是和new不同。它只用于

-   chan
-   map
-   slice

的内存创建，而且它返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以就没有必要返回他们的指针了。


注意，因为这个三种类型是引用类型，所以必须得初始化，但不是置为0值，这个和new是不一样的。引用类型中包含着其他的属性来描述这个类型的，不能全部置为0值。

### 创建slice

```
make([]Type, len, cap)
```

cap可以省略。当cap省略时，默认等于len。此外cap >= len >= 0的条件必须成立。例如，创建一个len和cap均为10的int型slice。



###  创建map

```
make(map[keyType] valueType, size)
```

keyType表示map的key类型，valueType表示map的value类型。size是一个整型参数，表示map的存储能力，该参数可省略



### 创建channel

```
make(chan Type, size)
```


使用make创建channel，第一个参数是channel类型。size表示缓冲槽大小，是一个可选的大于或等于0的整型参数，默认size = 0。当缓冲槽不为0时，表示通道是一个异步通道。



# new 和make对比

## 相同

堆空间分配

## 不同

make只使用于slice map 以及channel的初始化，无可替代。

new 用于类型内存分配（初始化为0）不常用。

new的作用是初始化一个指向类型的指针（*T）。使用new函数来分配空间，传递给new函数的是一个类型，不是一个值。返回的是指向这个新分配的零值的指针


