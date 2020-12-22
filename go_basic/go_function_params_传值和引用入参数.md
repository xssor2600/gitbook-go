#### GO值传递和引用传递

> 值传递： 值传递是指在调用函数时将实际参数复制一份传递到函数中，不会影响原内容数据 

> 引用传递

- 引用传递是在调用函数时将实际参数的地址传递到函数中，那么在函数中对参数所进行的修改将影响原内容数据
- Go中可以借助指针来实现引用传递。函数参数使用指针参数，传参的时候其实是复制一份指针参数，也就是复制了一份变量地址
- 函数的参数如果是指针，当调用函数时，虽然参数是按复制传递的，但此时仅仅只是复制一个指针，也就是一个内存地址，这样不会造成内存浪费、时间开销

1. 函数传变量传值和引用

   ```go
   func TestReference(t *testing.T) {
   	a:=200
   	fmt.Printf("变量a内存地址%p,值=%v\n",&a,a)  // 输出变量a的内存地址，和内存地址存储的值
   	chageIntVal(a) // 传值，会复制入参a的值，开辟新内存地址保存a的值，传入a=a1，a1
   	fmt.Printf("chageIntVal调用后，变量a内存地址%p,值=%v\n",&a,a) //传值入参并不会改变原变量值
   	changeintPtr(&a)  // 传入变量a的内存地址,传值引用，值为内存地址a的拷贝,&a=&a1,a1=0x1234..,两者指向同样位置
   	fmt.Printf("changeintPtr调用后，变量a内存地址%p,值=%v\n",&a,a)
   
   }
   
   func chageIntVal(n int){
   	fmt.Printf("changeIntVal函数，传递的参数n的内存地址：%p，值为：%v\n",&n,n) // 输出相同值n,存储的新内存地址
   	n=100
   }
   
   func changeintPtr(n *int){
   	fmt.Printf("changeIntPtr函数，传递的参数n的内存地址：%p，值为：%v\n",&n,n) // 传入n内存地址，&n为新内存地址，保存着入参地址
   	*n=50  // 通过指针修改
   }
   
   // 输出
   变量a内存地址0xc0000b1080,值=200
   changeIntVal函数，传递的参数n的内存地址：0xc0000b1090，值为：200
   chageIntVal调用后，变量a内存地址0xc0000b1080,值=200
   changeIntPtr函数，传递的参数n的内存地址：0xc0000ac320，值为：0xc0000b1080
   changeintPtr调用后，变量a内存地址0xc0000b1080,值=50
   ```

   

