#### Go 程序执行过程（const>var>init)
[image](https://s3.ax1x.com/2020/12/20/rartZd.png)
从一下几点针对Go源代码中初始化过程进行学习理解。

> 什么是init函数，可以定义在那些地方，是否可以重复定义多个同名函数?

`init`函数是程序在编译源代码初始化过程中执行的函数，类似一个编译器提供给开发者的一个钩子。

**可以定义在go文件或者package下，是可以同时定义多个同名的init函数的。**



> init函数的调用顺序？要从横向和纵向两个方向来进行思考。

由上图可知，`init`函数是在常量`const`和变量`var`之后执行的，因为init函数内部可以引用常量和变量定义的内容。`const>var>init`

在不同的包中对init函数调用顺序规则是：**优先加载或者执行导入包下的init函数，然后再执行本包或者文件的init函数。**


特别的：

- 在go文件中：
  1. 同一个go文件可以定义多个init函数。
  2. 在同一个go文件中多个init函数执行顺序：**按照在代码中编写的顺序先后依次执行。**
- 在包package中：
  1. 同一个包package下可以多个go文件定义多个init函数。
  2. 同一个package中，不同go文件的init函数执行顺序：**按照文件名顺序先后执行各个文件中的init函数。**

```go
.
├── Demo1
│   └── A
│       └── initTest.go  // -> Demo2.TestDemo()
├── Demo2
│   ├── init2.go
│   └── initTest.go
├── go.mod
├── go.sum
└── hello.go   // -->  A.TestDemo()

// output
Demo2.init2
Demo2.initTest init..
demo2 const
demo2_str_var
demo1.a init..
demo1.a
demo1.a_var
```

