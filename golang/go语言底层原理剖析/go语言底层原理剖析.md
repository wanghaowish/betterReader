# go语言底层原理剖析

## 1. go语言编译器

go编译器阶段：词法解析→语法解析→抽象语法树构建→类型检查→变量捕获→函数内联→**逃逸分析**→闭包重写→遍历并编译函数→SSA生成→机器码生成

编译器可以对代码进行一定程度的优化

#### 逃逸分析

逃逸分析是用于标识变量内存应该分配在栈还是堆。函数执行完成后，栈会被销毁，当继续访问被销毁栈上的对象指针，就会产生逃逸。

常见的内存逃逸场景：

- **在方法内将局部变量指针返回。**局部变量原本应该在栈中分配，在栈中回收。但是由于返回时被外部引用，因此其生命周期大于栈，则溢出。
- **发送指针或者带指针的值到channel中。**在编译时，是没有办法知道哪个 goroutine 会在 channel 上接收数据。所以编译器没法知道变量什么时候才会被释放。
- **在一个切片存储指针或者带指针的值。**一个典型的例子就是 []*string 。这会导致切片的内容逃逸。尽管其后面的数组可能是在栈上分配的，但其引用的值一定是在堆上。
- **slice 的背后数组被重新分配了，因为 append 时可能会超出其容量( cap )。** slice 初始化的地方在编译时是可以知道的，它最开始会在栈上分配。如果切片背后的存储要基于运行时的数据进行扩充，就会在堆上分配。
- **在interface上调用方法。**在 interface 类型上调用方法都是动态调度的 —— 方法的真正实现只能在运行时知道。想像一个 io.Reader 类型的变量 r , 调用 r.Read(b) 会使得 r 的值和切片b 的背后存储都逃逸掉，所以会在堆上分配。
## 2. 浮点数

fmt包打印浮点数核心是调用标准库的strconv.FormatFloat函数。

| 占位符 | 说明                                                   |
| :----: | ------------------------------------------------------ |
|   %b   | 无小数部分、二进制指数的科学计数法，如-123456p-78      |
|   %E   | 科学计数法，如-1234.456e+78                            |
|   %e   | 科学计数法，如-1234.456E+78                            |
|   %f   | 有小数部分但无指数部分，如123.456                      |
|   %F   | 等价于%f                                               |
|   %g   | 根据实际情况采用%e或%f格式（以获得更简洁、准确的输出） |
|   %G   | 根据实际情况采用%E或%F格式（以获得更简洁、准确的输出） |

shopspring/decimal 第三方库，在处理货币方面有一定优势

## 3. 类型推断

`:=`用于变量的类型推断。类型推断依赖于编译器的处理能力。

## 4. 常量与隐式类型转换

const关键字声明常量，声明时可以指定或者忽略类型。等号左边的叫命名常量，等号右边的叫未命名常量，未命名常量只会在编译期间存在，因此不会存在在内存中。命名常存在内存静态只读区， 不能被修改。go语言禁止对常量进行取地址操作。

隐式类型转换的规则是有类型常量优先于无类型。无类型常量运算时的优先级为：复数(Imag)>浮点数(float)>符文数(rune)>整数(int)

## 5. 字符串的本质与实现

go语言中字符常量存储于静态编译区，字符串不能被修改，只能被访问。字符串本质是一个定长的字符数组。字符常量的拼接发生在编译时，字符串常量的拼接发生在运行时。字节数组和字符串的相互转换并不是无损的指针引用，而是涉及了复制。

字符串的struct：

```go
type StringHeader struct{
    Data uintptr //指向底层的字符数组
    Len int //代表字符串的长度
}
```

字母占据1个字节，中文一般占据3个字节。strings库内包含很多字符处理函数。strconv包含很多字符串转换的函数。``可以换行，""不能换行。字符串拼接的原理并不是简单的将一个字符串合并到另一个字符串中，而是找一个更大的空间，通过内存复制的形式将字符串复制到其中。

## 6. 数组

三种声明方式

```go
var arr [3]int
var arr2 =[3]int{1,2,3}
arr3:=[...]int{1,2,3} //语法糖 ... 这种声明方式在编译时自动推断长度
```

数组在赋值和函数调用时的形参都是值复制。数组在编译时会进行重要的优化，当数组长度小于4时，运行时数组会被放置在栈中，大于4则会放在内存的静态只读区。数组一般在Go语言中比较少用。

## 7. 切片

切片相对于数组而言在Go中更常用。切片是长度可变的序列，序列中的每个元素都有相同的类型。它和数组不同的是切片不需要指定长度。切片是一种轻量级的数据结构，提供了访问数组任意元素的功能。

```go
type SliceHeader struct{
    Data uintptr
    Len int
    Cap int
}
```

指针指向切片元素对应的底层数组元素的地址。len对应切片中元素的数目，长度不能超过cap。容量一般是从切片的开始位置到底层数据的结束位置的长度。

### 切片的初始化

切片的初始化需要用到make函数。可通过make函数指定长度和容量，不指定容量的情况下默认容量等于长度。使用数组创建slice时slice和原数组共用一部分内存。

![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202207191423469.png)

```go
//nil切片
var slice []int
//空切片
silce := make( []int , 0 )
slice := []int{ }
```

![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202207191423045.png)

空切片和 nil 切片的区别在于，空切片指向的地址不是nil，指向的是一个内存地址，但是它没有分配任何内存空间，即底层元素包含0个元素。

### 切片的截取

切片和数组一样，切片中的数据仍然是内存中的一片连续区域。截取切片可以通过下标的方式来截断。被截断后的切片长度跟容量都发生了变化。被截取后的切片仍然指向原始切片的底层数据。

### 切片复制和数据引用

