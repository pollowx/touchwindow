#### GO Study



##### 



---

##### GO语言中的数组和slice. slice是个好东西，值得深究，slice对与底层的array是共享的，index后，findArrayIndex. 我们来看下sample和源码

##### len是slice的实际数据数量，cap实际上是容量，在扩容和数据拷贝的时候会评估该数值

![image-20200123155832204](/Users/bytedance/Library/Application Support/typora-user-images/image-20200123155832204.png)

```go
func printSliceCapAndLen(slice []string) {
  fmt.Printf("len:%d, cap:%d, value:%v \n", len(slice), cap(slice), slice)
}

func main() {
  vb := []string{0: "000", 1: "111", 2: "222", 3: "333", 4: "444"} // 0,1,2,3,4
  vb1 := vb[1:3]                                                   // 从1到2的下标1,2
  vb2 := vb[2:3]                                                   // 从2到3的下标2

  printSliceCapAndLen(vb)  // len:5, cap:5, value:[000 111 222 333 444] 
  printSliceCapAndLen(vb1) // len:2, cap:4, value:[111 222]
  printSliceCapAndLen(vb2) // len:1, cap:3, value:[222] 

  fmt.Println()
  vb1[1] = "111Change"
  fmt.Println(vb1) // [111 111Change]  index = 1的值被改变了
  fmt.Println(vb2) // [111Change] index = 1的值被改变了

  vb = append(vb, "555")
  fmt.Println("--------------555--------------")
  printSliceCapAndLen(vb) // len:6, cap:10, value:[000 111 111Change 333 444 555] 
  printSliceCapAndLen(vb1) // len:2, cap:4, value:[111 111Change] 
  printSliceCapAndLen(vb2) // len:1, cap:3, value:[111Change] 

  vb = append(append(append(append(append(vb, "666"), "777"), "888"), "999"), "101010")
  fmt.Println("--------------10101010101--------------")
  printSliceCapAndLen(vb) // len:11, cap:20, value:[000 111 111Change 333 444 555 666 777 888 999 101010] 
  printSliceCapAndLen(vb1) // len:2, cap:4, value:[111 111Change] 
  printSliceCapAndLen(vb2) // len:1, cap:3, value:[111Change] 
}

```

##### 上面的输出我们可以看到，slice的cap和len的差异，并且append的过程中slice cap扩容的变化, 源码我们看下, 发现扩容的规律是1024前每次扩容2倍，反之每次扩容1.25倍

```go
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
  newcap = cap
} else {
  if old.len < 1024 {
    newcap = doublecap
  } else {
    // Check 0 < newcap to detect overflow
    // and prevent an infinite loop.
    for 0 < newcap && newcap < cap {
      newcap += newcap / 4 // 扩容1，25倍
    }
    // Set newcap to the requested cap when
    // the newcap calculation overflowed.
    if newcap <= 0 {
      newcap = cap
    }
  }
}
```





---

##### 可以说GO里面都是值传递，call func中参数的传递明显是一个副本，所以指针代表的对变量地址的引用作为参数显得很便利。可以这样说：1. 如果在一个func里面不需要对值update，只是进行值的判断等，那么可用非指针参数 2. 如果需要在func里面对参数进行update，那么则必须传递指针参数

```go
func fins(arr *[3]int) { // 在fins里面对index的下标进行了修改，必须用指针传递引用参数
  for i, _ := range arr {
    arr[i] = i + 1
  }
}

func main() {
  arr := [3]int{}
  fins(&arr)
  for _, value := range arr {
    fmt.Println(value) // 1, 2, 3
  }
}
```





---

##### 这里有一个关键字iota, 比较有意思，在其他语言里并不是经常见到，再次记录下

```go
const (
  _  = 1 << (10 * iota) // iota是0, 下面每一行iota都会增加1
  M1                    // iota是1, 1024
  M2                    // iota是2, 1048576
  M3                    // iota是3, 1073741824
  M4                    // iota是4, 1099511627776
  M5                    // iota是5, 1125899906842624
  M6                    // iota是6, 1152921504606846976
)
```





---

##### 由于众所周知的原因，GO语言中的源码文件编码是UTF-8, 所以在判断字符上的很多场景会带来很大便利，比如contains/startWith,endWith等

```go
str := "bingxin"
pre := "bin"
fmt.Println("prefix : ", len(str) >= len(pre) && str[:len(pre)] == pre)

func containSubStr(pre, str string) bool { // contains
	if pre == "" {
		return true
	}
	for i := 0; i < len(str); i++ {
		if len(str) >= len(pre) && str[:len(pre)] == pre {
			return true
		}
	}
	return false
}
```

###### 并且go里面的string是共享底层数据的，所以在字符串上的操作会更加便利。如下图所示

```go
var str = "hello, world"
var st  = "hello"
var sr  = "world" 
// 实际上在存储上hello, world是一个共用的地址来存储这个数据
// string对象有数据和index的下标来表示这三个string的指向
```



![image-20200122145156560](/Users/bytedance/Library/Application Support/typora-user-images/image-20200122145156560.png)





---

##### 要注意在变量初始化的时候踩坑，局部变量和全局变量的作用域问题

```go
var ccs int // 注意ccs并没有被准确赋值
func main() {
  ccs := fibonacci(5) // 这里的ccs是局部变量，go的编译器按照从小到大的原则来find变量，找到了就会停止
  fmt.Println(ccs) 
}

// 正确的应该是
func main() {
  ccs = fibonacci(5) // 这里的ccs是全局变量
  fmt.Println(ccs) 
}
```



---

##### init函数不能被调用或者引用, 每个文件可以有多个init函数，在main执行之前，所有的init都已经被执行. 初始化是自上而下的.

```go
func init()  { // 最先被执行
  fmt.Println("init First")
}
func init()  { // 其次执行
  fmt.Println("init Second")
}
func main() { // 最后执行
  // xxx do something
}
```



---

##### Go语言中也有对象逃逸的问题，对象是在栈上分配还是在堆上分配是有区别的。包级别的变量如果在函数内部初始化，那么这个内存的分配就会在堆上分配，该变量逃逸出了该func.

```go
var global *int // 包级别的变量（Java中的类变量）
func deep() {
  var x int
  x = 1
  global = &x // global对象是全局变量，x变量就需要在堆上分配。deep推出后，x仍能通过global访问到
}
```



---

##### Go语言在垃圾回收时是从包级别或者当前运行的函数每一个局部变量开始找起，通过指针或者引用路径遍历，找不到就要被垃圾收集器回收