2. Slice切片函数传值和引用
   数组类型变量的地址就是其首元素地址。

   ```go
   // 数组传值和引用： 数组通过传值传递，通过拷贝数组数据进行传递
   func TestArray_1(t *testing.T) {
   	arr2 := [3]int{
   		1,
   		2,
   		3,
   	}
   	fmt.Printf("1 %p - %v\n", &arr2, arr2)
   	fmt.Printf("1.0 %p - %v\n", &arr2[0], arr2[0])
   	Array_2(arr2)
   	fmt.Printf("4 %p - %v\n", &arr2, arr2)
   	fmt.Printf("4.0 %p - %v\n", &arr2[0], arr2[0])
   }
   
   func Array_2(arr [3]int) {  // 拷贝入参数组元素值，函数内修改的不影响原数组
   	fmt.Printf("2 %p - %v\n", &arr, arr)
   	fmt.Printf("2.0 %p - %v\n", &arr[0], arr[0])
   	arr[0] = 4
   	arr[1] = 5
   	arr[2] = 6
   	fmt.Printf("3 %p - %v\n", &arr, arr)
   	fmt.Printf("3.0 %p - %v\n", &arr[0], arr[0])
   }
   // 输出
   1 0xc0000ceba0 - [1 2 3]
   1.0 0xc0000ceba0 - 1
   2 0xc0000cebe0 - [1 2 3]
   2.0 0xc0000cebe0 - 1
   3 0xc0000cebe0 - [4 5 6]
   3.0 0xc0000cebe0 - 4
   4 0xc0000ceba0 - [1 2 3]
   4.0 0xc0000ceba0 - 1
   ```

   

   切片slice

   作为变量传值时候，会拷贝一份到新内存地址，但是新旧变量都指向相同的元素内存地址。

   将slice地址作为变量传递，不进行拷贝，直接传入相同内存地址。(会开辟新内存地址存储原slice的内存地址)

   ```go
   // slice源代码结构
   type slice struct {
    array unsafe.Pointer
    len int
    cap int
   }
   
   // slice作为变量传入:拷贝一份副本变量，指向相同元素指针
   func TestSlice_1(t *testing.T) {
   	arr2 := []int{
   		1,
   		2,
   		3,
   	}
   	fmt.Printf("1 %p - %v\n", &arr2, arr2)
   	fmt.Printf("1.0 %p - %v\n", &arr2[0], arr2[0])
   	Slice_2(arr2) // 原slice值拷贝，新内存存放新slice内存值，但是保留首元素内存指针（新旧拷贝值指向相同元素地址）
   	fmt.Printf("4 %p - %v\n", &arr2, arr2)
   	fmt.Printf("4.0 %p - %v\n", &arr2[0], arr2[0])
   }
   
   func Slice_2(arr []int) { // 通过修改元素内存指针修改原切片指向元素内容
   	fmt.Printf("2 %p - %v\n", &arr, arr)
   	fmt.Printf("2.0 %p - %v\n", &arr[0], arr[0])
   	arr[0] = 4
   	arr[1] = 5
   	arr[2] = 6
   	fmt.Printf("3 %p - %v\n", &arr, arr)
   	fmt.Printf("3.0 %p - %v\n", &arr[0], arr[0])
   }
   
   // 输出
   1 0xc0000b52a0 - [1 2 3]
   1.0 0xc0000ceba0 - 1
   2 0xc0000b52e0 - [1 2 3]
   2.0 0xc0000ceba0 - 1
   3 0xc0000b52e0 - [4 5 6]
   3.0 0xc0000ceba0 - 4
   4 0xc0000b52a0 - [4 5 6]
   4.0 0xc0000ceba0 - 4
   
   
   
   // slice将地址作为变量传入： 将原slice变量指针传入，没有拷贝发生
   func TestSlice_2(t *testing.T) {
   	arr2 := []int{
   		1,
   		2,
   		3,
   	}
   	fmt.Printf("1 %p - %v\n", &arr2, arr2)
   	fmt.Printf("1.0 %p - %v\n", &arr2[0], arr2[0])
   	Slice_3(&arr2)
   	fmt.Printf("4 %p - %v\n", &arr2, arr2)
   	fmt.Printf("4.0 %p - %v\n", &arr2[0], arr2[0])
   }
   
   func Slice_3(arr *[]int) {
   	fmt.Printf("2 %p - %v\n", arr, *arr)
   	fmt.Printf("2.0 %p - %v\n", &(*arr)[0], (*arr)[0])
   	(*arr)[0] = 4
   	(*arr)[1] = 5
   	(*arr)[2] = 6
   	fmt.Printf("3 %p - %v\n", arr, *arr)
   	fmt.Printf("3.0 %p - %v\n", &(*arr)[0], (*arr)[0])
   }
   
   // 输出
   1 0xc0000b52a0 - [1 2 3]
   1.0 0xc0000ceba0 - 1
   2 0xc0000b52a0 - [1 2 3]
   2.0 0xc0000ceba0 - 1
   3 0xc0000b52a0 - [4 5 6]
   3.0 0xc0000ceba0 - 4
   4 0xc0000b52a0 - [4 5 6]
   4.0 0xc0000ceba0 - 4
   
   
   // 
   func TestSliceValue(t * testing.T){
   	a :=[]int{1,2,3,4}
   	fmt.Printf("变量a的内存地址%p,首元素地址%p,值为:%v\n",&a,&(a[0]),a)
   	changeSliceValue(a)
   	fmt.Printf("changeSliceValue调用后｜变量a的内存地址%p,值为:%v\n",&a,a)
   	changeSlicePtr(&a)
   	fmt.Printf("changeSlicePtr调用后｜变量a的内存地址%p,值为:%v\n",&a,a)
   }
   
   func changeSliceValue(n []int){
   	fmt.Printf("changeSliceValue传入｜变量n的内存地址%p,首元素地址%p,值为:%v\n",&n,&(n[0]),n)
   	n[0] = 20
   }
   
   func changeSlicePtr(n *[]int){
   	fmt.Printf("changeSlicePtr传入｜变量n的内存地址%p,首元素地址%p,值为:%v\n",n,&(*n)[0],*n)
   	(*n)[1] = 30
   }
   
   // 输出
   变量a的内存地址0xc00012f2a0,首元素地址0xc000148bc0,值为:[1 2 3 4]
   changeSliceValue传入｜变量n的内存地址0xc00012f2e0,首元素地址0xc000148bc0,值为:[1 2 3 4]
   changeSliceValue调用后｜变量a的内存地址0xc00012f2a0,值为:[20 2 3 4]
   changeSlicePtr传入｜变量n的内存地址0xc00012f2a0,首元素地址0xc000148bc0,值为:[20 2 3 4]
   changeSlicePtr调用后｜变量a的内存地址0xc00012f2a0,值为:[20 30 3 4]
   ```

   

   

3. 结构体函数传值与引用传值

   ```go
   func TestStructReference(t *testing.T)  {
   	//函数传结构体
   	a:=Teater{"xll",200}
   	fmt.Printf("变量a的内存地址%p,值为：%v\n",&a,a)
   	changeStructVal(a)
   	fmt.Printf("changeStructVal函数调用后变量a的内存地址%p,值为：%v\n",&a,a)
   	changeStructPtr(&a)
   	fmt.Printf("changeStructPtr函数调用后变量a的内存地址%p,值为：%v\n",&a,a)
   }
   func changeStructVal(n Teater)  {
   	fmt.Printf("changeStructVal函数，传递的参数n的内存地址：%p，值为：%v\n",&n,n)
   	n.name="testtest"
   }
   func changeStructPtr(n *Teater)  {
   	fmt.Printf("changeStructPtr函数，传递的参数n的内存地址：%p，传递参数n内存地址内的值:%p,值为：%v \n",&n,n,n)
   	(*n).name="ptrptr"
   	(*n).age=899
   }
   
   // 输出
   变量a的内存地址0xc00000f340,值为：{xll 200}
   changeStructVal函数，传递的参数n的内存地址：0xc00000f380，值为：{xll 200}
   changeStructVal函数调用后变量a的内存地址0xc00000f340,值为：{xll 200}
   changeStructPtr函数，传递的参数n的内存地址：0xc000010330，传递参数n内存地址内的值:0xc00000f340,值为：&{xll 200} 
   changeStructPtr函数调用后变量a的内存地址0xc00000f340,值为：{ptrptr 899}
   ```

   可以看到，`struct`结构体若是传值，则会拷贝内存地址值，在函数内的修改不会影响其原来的值。

   若是传入内存地址作为入参，则函数内的修改是会改变其值的。

   