数组的复制是值复制，修改复制后的数组不会影响原数组，但是修改复制后的切片会影响原切片。切片的复制其实也是值复制，只不过这里复制的值是指对于运行时SliceHeader结构的进行复制。底层指针仍然指向相同的底层数据的数组地址。如果想完全拷贝切片可以使用copy函数。逻辑是新建一个内存并复制过去。

### 切片的扩缩容

append函数可以添加新元素到切片的末尾，它可以接受可变长度的元素，并且可以自动扩容。

切片的扩容策略：

如果切片的容量小于 1024 个元素，于是扩容的时候就翻倍增加容量。上面那个例子也验证了这一情况，总容量从原来的4个翻倍到现在的8个。

一旦元素个数超过 1024 个元素，那么增长因子就变成 1.25 ，即每次增加原来容量的四分之一。

注意：扩容扩大的容量都是针对原来的容量而言的，而不是针对原来数组的长度而言的。

删除切片的某个元素的话直接使用截取即可。

```go
a:=int(len(numbers)/2)
numbers:=append(numbers[:a],numbers[a+1:]...)
```



## 8. 哈希表（map）

哈希表的原理是将多个键值对分散存储在buckets中。给定一个keyhash算法会计算出键值对存储的位置。map是o(1)时间复杂度的操作。

### 哈希碰撞

哈希碰撞就是不同的键通过哈希算法可能产生相同的值。避免哈希碰撞的策略主要有两种：拉链法和开放寻址法。

- 拉链法

  将同一个桶中的元素通过链表的形式形成链接。随着桶中元素的增加，可以不断链接新的元素，同时不用预先为元素分配内存。它的不足之处在于它需要存储额外的指针用于链接元素，增加了整个哈希表的大小。同时由于链表存储的地址不连续，所以无法高效地利用CPU高速缓存。Go语言中的哈希表采用的是优化的拉链法，每一个桶中存储了8个元素用于加速访问。

- 开放寻址法

  所有的元素都存储在桶的数组中。当必须插入新的条目时，按照某种探测策略操作，直到找到使用的数组插槽为止。当搜索元素时，将按相同的循序扫描存储桶，直到查找到目标记录或者找到未使用的插槽为止。

### map

- 初始化：map一般通过make函数初始化。也可通过字面量形式初始化。

- 访问map 

  ```go
  v:=hash[key]
  v,ok:=hash[key] //ok表示当前key是否存在
  ```

- 赋值

  ```go
  hash[key]=value
  delte(hash,key)
  ```

- map的并发冲突

  map并不支持并发的读写。如果有业务需求也可以通过各种方法来实现。主要思路是通过加锁保证每个协程同步操作内存。

  - 加锁 互斥锁`sync.Mutex`  读写锁`sync.RwMutex`

  - 利用channel

  - 使用`sync.map`。标准库中的`sync.Map`是专为`append-only`场景设计的。`sync.Map`在读多写少性能比较好，否则并发性能很差

  - 使用第三方包的map。例如：`concurrent-map`提供了一种高性能的解决方案:通过对内部`map`进行分片，降低锁粒度，从而达到最少的锁等待时间(锁冲突)

### 哈希表底层结构

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}

```

其中：

```go
     - count代表map中的元素数量
     - flags代表当前map的状态（是否处在正在写入的状态等等）
     - 2的B次幂标识当前map中桶的数量，2^B=Buckets size
     - noverflow为map中溢出桶的数量。当溢出桶太多时，map会进行same-size map growth。其实质是避免桶过大导致内存泄露。
     - hash0代表生成hash的随机数种子
     - buckets指向当前map对应的桶的指针。
     - oldbuckets是在map扩容时存储旧桶的。当所有旧桶中的数据都已经转移到了新桶中时，则清空
     - nevacuate在扩容时使用，用于标记当前旧桶中小于nevacuate的数据已经被转移到了新桶中
     - extra存储map中的溢出桶
```

```go

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

tophash此字段顺序存储key的hash值的前8位。桶在存储的taophash字段后，会存储key数组和value数组。

### map原理图解

![image-20200919191800912](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202207251349167.png)

- 创建map

  ```
  // 初始化一个可容纳10个元素的map
  info = make(map[string]string,10)
  ```

  - 第一步：创建一个hmap结构体对象。

  - 第二步：生成一个哈希因子hash0 并赋值到hmap对象中（用于后续为key创建哈希值）。

  - 第三步：根据hint=10，并根据算法规则来创建 B，当前B应该为1。B的计算逻辑为 
    $$
    初始元素个数 ≤ 2^B * 6.5
    $$
    让B从0开始依次递增，直到遇到让该公式成立的最小B值即可.比如hint=52,52 ≤ 2^3 * 6.5 => 52≤ 52,B=3
    
    ```
    hint            B
    0~8				0
    9~13            1
    14~26           2
    ...
    ```
    
  - 第四步：根据B去创建去创建桶（bmap对象）并存放在buckets数组中，当前bmap的数量应为2.只有在map的数量>52也就是B≥4，才会生成溢出桶。
  
    - 当B<4时，根据B创建桶的个数的规则为：2<sup>B</sup>（标准桶）
    - 当B>=4时，根据B创建桶的个数的规则为：2<sup>B</sup> + 2<sup>B-4</sup>（标准桶+溢出桶）
  
    注意：每个bmap中可以存储8个键值对，当不够存储时需要使用溢出桶，并将当前bmap中的overflow字段指向溢出桶的位置。
  
