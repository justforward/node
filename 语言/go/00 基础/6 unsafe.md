# 参考地址

https://mp.weixin.qq.com/s?__biz=MjM5MDUwNTQwMQ==&mid=2257483779&idx=1&sn=462a76ec6f36a012ed7d2705c227f562&chksm=a53918d5924e91c350045f332174d2de75c35efadb96b7896bdcf2c5027cc5171fe4ee4b2f0d&scene=178&cur_album_id=2059055387672117248#rd

# 作用


```
unsafe.Pointer

type Pointer *ArbitraryType

type ArbitraryType int
```

从命名来看， `Arbitrary` 是任意的意思，也就是说 Pointer 可以指向任意类型，实际上它类似于 C 语言里的 `void*`。

1.  任何类型的指针和 unsafe.Pointer 可以相互转换。
2.  uintptr 类型和 unsafe.Pointer 可以相互转换。

![[Pasted image 20221227171914.png]]


pointer 不能直接进行数学运算，但可以把它转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型。

```
// uintptr 是一个整数类型，它足够大，可以存储
`type uintptr uintptr`
```

还有一点要注意的是，uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收。而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。




## string 和 slice 的相互转换




