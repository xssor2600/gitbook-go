# go unsafe\_ptr和uintptr关系

在golang的源代码中，经常看到几种指针类型关键字：`*`,`unsafe.Pointer`,`uintptr`。那么这几种指针之间有什么关系，每个指针适用的应用场景又是哪些呢？

## Golang指针

* **\*类型**：普通指针类型，用于传递对象地址，`不能进行指针运算`。多数通常应用与开发者和源代码中。
* **unsafe.Pointer**: 通用指针类型，用于转换不同类型指针，`不能进行指针运算`，不能读取内存存储的值。（_指针转换器_）
* **uintptr**: 用于指针运算，GC不会把uintptr当作指针，uintptr无法持有对象。uintptr类型目标会被GC回收。通常多数都是在源代码中使用。

> **unsafe.Pointer 是桥梁，可以让任意类型的指针实现相互转换，也可以将任意类型的指针转换为 uintptr 进行指针运算。**

**要在某个指针地址上加上一个偏移量，Pointer是不能做这个运算的，那么谁可以呢?**

可以做指针运算的只有`uintptr`，所以需要将Pointer类型转换成uintptr类型，做完加减法后，转换成Pointer，通过\*操作，就可以取值，修改值了。

```go
b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))

//通过uintptr将通用指针p转换成二进制，在于x值进行指针运算，加上偏移量
//最后又通过通过指针将内存地址值转换成指针返回
// Pointer不能指针运算，只能转换成uintptr
func add(p unsafe.Pointer, x uintptr) unsafe.Pointer {
    return unsafe.Pointer(uintptr(p) + x)
}
```

**`unsafe.Pointer` 可以让你的变量在不同的普通指针类型转来转去，也就是表示为任意可寻址的指针类型。而 `uintptr` 常用于与 `unsafe.Pointer` 打配合，用于做指针运算。**

## unsafe.Pointer

unsafe.Pointer称为通用指针，官方文档对该类型有四个重要描述：

1. 任何类型的指针都可以被转化为Pointer
2. Pointer可以被转化为任何类型的指针
3. uintptr可以被转化为Pointer
4. Pointer可以被转化为uintptr

unsafe.Pointer是特别定义的一种指针类型（译注：类似C语言中的void类型的指针），在golang中是用于各种指针相互转换的桥梁，它可以包含任意类型变量的地址。但是不可以直接通过\*p来获取unsafe.Pointer指针指向的真实变量的值，因为我们并不知道变量的具体类型。

什么叫 **unsafe.Pointer**它可以包含任意类型变量的地址？

可以得到一点，就是 **unsafe.Pointer** 转换的变量，该变量一定要是指针类型，否则编译会报错。

```go
a := 1
b := unsafe.Pointer(a) //报错
b := unsafe.Pointer(&a)
```

**一个普通的T类型指针可以被转化为unsafe.Pointer类型指针，并且一个unsafe.Pointer类型指针也可以被转回普通的指针，被转回普通的指针类型并不需要和原始的T类型相同。**

**unsafe.Pointer 转换的变量类型，一定是指针类型**,**& 取址，\* 取值；**

```go
 func Float64bits(f float64) uint64 {
    fmt.Println(reflect.TypeOf(unsafe.Pointer(&f)))  //unsafe.Pointer
    fmt.Println(reflect.TypeOf((*uint64)(unsafe.Pointer(&f))))  //*uint64<br>　　　　　//(*uint64)(&f)  //这种类型转换语法是无效的
    return *(*uint64)(unsafe.Pointer(&f))  //在变量前加*，是取值
}
```

> 当一个协程的栈的大小改变(grow)时，一个新的内存段将申请给此栈使用。原先已经开辟在老的内存段上的内存块将很有可能被转移到新的内存段上，或者说这些内存块的地址将改变。 **相应地，引用着这些开辟在此栈上的内存块的指针（它们同样开辟在此栈上）中存储的地址也将得到刷新**。 这里很重要，这也是 uintptr 变量不要轻易使用的原因。

## uintptr

uintptr是golang的内置类型，是能存储指针的整型，在64位平台上底层的数据类型是:(uintptr没有指针的语义)

```go
typedef unsigned long long int  uint64;
typedef uint64          uintptr;


// uintptr is an integer type that is large enough to hold the bit pattern of
// any pointer.
// 是一个整数类型，作用是存放任意指针对应地址的二进制值（足够大）
type uintptr uintptr


// 通常使用方式
// 其中p可以是任意值
uintptr(p)
```

一个unsafe.Pointer指针也可以被转化为uintptr类型，然后保存到指针型数值变量中（注：这只是和当前指针相同的一个数字值，并不是一个指针），然后用以做必要的指针数值运算。

```go
func Test_addr(t *testing.T){
    //unsafe.Offsetof 函数的参数必须是一个字段 x.b, 然后返回 b 字段相对于 x 起始地址的偏移量
    var x struct {
        a bool
        b int16
        c []int
    }
    //uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)
    // uintptr运算x起始地址+b相对于x的地址偏移量得到x.b的内存地址,在将uintptr转成通用指针
    pd := (*int16)(unsafe.Pointer(uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)))
    *pd = 42
    fmt.Println(x.b)  // 42

}
// 通常golang中底层源码指针都是通过unsafe.Pointer通用返回
```

我们不可以直接通过_p来获取unsafe.Pointer指针指向的真实变量的值，因为我们并不知道变量的具体类型。所以上述返回`x.b`内存地址的时候，是通过\*\*(\\_int16)\*\*来明确类型的，需要强制转换。

**对于uintptr的一些说明：**

1. 如果一个对象只有一个 uintptr 表示的地址表示"引用"关系，那么这个对象会在GC时被无情的回收掉，那么uintptr表示一个不确定的地址。
2. 如果uintptr表示的地址指向的对象发生了copy移动(比如协程栈增长，slice的扩容等)，那么uintptr也表示一个未知地址。（uintptr引用的地址不会因为内存copy而刷新）
3. unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收，并且在发生移动时候更新地址值。