- 写入map

  在map中写入数据时，内部的执行流程为：

  - 第一步：结合哈希因子和键 `name`生成哈希值 `011011100011111110111011011`。

  - 第二步：获取哈希值的`后B位`，并根据后B为的值来决定将此键值对存放到那个桶中（bmap）。

    ```
    将哈希值和桶掩码（B个为1的二进制）进行 & 运算，最终得到哈希值的后B位的值。假设当B为1时，其结果为 0 ：
    哈希值：011011100011111110111011010
    桶掩码：000000000000000000000000001
    结果：  000000000000000000000000000 = 0
    
    通过示例你会发现，找桶的原则实际上是根据后B为的位运算计算出 索引位置，然后再去buckets数组中根据索引找到目标桶（bmap)。
    ```

  - 第三步：在上一步确定桶之后，接下来就在桶中写入数据。

    ```
    获取哈希值的tophash（即：哈希值的`高8位`），将tophash、key、value分别写入到桶中的三个数组中。
    如果桶已满，则通过overflow找到溢出桶，并在溢出桶中继续写入。
    
    注意：以后在桶中查找数据时，会基于tophash来找（tophash相同则再去比较key）。
    ```

  - 第四步：hmap的个数count++（map中的元素个数+1）

- 读取map

  首先找到桶的位置，然后遍历tophash数组，如果找到了相同的hash就可以用指针的寻址操作找到对应的key和value。如果key的hash不存在指定桶的tophash数组中，就需要遍历溢出桶的数据。

  在map中读取数据时，内部的执行流程为：

  - 第一步：结合哈希引子和键 `name`生成哈希值。

  - 第二步：获取哈希值的`后B位`，并根据后B为的值来决定将此键值对存放到那个桶中（bmap）。

  - 第三步：确定桶之后，再根据key的哈希值计算出tophash（高8位），根据tophash和key去桶中查找数据。

    ```
    当前桶如果没找到，则根据overflow再去溢出桶中找，均未找到则表示key不存在。
    ```

- map的扩容

  在向map中添加数据时，当达到某个条件，则会引发字典扩容。

  扩容条件：

  - map中数据总个数 / 桶个数 （负载因子）> 6.5 ，引发翻倍扩容。
  - 使用了太多的溢出桶时（溢出桶使用的太多会导致map处理速度降低）。
    - B <=15，已使用的溢出桶个数 >= 2<sup>B</sup> 时，引发等量扩容。
    - B > 15，已使用的溢出桶个数 >= 2<sup>15</sup>时，引发等量扩容。

  ```
  func hashGrow(t *maptype, h *hmap) {
  	// If we've hit the load factor, get bigger.
  	// Otherwise, there are too many overflow buckets,
  	// so keep the same number of buckets and "grow" laterally.
  	bigger := uint8(1)
  	if !overLoadFactor(h.count+1, h.B) {
  		bigger = 0
  		h.flags |= sameSizeGrow
  	}
  	oldbuckets := h.buckets
  	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)
  	...
  }
  ```

  当扩容之后：

  - 第一步：B会根据扩容后新桶的个数进行增加（翻倍扩容新B=旧B+1，等量扩容 新B=旧B）。

  - 第二步：oldbuckets指向原来的桶（旧桶）。

  - 第三步：buckets指向新创建的桶（新桶中暂时还没有数据）。

  - 第四步：nevacuate设置为0，表示如果数据迁移的话，应该从原桶（旧桶）中的第0个位置开始迁移。

  - 第五步：noverflow设置为0，扩容后新桶中已使用的溢出桶为0。

  - 第六步：extra.oldoverflow设置为原桶（旧桶）已使用的所有溢出桶。即：`h.extra.oldoverflow = h.extra.overflow`

  - 第七步：extra.overflow设置为nil，因为新桶中还未使用溢出桶。

  - 第八步：extra.nextOverflow设置为新创建的桶中的第一个溢出桶的位置。

数据转移遵行写时复制的规则，只有在真正赋值时，才会选择是否需要进行数据转移，才会将旧桶的数据打散放入新桶。

## 9. 函数和栈

函数是程序中为了执行特定任务而存在的一系列执行代码。在Go语言中，函数是一等公民，这意味着可以将它看作变量，并且它可以作为参数传递，返回及赋值。它还可以具有多返回值。

### 函数闭包

闭包是在函数作为一类公民的编程语言中实现语法绑定的一种技术，闭包包含了函数的入口地址和其关联的环境。闭包和普通函数最大的区别在于它可以引用闭包外的变量。

### 函数栈

当函数执行时，函数的参数、返回地址、局部变量会被压入栈中，当函数退出时，这些数据会被回收。当函数还没有退出就调用了另一个函数时，形成了一条函数调用链。每个函数在执行过程中都是用一块栈内存来保存返回地址、局部变量、函数参数等，这块区域成为函数的栈帧。函数调用时参数的压栈操作是从右往左进行的。

### 栈扩容和栈转移

在Go语言中，每一个协程都有一个栈。栈扩容的重要一步是将旧栈的内容转移到新栈中。首先将协程的状态设置为_Gcopystack，以便在垃圾回收状态下不会扫描该协程带来错误。在分配到新栈后，如果有指针指向旧栈，那么需要将其调整到新栈。

### 栈调试

- 设置stackDebug为1（需要修改go的源码，并重新进行编译）
- 使用`debug.PrintStack`方法
- 获取某一时刻的堆栈信息，还可以使用标准库`pprof`。`pprof.LookUp("goroutine")`可以获取当前时刻协程的栈信息。

## 10. defer的延迟调用

defer是Go语言中的关键字。在Go语言中，defer一般用于资源的释放以及异常panic的处理。

```go
defer func(...){
    //实际处理
}()
```

### defer优势及特性

#### 优势

- 资源释放

  例如os包的文件操作：`defer src.close()`，锁的释放：`defer l.unlock()`

- 异常捕获

  defer的延迟执行特性为异常捕获提供了很好的时机。异常捕获通常结合recover函数一起使用。
#### 特性

- 延迟执行

  defer后的函数并不会立即执行，而是推迟到了函数结束后执行。这一特性一般用于资源的释放，有时也用于函数的中间件。还可以设计一种类似计算函数执行时间的日志。

- 参数预计算

  当函数到达defer语句时，延迟调用的参数将预先求值，传递到defer函数的参数将预先被固定，而不会等到函数执行完成后再传递参数到defer中。

- 参数多次执行与LIFO执行顺序

  last-in first-out （后进先出）。

defer的设计经历了复杂的演进过程，从最初的堆分配defer内存并放入协程的链表中，到Go1.13的栈分配将defer放置到栈内存并放入协程链表中，再到Go1.14后的内联defer，使得defer的性能已经与函数的直接调用相似。因此，在实践中，不需要考虑defer函数带来的性能损耗。
## 11. 异常和异常捕获

### panic函数

`func panic(interface())`panic函数可以存储任何形式的错误信息，并进行传递，在异常退出时会打印出来。Go程序在panic时会终止当前函数的正常执行，执行defer函数并逐级返回。Go语言运行阶段也会检查并触发panic。

### 异常捕获和recover

`func recover() interface{}`为了让程序在panic时仍然能执行后续的流程，Go语言提供了内置的recover函数用于异常恢复。

### panic结合recover

recover函数最终捕获的是最近发生的panic。

## 12. 接口和程序设计模式

Go语言中可以为任何自定义的类型添加方法，没有任何形式的基于类型的继承，使用接口来实现扁平化、面向组合的设计模式。在Go语言中，接口是一种特殊的类型，是其他类型可以实现的方法签名的集合。方法签名只包含方法名、输入参数和返回值。

### Go接口的使用

接口包含两种形式： 一种是带方法签名的接口，一种是空接口。`type InterfaceName interface{}`。Go当中接口是隐式的，只要某一类型的方法中实现了接口中的全部方法签名，就意味着此类型实现了这个接口。多个类型可以实现同一个接口，一个类型也可以同时实现多个接口。定义的接口也可以是其他接口的组合。

一个接口包含的方法越多，其抽象性就越低，表达的行为就越具体。Go语言中的接口可以使程序自然、优雅、安全的增长，接口的更改仅影响实现接口的直接类型。使用`i.(Type)`在运行时获取存储在接口中的类型。

### 空接口

空接口增强了代码的扩展性和通用性。

### 接口的比较

两个接口之间可以通过==和!=进行比较。接口的比较规则如下：

* 动态值为nil的接口变量总是相等的。
* 如果只有一个接口为nil，那么一定不相等。
* 如果两个接口不为nil，且接口变量具有相同的动态类型和动态类型值，那么两个接口是相等的。
* 如果接口存储的动态类型值是不可比较的，那么在运行时会报错。（可比较的类型有`bool`、数值型、字符、指针、数组，结构体内的所有成员都可比较的话，结构体也可以比较。切片、map、函数等不可比较）

## 13. 反射

反射可以在运行时探测到结构体变量中的方法名。反射为Go语言提供了复杂的、意想不到的处理能力及灵活些。这种灵活些以牺牲效率和可理解性为代价。

### 反射的基本使用方法

```go
func ValueOf(i interface{}) Value //反射的值
//reflect.Value类型中的Type方法可以获取当前反射的类型
func (v value) Type() Type

func TypeOf(i interface{}) Type //反射的类型

```

`reflect.Value`和`reflect.Type`都有Kind方法获取标识类型的Kind，通过Kind类型.可以方便地验证反射的类型。可以通过接口的断言语法对接口`reflect.Value`进行转换。也可以通过`reflect.Value`提供的转换具体类型的方法，这些特殊的方法可以加快转换的速度。但是要注意的是这些方法要转换的类型和实际类型需要相符。如果反射中存储的是指针或者接口，可以通过Elem方法访问指针或者接口返回的数据。

```go
	a := "test"
	v := reflect.ValueOf(a)
	b := v.Interface().(string)
	c := v.String()
	d := reflect.ValueOf(&a).Elem().String() //需要注意的是Elem方法只适用于Value存储的是指针或者接口。
```

修改反射的值可以用Set方法。

``` go
func (v Value) Set (x Value)//要求反射的类型必须是指针 可以用CanSet方法去获取当前的反射值是否可以赋值
```

应用反射的大部分情况都涉及结构体。

`NumField`函数可以获取结构体中字段的个数。`reflect.Type`的Field方法主要用于获取结构体的元信息。`reflect.Value`的Field方法主要返回结构体字段的值类型，后续可以使用它修改结构体字段的值。reflect.ValueOf()传递结构体指针，然后通过Elem方法获取指针指向的结构体值类型，再调用Field方法赋值。

```go
	var s struct {
		X int
		y float64
	}
	vs := reflect.ValueOf(&s).Elem()
	vs.Field(0).Set(reflect.ValueOf(1))//X可以被复制，但是y不可以
```

`reflect.Type`还有`Method`方法、`MethodByName`方法以及`NumMethod`方法。`reflect.Value`还有`MethodByName`方法。还可以通过Type方法把`reflect.Value`转换为`reflect.Type`。获取到代表方法的`reflect.Value`对象后，可以通过call方法在运行时调用方法。

```go
type User struct {
	Id   int
	Name string
	Age  int
}

func (u User) PrintAge(age *int) {
	fmt.Println(*age)
}

func (u User) PrintName(name string) {
	fmt.Println(name)
}
func main() {
	user := User{Name: "test", Age: 18}
	getType := reflect.TypeOf(user)
	getValue := reflect.ValueOf(user)
	for i := 0; i < getType.NumMethod(); i++ {
		m := getType.Method(i)
		fmt.Println(m.Name, m.Type)
	}
	methodValue := getValue.MethodByName("PrintAge")
	args := make([]reflect.Value, 0)
	age := 18
	args = append(args, reflect.ValueOf(&age))
	methodValue.Call(args)
	tf := getValue.MethodByName("PrintAge").Type()
	fmt.Println(tf.NumIn(), tf.NumOut(), getValue.NumMethod())
}
```

函数的动态调用和方法的动态调用是相同的。

## 14. 协程初探 

并发不等于并行！

### 进程和线程

线程是可以由调度程序独立管理的最小程序指令集，而进程是程序运行的实例。大多数情况下，线程是进程的组成部分。一个进程中可以由多个线程，这些线程并发执行并共享进程的内存等资源。而进程之间相互独立。

### 线程上下文切换

当发生线程上下文切换时，需要从操作系统用户态转移到内核态，记录上一个线程的重要寄存器值、进程状态等信息，这些信息存储在操作系统线程控制块中，当切换到下一个要执行的线程时，需要加载重要的CPU寄存器值，并从内核态切换到操作系统用户态，如果线程在上下文切换时属于不同的进程，那么需要更新额外的状态信息以及内存地址空间，同时将新的页表导入内存。不同进程的切换要显著慢于同一进程中的线程切换。

### 线程与协程

在Go语言当中，协程认为是轻量级的线程。协程与线程的区别在于以下几点：

* 调度方式 协程是用户态的，协程的管理依赖Go语言运行时的调度器，协程是从属于某一个线程的，多对多关系。

* 上下文切换速度 协程的速度要快于线程，因为协程切换不需要经过操作系统用户态与内核态的切换。并且协程切换只需要保留极少的状态和寄存器变量值。

* 调度策略 线程的调度大多都是抢占式的，操作系统调度器为了均衡执行周期，会定时发送中断信号。而Go的协程一般情况下都是协作式调度，可以主动将执行权限让给其他协程。当一个协程运行过长时间，Go调度器才会强制抢占其执行。

* 栈的大小 线程的栈一般都是创建时指定，默认栈会比较大（例如2 MB）。Go语言中的协程栈默认2 KB。线程的栈在运行时不能更改，Go语言的协程栈在会动态检测栈的大小并扩容。协程在实践中可以看作轻量的资源。

### 并发与并行

通俗的说，并发指同时处理多个任务的能力。

Go使用协程非常方便，只需要在特定的函数前加上关键字go即可。Go语言号称天生支持高并发，是开发大规模、高并发项目的极佳选择。

## 15. 深入协程设计和调度原理

### 协程的状态

![image-20220830150027066](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/image-20220830150027066.png)

* `_Gidle`为协程刚开始创建时的状态，当新创建的协程初始化后，会变为`_Gdead`的状态，`_Gdead`状态也是协程被销毁时的状态。

* `_Grunnable`表示当前协程在运行队列中，正在等待运行

* `_Grunning`代表当前协程正在被运行，已经被分配给了逻辑处理器和线程

* `_Gwaiting`表示当前协程在运行时被锁定，不能执行用户代码。在垃圾回收及channel通信时经常会遇到这种情况。

* `_Gsyscall`代表当前协程正在执行系统调用

* `_Gpreempted`是Go1.14新加的状态，代表协程G被强制抢占后的状态

* `_Gcopystack`代表在进行协程栈扫描时发现需要扩容或缩小协程栈空间，将协程中的栈转移到新栈时的状态。

还有几个状态（**_Gscan**,**_Gscanrunnable**,**_Gscanrunning**等）涉及垃圾回收阶段。

### 特殊协程g0和协程切换

每个线程中都有一个特殊的协程g0。它运行在操作系统线程栈上，其作用主要是执行协程调度的一系列运行代码。

在Go语言中没有直接暴露线程本地存储的编程方式。但是Go语言运行时的调度器使用线程本地存储将具体操作系统的线程和运行时代表现线程的m结构体绑定在一起，因此在任意一个线程内部，通过线程本地存储，都可以在任意时刻获取绑定到当前线程上的协程g、结构体m、逻辑处理器P、特殊协程g0等信息。

### 调度循环

![image-20220830161300798](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/image-20220830161300798.png)

调度循环指从调度协程g0开始，找到接下来要运行的协程g、正在从协程g切换到协程g0开始新一轮调度的过程。

从g0调度到协程g，经历了从schedule函数到execute函数再到gogo函数的过程。schedule函数处理具体的调度策略，选择下一个要执行的协程；execute函数执行一些具体的状态转移、协程g与结构体m之间的绑定等操作；gogo函数是与操作系统有关的函数，用于完成栈的切换及CPU寄存器的恢复。

执行完毕后，切换到g执行。当协程g主动让渡、被强占或退出后，又会切换到协程g0进行第二轮调度。mcall函数用于保存当前协程的执行现场，并切换到协程g0继续执行。切换到协程g0后会根据切换的原因执行不同的函数。例如用户调用Gosched函数则主动让渡执行权，执行gosched_m函数。如果协程已经退出，则执行goexit函数，将协程g放入p的freeg队列，方便下次重用。执行完毕后，再次调用schedule函数进行新一轮的调度循环。

### 调度策略

调度的核心策略位于schedule函数当中。schedule函数首先会检测程序是否处于垃圾回收阶段，如果是，则检测是否需要执行后台标记协程。

Go语言调度器将运行队列分为局部运行队列和全局运行队列。

局部运行队列就是每个P特有的长度为256的数组，该数组模拟了一个循环队列，其中runqhead标识了循环队列的开头，runqtail标识了循环队列的末尾。每次将G放入本地队列时，都从循环队列的末尾插入，而获取时从循环队列的头部获取。并且每个P内部还有一个特殊的runnext字段标识下一个要执行的协程，如果runnext不为空，则会直接执行runnext指向的协程，而不会去runq数组中寻找。

被所有P共享的全局运行队列存储在schedt.runq中。

```go
//局部运行队列
type p struct {
    ...
    // Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	//
	// Note that while other P's may atomically CAS this to zero,
	// only the owner P can CAS it to a valid G.
	runnext guintptr
    ...
}

//全局运行队列
type schedt struct {
    ...
    // Global runnable queue.
	runq     gQueue
    ...
}

// A gQueue is a dequeue of Gs linked through g.schedlink. A G can only
// be on one gQueue or gList at a time.
type gQueue struct {
	head guintptr
	tail guintptr
}
```

改进后的GMP模型：

![image-20220830174006279](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/image-20220830174006279.png)

Go为了避免循环往复的执行局部队列的G而不执行全局队列的G使用了一种策略：P中每执行61次调度，就需要优先从全局队列获取一个G到当前P中，并执行下一个要执行的G。

``` go
	if gp == nil {
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	} 
```

调度协程的优先级如下图所示。排除从全局运行队列获取的情况下，每个P在执行调度时都会先尝试从runnext中获取下一个要执行的G，如果runnext为空，则从局部运行队列runq获取，如果runq为空就会从全局运行队列schet.runq获取G，如果还没有，那就会尝试从其他P窃取可用的协程。如果窃取不到任务，那么当前的P会与M解除绑定，P会放入到空闲的P队列中，解除绑定的M会进入休眠。

![image-20220831150802448](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/image-20220831150802448.png)

#### 获取本地运行队列runq

当qunhead!=runtail，就表明当前的本地运行队列有可用的G。在这里的访问需要加锁。

#### 获取schet.runq的G

全局运行队列的数据结构是一根链表。由于每个P都共享的全局运行队列，因此为了保证公平，先根据P的数量平分全局运行队列，同时转移的G的数量不能超过局部运行队列容量的一半（当前是256/2=128个）。再通过循环调用runqput将全局队列的G放入到P的局部运行队列。如果runq已经满了，那么调度器会将runq的一半放入到全局运行队列，这保证了当程序中以后很多协程时，每个协程都有执行的机会。

#### 获取准备就绪的网络协程

当p.runq和schet.runq都找不到可用协程时，调度器会寻找当前是否有已经准备好运行的网络协程。runtime.netpoll函数获取当前可运行的协程列表，返回第一个协程并通过injectglist函数将其余协程放入全局运行队列。

```go
	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}
```

#### 协程窃取

所有的P都存储在全局的allp []*p中。为了既保证随机性，又保证allp数组中的每个P都能被依次遍历，有了fastrand函数。

假设一共有8个P，第1步：fastrand函数新选择一个随机数对8进行取模，假设算法选择了6。第2步：找到一个比8小且与8互质的数。[1,3,5,7]。代码中取的是coprimes[6%4]=coprimes[2]=5。计算过程为：

```go
(6+5)%8=3
(3+5)%8=0 (0+5)%8=5 (5+5)%8=2 (2+5)%8=7 (7+5)%8=4 (4+5)%8=1
(1+5)%86
```

所以这种数学特性既保证随机性，又保证allp数组中的每个P都能被依次遍历。找到了要窃取的P之后就是将要窃取的P的本地运行队列中G个数的一半放入自己的运行队列中。

```go
// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
    ...
	// Spinning Ms: steal work from other Ps.
	//
	// Limit the number of spinning Ms to half the number of busy Ps.
	// This is necessary to prevent excessive CPU consumption when
	// GOMAXPROCS>>1 but the program parallelism is low.
	procs := uint32(gomaxprocs)
	if _g_.m.spinning || 2*atomic.Load(&sched.nmspinning) < procs-atomic.Load(&sched.npidle) {
		if !_g_.m.spinning {
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		now = tnow
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}
    ...
}

// stealWork attempts to steal a runnable goroutine or timer from any P.
//
// If newWork is true, new work may have been readied.
//
// If now is not 0 it is the current time. stealWork returns the passed time or
// the current time if now was passed as 0.
func stealWork(now int64) (gp *g, inheritTime bool, rnow, pollUntil int64, newWork bool) {
	pp := getg().m.p.ptr()

	ranTimer := false

	const stealTries = 4
	for i := 0; i < stealTries; i++ {
		stealTimersOrRunNextG := i == stealTries-1

		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				// GC work may be available.
				return nil, false, now, pollUntil, true
			}
			p2 := allp[enum.position()]
			if pp == p2 {
				continue
			}

			// Steal timers from p2. This call to checkTimers is the only place
			// where we might hold a lock on a different P's timers. We do this
			// once on the last pass before checking runnext because stealing
			// from the other P's runnext should be the last resort, so if there
			// are timers to steal do that first.
			//
			// We only check timers on one of the stealing iterations because
			// the time stored in now doesn't change in this loop and checking
			// the timers for each P more than once with the same value of now
			// is probably a waste of time.
			//
			// timerpMask tells us whether the P may have timers at all. If it
			// can't, no need to check at all.
			if stealTimersOrRunNextG && timerpMask.read(enum.position()) {
				tnow, w, ran := checkTimers(p2, now)
				now = tnow
				if w != 0 && (pollUntil == 0 || w < pollUntil) {
					pollUntil = w
				}
				if ran {
					// Running the timers may have
					// made an arbitrary number of G's
					// ready and added them to this P's
					// local run queue. That invalidates
					// the assumption of runqsteal
					// that it always has room to add
					// stolen G's. So check now if there
					// is a local G to run.
					if gp, inheritTime := runqget(pp); gp != nil {
						return gp, inheritTime, now, pollUntil, ranTimer
					}
					ranTimer = true
				}
			}

			// Don't bother to attempt to steal if p2 is idle.
			if !idlepMask.read(enum.position()) {
				if gp := runqsteal(pp, p2, stealTimersOrRunNextG); gp != nil {
					return gp, false, now, pollUntil, ranTimer
				}
			}
		}
	}

	// No goroutines found to steal. Regardless, running a timer may have
	// made some goroutine ready that we missed. Indicate the next timer to
	// wait for.
	return nil, false, now, pollUntil, ranTimer
}
```

### 调度时机

调度时机根据调度方式的不同可以分为主动、被动和抢占调度三种。

#### 主动调度

协程可以主动让渡自己的执行权利。通过runtime.Gosched函数实现。原理就是先从当前协程切换到g0，取消G和M的绑定关系，将G放入全局运行队列，并调用schedule函数进行新一轮调度。

#### 被动调度

被动调度指协程在休眠、channel通道阻塞、网络I/O阻塞、执行垃圾回收而暂停时，被动让渡自己执行权利的过程。被动调度具有重大意义，可以保证最大化利用CPU的资源。被动调度也需要先从当前协程切换到g0，更新协程的状态并解绑与M的关系，重新调度，但是被动调度不会将G放入全局队列，当前G的状态也不是`_Grunnable`而是`_Gwaitting`。如果当前协程需要被唤醒，那么会先将协程的状态从`_Gwaitting`转换为`_Grunnable`，并添加到当前P的局部运行队列中。

#### 抢占调度

Go语言在初始化时会启动一个特殊的线程来执行系统监控任务。系统监控在一个独立的M上运行，不用绑定逻辑处理器P，系统监控每隔10ms会检测是否有准备就绪的网络协程，并放置到全局队列中。系统监控服务会判断当前协程是否运行时间过长，或者处于系统调用阶段，如果是，则会抢占当前G的执行。

##### 执行时间过长的抢占

Go在1.14之后引入了信号强制抢占的机制。Go语言借助用户态在信号处理时完成协程的上下文切换的操作，需要借助进程对特定的信号进行处理。在抢占时，调度器通过向线程发送sigPreempt信号，触发信号处理。在遇到sigPreempt抢占信号时，触发运行时的异步抢占机制。

##### 系统调用的抢占

发生系统调用时有三种情况需要抢占调度：

* 当前局部运行队列中有等待运行的G。这种情况下，抢占调度只是为了让局部运行队列中的协程有运行的机会，其一般是当前P私有的。

* 当前没有空闲的P和自旋的M。如果有空闲的P和自旋的M，说明当前比较空闲，那么释放当前的P也没有太大意义。

* 当前系统调用的时间已经超过了10ms，这时需要立即抢占。

系统调用时的抢占原理主要是通过将P的状态转化为_Pidle。目的是让M接管P的执行，主要逻辑位于handoffp函数中，该函数需要判断是否需要找到一个新的M来接管当前的P。以下情况需要启动一个M来接管：

* 本地运行队列有等待运行的G

* 需要处理一些垃圾回收的后台任务

* 所有其他P都在运行G，并且没有自旋的M

* 全局运行队列不为空

* 需要处理网络socket读写等事件

当这些条件都不满足时，才会将当前的P放入空闲队列中。寻找可用的M时，需要现在M的空闲列表查找，如果没有，则向操作系统申请一个新的M。

执行系统调用之前，运行时调用了reentersyscall函数，保存当前G的执行环境，并解除P与M之间的绑定，将P放置到oldp中。接解除绑定是为了系统调用返回后，当前线程能够绑定不同的P，但是会优先选择oldp。工作线程的P被抢占，系统调用的工作线程从内核返回后，被阻塞的协程继续执行，调用exitsyscall函数以便协程重新执行。exitsyscalll函数会尝试绑定oldp，当P不可用，加锁从全局空闲队列寻找空闲的P。如果空闲队列没有空闲的P，则会将当前的G放入全局运行队列，当前工作线程M进入休眠状态。

## 16. 通道和协程通信

### 通道

普通的通道类型可以转换为单方向的通道，反之不可以。

#### 初始化

chan作为Go语言中的类型，最基本的声明方式如下：

```go
var name chan T
```

name代表通道的名称，chan T代表通道的类型，T代表通道中的元素类型。通道的表现形式有三种：chan T、chan <- T、 -> chan T。不带箭头的可读可写，带箭头的类型限制了通道的读写。要对通道进行操作，需要使用make操作符分配通道空间。

#### 写入

`c <- 5`对于无缓冲通道，能够向通道写入数据的前提是必须有另一个协程在读取通道。否欧泽当前通道会陷入休眠状态，知道能够向通道成功写入数据。无缓冲通道的读与写应该在不同协程当中。

#### 读取

`<- c`可以读取通道的数据。和写入数据一样，如果不能直接读取通道的数据，那么当前的读取协程会陷入阻塞，知道有协程写入通道为止。读取通道可以有两个返回值，第一个返回值返回通道的数据，第二个返回bool类型。`data,ok:= <- c`。

#### 关闭

通道可以使用内置的close函数关闭。关闭的通道可以读取，但是不能写入。

* 对一个关闭的通道再发送值就会导致panic。

* 对一个关闭的通道进行接收会一直获取值直到通道为空。

* 对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。

* 关闭一个已经关闭的通道会导致panic。

### select多路复用

select通常和通道结合使用，他的每一个case都必须对应通道的读写操作。当多个通道同时准备好执行读写操作，select的执行具有一定的随机性，case是随机选取的。如果select中没有任何通道准备好，当前select所在的协程将会阻塞直到有一个case的通道准备好为止。

### channel的原理

```go
type hchan struct {
	qcount   uint           // total data in the queue 队列中的总元素个数
	dataqsiz uint           // size of the circular queue 环形队列大小，即可存放元素的个数
	buf      unsafe.Pointer // points to an array of dataqsiz elements  环形队列指针
	elemsize uint16 //每个元素的大小
	closed   uint32 //标识关闭状态
	elemtype *_type // element type  元素类型
	sendx    uint   // send index 发送索引，元素写入时存放到队列中的位置
	recvx    uint   // receive index 接收索引，元素从队列的该位置读出
	recvq    waitq  // list of recv waiters  等待读消息的goroutine队列
	sendq    waitq  // list of send waiters 等待写消息的goroutine队列

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex //互斥锁，chan不允许并发读写
}
```

**读写流程**

**向 channel 写数据:**

* 若等待接收队列 recvq 不为空，则缓冲区中无数据或无缓冲区，将直接从 recvq 取出 G ，并把数据写入，最后把该 G 唤醒，结束发送过程。

* 若缓冲区中有空余位置，则将数据写入缓冲区，结束发送过程。

* 若缓冲区中没有空余位置，则将发送数据写入 G，将当前 G 加入 sendq 链表末尾，进入睡眠，等待被读 goroutine 唤醒。

**从 channel 读数据**

* 若等待发送队列 sendq 不为空，且没有缓冲区，直接从 sendq 中取出 G ，把 G 中数据读出，最后把 G 唤醒，结束读取过程。
* 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程。

* 如果当前通道无缓冲区或者缓冲区是空的，

* 将当前G 加入 recvq 链表末尾，进入睡眠，等待被写 goroutine 唤醒。

### select原理

select中的每个case在运行时都是一个scase结构体，存放了通道和通道中的元素类型等信息。

```go
// Select case descriptor.
// Known to compiler.
// Changes here must also be made in src/cmd/compile/internal/walk/select.go's scasetype.
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}

//依次解锁
func sellock(scases []scase, lockorder []uint16) {
	var c *hchan
	for _, o := range lockorder {
		c0 := scases[o].c
		if c0 != c {
			c = c0
			lock(&c.lock)
		}
	}
}

//依次加锁
func selunlock(scases []scase, lockorder []uint16) {
	// We must be very careful here to not touch sel after we have unlocked
	// the last lock, because sel can be freed right after the last unlock.
	// Consider the following situation.
	// First M calls runtime·park() in runtime·selectgo() passing the sel.
	// Once runtime·park() has unlocked the last lock, another M makes
	// the G that calls select runnable again and schedules it for execution.
	// When the G runs on another M, it locks all the locks and frees sel.
	// Now if the first M touches sel, it will access freed memory.
	for i := len(lockorder) - 1; i >= 0; i-- {
		c := scases[lockorder[i]].c
		if i > 0 && c == scases[lockorder[i-1]].c {
			continue // will unlock it on the next iteration
		}
		unlock(&c.lock)
	}
}

// selectgo implements the select statement.
//
// cas0 points to an array of type [ncases]scase, and order0 points to
// an array of type [2*ncases]uint16 where ncases must be <= 65536.
// Both reside on the goroutine's stack (regardless of any escaping in
// selectgo).
//
// For race detector builds, pc0 points to an array of type
// [ncases]uintptr (also on the stack); for other builds, it's set to
// nil.
//
// selectgo returns the index of the chosen scase, which matches the
// ordinal position of its respective select{recv,send,default} call.
// Also, if the chosen scase was a receive operation, it reports whether
// a value was received.
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
	...
	// NOTE: In order to maintain a lean stack size, the number of scases
	// is capped at 65536.
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	ncases := nsends + nrecvs
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]
	// NOTE: pollorder/lockorder's underlying array was not zero-initialized by compiler.

	// Even when raceenabled is true, there might be select
	// statements in packages compiled without -race (e.g.,
	// ensureSigM in runtime/signal_unix.go).
	var pcs []uintptr
	if raceenabled && pc0 != nil {
		pc1 := (*[1 << 16]uintptr)(unsafe.Pointer(pc0))
		pcs = pc1[:ncases:ncases]
	}
	casePC := func(casi int) uintptr {
		if pcs == nil {
			return 0
		}
		return pcs[casi]
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	// generate permuted order
	norder := 0
	for i := range scases {
		cas := &scases[i]

		// Omit cases without channels from the poll and lock orders.
		if cas.c == nil {
			cas.elem = nil // allow GC
			continue
		}

		j := fastrandn(uint32(norder + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]
	
    //堆排序操作
	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	for i := range lockorder {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}
	...

	// lock all the channels involved in the select
	sellock(scases, lockorder)
	...
}

// 伪代码
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    //1. 锁定scase语句中所有的channel
    //2. 按照随机顺序检测scase中的channel是否ready
    //   2.1 如果case可读，则读取channel中数据，解锁所有的channel，然后返回(case index, true)
    //   2.2 如果case可写，则将数据写入channel，解锁所有的channel，然后返回(case index, false)
    //   2.3 所有case都未ready，则解锁所有的channel，然后返回（default index, false）
    //3. 所有case都未ready，且没有default语句
    //   3.1 将当前协程加入到所有channel的等待队列
    //   3.2 当将协程转入阻塞，等待被唤醒
    //4. 唤醒后返回channel对应的case index
    //   4.1 如果是读操作，解锁所有的channel，然后返回(case index, true)
    //   4.2 如果是写操作，解锁所有的channel，然后返回(case index, false)
}
```

selectgo函数当中，pollorder代表乱序后的scase序列，轮询需要乱序保证公平性。lockorder是按照大小对通道地址排序的算法，对所有的scase按照其通道在堆区的地址大小，使用了大根堆排序算法进行排序，selectgo会按照lockorder序列依次加锁，按照地址排序的目的是为了避免多个协程并发加锁带来的死锁问题。

* order0 为一个两倍 cas0 数组长度的 buffer，保存 scase 随机序列 pollorder 和 scase 中 channel 地址序列 lockorder

* pollorder：每次selectgo执行都会把scase序列打乱，以达到随机检测case的目的

* lockorder：所有case语句中channel序列，以达到去重防止对channel加锁时重复加锁的目的

## 17. 并发



## 18. 内存分配管理

## 19. 垃圾回收（GC）初探

## 20. 深入垃圾回收全流程

## 21. 调试：特征分析和事件追踪

