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

一旦元素个数超过 1024 个元素，那么增长因子就变成 1.25 ，即每 次增加原来容量的四分之一。

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

![image-20200919191800912](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202207251349167.png)

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

`c <- 5`对于无缓冲通道，能够向通道写入数据的前提是必须有另一个协程在读取通道。否则当前通道会陷入休眠状态，知道能够向通道成功写入数据。无缓冲通道的读与写应该在不同协程当中。

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

### context

为了能够优雅的管理协程的退出，特别是多个协程甚至网络服务之间的退出，Go引入了context包。

#### context使用

```go

// A Context carries a deadline, a cancellation signal, and other values across
// API boundaries.
//
// Context's methods may be called by multiple goroutines simultaneously.
type Context interface {
	// Deadline returns the time when work done on behalf of this context
	// should be canceled. Deadline returns ok==false when no deadline is
	// set. Successive calls to Deadline return the same results.
	Deadline() (deadline time.Time, ok bool)

	// Done returns a channel that's closed when work done on behalf of this
	// context should be canceled. Done may return nil if this context can
	// never be canceled. Successive calls to Done return the same value.
	// The close of the Done channel may happen asynchronously,
	// after the cancel function returns.
	//
	// WithCancel arranges for Done to be closed when cancel is called;
	// WithDeadline arranges for Done to be closed when the deadline
	// expires; WithTimeout arranges for Done to be closed when the timeout
	// elapses.
	//
	// Done is provided for use in select statements:
	//
	//  // Stream generates values with DoSomething and sends them to out
	//  // until DoSomething returns an error or ctx.Done is closed.
	//  func Stream(ctx context.Context, out chan<- Value) error {
	//  	for {
	//  		v, err := DoSomething(ctx)
	//  		if err != nil {
	//  			return err
	//  		}
	//  		select {
	//  		case <-ctx.Done():
	//  			return ctx.Err()
	//  		case out <- v:
	//  		}
	//  	}
	//  }
	//
	// See https://blog.golang.org/pipelines for more examples of how to use
	// a Done channel for cancellation.
	Done() <-chan struct{}

	// If Done is not yet closed, Err returns nil.
	// If Done is closed, Err returns a non-nil error explaining why:
	// Canceled if the context was canceled
	// or DeadlineExceeded if the context's deadline passed.
	// After Err returns a non-nil error, successive calls to Err return the same error.
	Err() error

	// Value returns the value associated with this context for key, or nil
	// if no value is associated with key. Successive calls to Value with
	// the same key returns the same result.
	//
	// Use context values only for request-scoped data that transits
	// processes and API boundaries, not for passing optional parameters to
	// functions.
	//
	// A key identifies a specific value in a Context. Functions that wish
	// to store values in Context typically allocate a key in a global
	// variable then use that key as the argument to context.WithValue and
	// Context.Value. A key can be any type that supports equality;
	// packages should define keys as an unexported type to avoid
	// collisions.
	//
	// Packages that define a Context key should provide type-safe accessors
	// for the values stored using that key:
	//
	// 	// Package user defines a User type that's stored in Contexts.
	// 	package user
	//
	// 	import "context"
	//
	// 	// User is the type of value stored in the Contexts.
	// 	type User struct {...}
	//
	// 	// key is an unexported type for keys defined in this package.
	// 	// This prevents collisions with keys defined in other packages.
	// 	type key int
	//
	// 	// userKey is the key for user.User values in Contexts. It is
	// 	// unexported; clients use user.NewContext and user.FromContext
	// 	// instead of using this key directly.
	// 	var userKey key
	//
	// 	// NewContext returns a new Context that carries value u.
	// 	func NewContext(ctx context.Context, u *User) context.Context {
	// 		return context.WithValue(ctx, userKey, u)
	// 	}
	//
	// 	// FromContext returns the User value stored in ctx, if any.
	// 	func FromContext(ctx context.Context) (*User, bool) {
	// 		u, ok := ctx.Value(userKey).(*User)
	// 		return u, ok
	// 	}
	Value(key interface{}) interface{}
}
```

Deadline方法的第一个返回值表示还有多久到期，第二个返回值表示是否到期。Done是使用最频繁的方法，其返回一个通道，一般的做法是监听该通道的信号，如果收到信号则表示通道已经关闭，需要执行退出。如果通道已经关闭，则Err()方法返回退出的原因。value方法返回指定key对应的value，这是context携带的值。

context携带值比较少见，一般在跨程序的API中使用，并且该值的作用域在结束时终结。key必须是访问安全的，因为可能有多个协程同时访问他。Value主要用于安全凭证、分布式跟踪ID、操作优先级、退出信号与到期时间等场景。使用value方法时要慎重。

#### context退出与传递

context.Background函数或者context.TODO函数会返回最简单的context实现。

```go
// WithCancel returns a copy of parent with a new Done channel. The returned
// context's Done channel is closed when the returned cancel function is called
// or when the parent context's Done channel is closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {}

// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete:
//
// 	func slowOperationWithTimeout(ctx context.Context) (Result, error) {
// 		ctx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
// 		defer cancel()  // releases resources if slowOperation completes before timeout elapses
// 		return slowOperation(ctx)
// 	}
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {}

// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {}

// WithValue returns a copy of parent in which the value associated with key is
// val.
//
// Use context Values only for request-scoped data that transits processes and
// APIs, not for passing optional parameters to functions.
//
// The provided key must be comparable and should not be of type
// string or any other built-in type to avoid collisions between
// packages using context. Users of WithValue should define their own
// types for keys. To avoid allocating when assigning to an
// interface{}, context keys often have concrete type
// struct{}. Alternatively, exported context key variables' static
// type should be a pointer or interface.
func WithValue(parent Context, key, val interface{}) Context {}
```

* WithCancel函数返回一个子context并且有cancel退出方法。子context调用cancel方法或者父context退出时都会退出。

* WithTimeout函数指定超时时间，当超时发生后，子context会退出。

* WithDeadline和WithTimeout处理方法相似。参数指的是最后到期的时间。

* WithValue函数返回带key-value的子context。

子context的退出不会影响父context。

#### context原理

context很大程度上利用了通道在close时会通知所有监听它的协程这一特性。

* context.Background函数和context.TODO函数是相似的。它们都返回一个emptyCtx。作为最初始的根对象。

* WithCancel或WithTimeout函数会产生一个子context结构cancelCtx，并保留了父context的信息。children字段保存当前context之后派生的子context的信息。每个context都会有一个新的通道，这保证了子context的退出不会影响父context。

  ```go
  // A cancelCtx can be canceled. When canceled, it also cancels any children
  // that implement canceler.
  type cancelCtx struct {
  	Context
  
  	mu       sync.Mutex            // protects following fields
  	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
  	children map[canceler]struct{} // set to nil by the first cancel call
  	err      error                 // set to non-nil by the first cancel call
  }
  ```

  WithTimeout函数最终会调用WithDeadline函数的方法。

WithDeadline函数源码解析:

```go
// WithDeadline returns a copy of the parent context with the deadline adjusted
// to be no later than d. If the parent's deadline is already earlier than d,
// WithDeadline(parent, d) is semantically equivalent to parent. The returned
// context's Done channel is closed when the deadline expires, when the returned
// cancel function is called, or when the parent context's Done channel is
// closed, whichever happens first.
//
// Canceling this context releases resources associated with it, so code should
// call cancel as soon as the operations running in this Context complete.
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    //判断父context是否比当前设置的超时参数d先退出，如果是那么子context随着父context退出而退出。
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
    //创建一个新的context，初始化通道
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
    //将子context加入到父协程的children哈希表，并开启一个定时器
	propagateCancel(parent, c)
    //定时器是否到期
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
    //当一切结束后，还需要从父context哈希表中移除该context
	return c, func() { c.cancel(true, Canceled) }
}

// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}

//cancel方法会关闭自身的通道，并且遍历当前children哈希表，调用当前所有子context的退出函数
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

### 数据争用检查

数据争用在Go语言中指两个协程同时访问相同的内存空间，并且至少有一个写操作的情况。

#### race工具

race可以使用在多个Go指令当中，当检测器在程序中找到数据争用时，将打印报告。该报告包含发生race冲突的协程栈，以及此时正在运行的协程栈。

#### race原理

race工具借助了google为了应对内部大量服务器端C++代码的数据争用问题而开发的ThreadSanitizer工具，Go语言内部通过CGO的形式进行调用。

矢量时钟技术（vector clock）用来观察事件之间happened-before的顺序，用于检测和确定分布式系统中的事件的因果关系，也可以用于数据争用的探测。在Go程序中，有n个协程就会有对应的n个逻辑时钟，而矢量时钟是所有这些逻辑时钟组成的数组。

在Go语言中，每个协程在创建之初都会初始化矢量时钟，并且在读取或者写入事件时修改自身的逻辑时钟。

触发race事件主要有两种方式，一是在Go语言运行时中大量（超过100处）注入触发事件。二是依靠编译器加上race指令。

### 锁

#### 原子锁

针对基本数据类型我们还可以使用原子操作来保证并发安全，因为原子操作是Go语言提供的方法它在用户态就可以完成，因此性能比加锁操作更好。Go语言中原子操作由内置的标准库`sync/atomic`提供。

| 方法                                                         | 解释           |
| ------------------------------------------------------------ | -------------- |
| func LoadInt32(addr `*int32`) (val int32)<br/>func LoadInt64(addr `*int64`) (val int64)<br/>func LoadUint32(addr`*uint32`) (val uint32)<br/>func LoadUint64(addr`*uint64`) (val uint64)<br/>func LoadUintptr(addr`*uintptr`) (val uintptr)<br/>func LoadPointer(addr`*unsafe.Pointer`) (val unsafe.Pointer) | 读取操作       |
| func StoreInt32(addr `*int32`, val int32)<br/>func StoreInt64(addr `*int64`, val int64)<br/>func StoreUint32(addr `*uint32`, val uint32)<br/>func StoreUint64(addr `*uint64`, val uint64)<br/>func StoreUintptr(addr `*uintptr`, val uintptr)<br/>func StorePointer(addr `*unsafe.Pointer`, val unsafe.Pointer) | 写入操作       |
| func AddInt32(addr `*int32`, delta int32) (new int32)<br/>func AddInt64(addr `*int64`, delta int64) (new int64)<br/>func AddUint32(addr `*uint32`, delta uint32) (new uint32)<br/>func AddUint64(addr `*uint64`, delta uint64) (new uint64)<br/>func AddUintptr(addr `*uintptr`, delta uintptr) (new uintptr) | 修改操作       |
| func SwapInt32(addr `*int32`, new int32) (old int32)<br/>func SwapInt64(addr `*int64`, new int64) (old int64)<br/>func SwapUint32(addr `*uint32`, new uint32) (old uint32)<br/>func SwapUint64(addr `*uint64`, new uint64) (old uint64)<br/>func SwapUintptr(addr `*uintptr`, new uintptr) (old uintptr)<br/>func SwapPointer(addr `*unsafe.Pointer`, new unsafe.Pointer) (old unsafe.Pointer) | 交换操作       |
| func CompareAndSwapInt32(addr `*int32`, old, new int32) (swapped bool)<br/>func CompareAndSwapInt64(addr `*int64`, old, new int64) (swapped bool)<br/>func CompareAndSwapUint32(addr `*uint32`, old, new uint32) (swapped bool)<br/>func CompareAndSwapUint64(addr `*uint64`, old, new uint64) (swapped bool)<br/>func CompareAndSwapUintptr(addr `*uintptr`, old, new uintptr) (swapped bool)<br/>func CompareAndSwapPointer(addr `*unsafe.Pointer`, old, new unsafe.Pointer) (swapped bool) | 比较并交换操作 |

原子锁是底层最基础的同步保证，通过原子操作可以构建起许多同步原语，例如自旋锁、信号量、互斥锁等。

#### 互斥锁

sync.Mutex构建起了互斥锁。互斥锁是一种混合锁，其实现方式包含了自旋锁，同时参考了操作系统锁的实现。Mutex包含了锁的状态state及信号量sema。

```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32 
	sema  uint32
}
```

state通过位图的形式存储了当前锁的状态，包含了锁是否为锁定状态、正在等待被锁唤醒的协程数量、两个和饥饿模式有关的标志。为了解决某一个协程可能长时间无法获取锁的问题，Go采用了饥饿模式，在饥饿模式下，unlock会唤醒最先申请的协程。sema是互斥锁中实现的信号量。

##### 加锁

互斥锁的第1个阶段是使用原子操作快速抢占锁。

```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

lockSlow方法在正常情况下回自旋尝试抢占锁一段时间，而不会立即进入休眠状态。runtime_canSpin函数会判断当前是否能进入自旋状态。
在以下四种情况，自旋状态会立即终止：

* 程序在单核CPU上运行

* 逻辑处理器P小于等于1

* 当前协程所在的逻辑处理器P的本地队列上有其他协程待运行

* 自旋次数超过设定的阈值

当长时间没有获取到锁，就进入互斥锁的第2个阶段，使用信号量进行同步。如果加锁操作进入信号量同步阶段，信号量计数减1，解锁操作则加1。当信号量计数值大于0时，意味着有其他协程执行了解锁操作，这时加锁协程可以直接退出。当信号量计数值等于0时，意味着当前加锁协程需要陷入休眠状态。

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64
	starving := false
	awoke := false
	iter := 0
	old := m.state
	for {
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```

互斥锁的第3个阶段，所有锁的信息都会根据锁的地址存储在全局semtable哈希表中。锁被放到全局的等待队列中并等待被唤醒，顺序为从前到后，遵循先入先出的准则，这样保证了公平性。当长时间无法获取锁时，当前的互斥锁会进入饥饿模式。在饥饿模式下，为了保证公平性，新申请锁的协程不会进入自旋状态，而是直接放入等待队列中。放入等待队列中的协程会切换自己的执行状态，让渡执行权力并进行新的调度循环。

##### 解锁

互斥锁的释放和锁定相对应。

如果当前锁处于普通的锁定状态，Unlock方法在修改mutexLocked状态后立即退出。

```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}
```
判断锁是否重复释放。锁不能重复释放，否则会在运行时报错。

```go
if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
```

如果锁处于饥饿状态，则进入信号量同步阶段，到全局哈希表中寻找当前锁的等待队列，以先进先出的顺序唤醒指定协程。

如果锁当前未处于饥饿状态且当前mutexWoken已设置，则表明有其他申请锁的协程准备从正常状态退出，这时锁释放后不用去当前锁的等待队列唤醒其他协程，而是直接退出。如果唤醒了等待队列中的协程，则将唤醒的协程放入当前协程所在逻辑处理器P的runnext字段中，存储runnext字段中的协程会被优先调度。如果在饥饿模式下，则当前协程会让渡自己的执行权利，让被唤醒的协程直接运行，这是通过将`runtime_Semrelease`函数的第2个参数设置为true实现的。

```go
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
            //当前没有等待被唤醒的协程或者mutexWoken已设置
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
            //唤醒等待中的协程
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
        //在饥饿模式下唤醒协程，并立即执行
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

#### 读写锁

读写锁适用于多读少写的场景。通过两种锁来实现：一种为读锁，一种为写锁。当进行读取操作时，需要加读锁，而进行写入操作时，需要加写锁。多个协程可以同时获取读锁并执行。如果此时有协程申请了写锁，那么该写锁会等待所有读锁被释放后才能获取写锁继续执行。如果当前协程申请读锁时已经存在写锁，那么读锁会等待写锁释放后在获取读锁继续执行。

读锁必须能观察到上一次写锁写入的值，写锁要等待之前的读锁释放才能写入。读可以有多个协程同时读，但是写只能有一个协程同时写。

##### 读写锁原理

```go
// There is a modified copy of this file in runtime/rwmutex.go.
// If you make any changes here, see if you should make them there.

// A RWMutex is a reader/writer mutual exclusion lock.
// The lock can be held by an arbitrary number of readers or a single writer.
// The zero value for a RWMutex is an unlocked mutex.
//
// A RWMutex must not be copied after first use.
//
// If a goroutine holds a RWMutex for reading and another goroutine might
// call Lock, no goroutine should expect to be able to acquire a read lock
// until the initial read lock is released. In particular, this prohibits
// recursive read locking. This is to ensure that the lock eventually becomes
// available; a blocked Lock call excludes new readers from acquiring the
// lock.
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
```

* 读锁

  读取操作先通过原子操作将readerCount加1，如果readerCount≥0就直接返回，所以如果只是获取读取锁的操作，那么其成本只有一个原子操作。如果readerCount＜0，说明当前有写锁，当前协程将借助信号量陷入等待状态，如果获取到信号量则立即退出，没有获取到信号量时的逻辑与互斥锁逻辑类似。

  ```go
  // Happens-before relationships are indicated to the race detector via:
  // - Unlock  -> Lock:  readerSem
  // - Unlock  -> RLock: readerSem
  // - RUnlock -> Lock:  writerSem
  //
  // The methods below temporarily disable handling of race synchronization
  // events in order to provide the more precise model above to the race
  // detector.
  //
  // For example, atomic.AddInt32 in RLock should not appear to provide
  // acquire-release semantics, which would incorrectly synchronize racing
  // readers, thus potentially missing races.
  
  // RLock locks rw for reading.
  //
  // It should not be used for recursive read locking; a blocked Lock
  // call excludes new readers from acquiring the lock. See the
  // documentation on the RWMutex type.
  func (rw *RWMutex) RLock() {
  	if race.Enabled {
  		_ = rw.w.state
  		race.Disable()
  	}
  	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
  		// A writer is pending, wait for it.
  		runtime_SemacquireMutex(&rw.readerSem, false, 0)
  	}
  	if race.Enabled {
  		race.Enable()
  		race.Acquire(unsafe.Pointer(&rw.readerSem))
  	}
  }
  ```

  读锁解锁时，如果当前没有写锁，则其成本只有一个原子操作并直接退出。

  ```go
  // RUnlock undoes a single RLock call;
  // it does not affect other simultaneous readers.
  // It is a run-time error if rw is not locked for reading
  // on entry to RUnlock.
  func (rw *RWMutex) RUnlock() {
  	if race.Enabled {
  		_ = rw.w.state
  		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
  		race.Disable()
  	}
  	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
  		// Outlined slow-path to allow the fast-path to be inlined
  		rw.rUnlockSlow(r)
  	}
  	if race.Enabled {
  		race.Enable()
  	}
  }
  ```

  如果当前有写锁在等待，则调用rUnlockSlow判断当前是否最后一个被释放的读锁，如果是则需要增加信号量并唤醒写锁。

  ```go
  func (rw *RWMutex) rUnlockSlow(r int32) {
  	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
  		race.Enable()
  		throw("sync: RUnlock of unlocked RWMutex")
  	}
  	// A writer is pending.
  	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
  		// The last reader unblocks the writer.
  		runtime_Semrelease(&rw.writerSem, false, 1)
  	}
  }
  ```

* 写锁

  读写锁申请写锁时要调用Lock方法，必须先获取互斥锁，因为它复用了互斥锁的功能。接着readerCount减去rwmutexMaxReaders阻止后续的读操作。如果当前有其他的G持有互斥锁的读锁，那么当前协程会加入全局等待队列并进入休眠状态，当最后一个读锁被释放时，会唤醒该协程。

  ```go
  // Lock locks rw for writing.
  // If the lock is already locked for reading or writing,
  // Lock blocks until the lock is available.
  func (rw *RWMutex) Lock() {
  	if race.Enabled {
  		_ = rw.w.state
  		race.Disable()
  	}
  	// First, resolve competition with other writers.
  	rw.w.Lock()
  	// Announce to readers there is a pending writer.
  	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
  	// Wait for active readers.
  	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
  		runtime_SemacquireMutex(&rw.writerSem, false, 0)
  	}
  	if race.Enabled {
  		race.Enable()
  		race.Acquire(unsafe.Pointer(&rw.readerSem))
  		race.Acquire(unsafe.Pointer(&rw.writerSem))
  	}
  }
  ```

  解锁时，调用Unlock方法。将readerCount加上rwmutexMaxReaders，表示不会堵塞后续的读锁，依次唤醒所有等待中的读锁。当所有读锁唤醒完毕后会释放互斥锁。

  ```go
  // Unlock unlocks rw for writing. It is a run-time error if rw is
  // not locked for writing on entry to Unlock.
  //
  // As with Mutexes, a locked RWMutex is not associated with a particular
  // goroutine. One goroutine may RLock (Lock) a RWMutex and then
  // arrange for another goroutine to RUnlock (Unlock) it.
  func (rw *RWMutex) Unlock() {
  	if race.Enabled {
  		_ = rw.w.state
  		race.Release(unsafe.Pointer(&rw.readerSem))
  		race.Disable()
  	}
  
  	// Announce to readers there is no active writer.
  	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
  	if r >= rwmutexMaxReaders {
  		race.Enable()
  		throw("sync: Unlock of unlocked RWMutex")
  	}
  	// Unblock blocked readers, if any.
  	for i := 0; i < int(r); i++ {
  		runtime_Semrelease(&rw.readerSem, false, 0)
  	}
  	// Allow other writers to proceed.
  	rw.w.Unlock()
  	if race.Enabled {
  		race.Enable()
  	}
  }
  ```

  所以读写锁在写操作时的性能和互斥锁类似，但是在只有读操作时效率会高很多，因为读锁可以被多个协程获取。

## 18. 内存分配管理

合理安排、组织、管理、释放内存是高效程序的基础。Go语言运行时依靠细微的对象切割、极致的多级缓存、精准的位图管理实现了对内存的精细化管理。

### GO内存分配

#### span和元素

Go语言将内存分成了大大小小67个级别的span，其中0代表特殊的大对象，其个数是不确定的。当具体的对象需要分配内存时，并不是直接分配span，而是分配不同级别的span中的元素。span的级别不是以每个span的大小为依据的，而是以span中的元素大小为依据。

| span等级 | 元素大小（字节） | span大小（字节） | 元素个数 |
| :------- | ---------------- | ---------------- | -------- |
| 1        | 8                | 8192             | 1024     |
| 2        | 16               | 8192             | 512      |
| 3        | 32               | 8192             | 256      |
| 4        | 48               | 8192             | 170      |
| 5        | 64               | 8192             | 128      |
| ...      | ...              | ...              | ...      |
| 65       | 28678            | 57344            | 2        |
| 66       | 32768            | 32768            | 1        |

#### 三级对象管理

为了方便对span进行管理，加速span对象的访问和分配，Go语言采取了三级对象管理结构，分别为mcache、mcentral、mheap。

Go语言采用了现代TCMalloc内存分配算法的思想，每个逻辑处理器P都存储了一个本地span缓存，成为mcache。如果协程需要内存可以直接从mcache中获取，由于同一时间只有一个协程运行在逻辑处理器P上，所以中间不需要加锁。mcache包含所有大小规格的mspan，但每种规格大小只包含一个。出了class0外，mcache的span都来自于mcentral。

* mcentral是被所有逻辑处理器P共享的。

* mcentral对象收集所有给定规格大小的span。每个mcentral都包含两个mspan的链表：empty msapnList表示没有空闲对象或span已经被mcache缓存的span链表，nonempty mspanList表示有空闲对象的span链表。

做这种区分是为了更快的分配span到mcache中。除了级别0，每个级别的span都会有一个mcentral用于管理span链表。而所有级别的这些mcentral其实都是一个数组，由mheap进行管理。

mheap的作用不只是管理central，大对象也会直接通过mheap进行分配。mheap实现了对虚拟内存线性地址空间的精准管理，建立了span与具体线性地址空间的联系，保存了分配的位图信息，是管理内存的最核心单元。

#### 四级对象内存块管理

根据对象的大小，Go语言将堆内存分成了HeapArea、chunk、span和page4种内存块进行管理。HeapArea内存块最大，其大小和平台相关，在Unix64位操作系统中占据64MB。chunk占据了512KB，span根据级别大小的不同而不同，但必须是page的倍数。1个page占据8KB。不同的内存块用于不同的场景，便于高效地对内存进行管理。

### 对象分配

不同大小的对象会被分配到不同的span中。运行时分配对象的逻辑主要位于mallocgc函数，此函数除了分配内存还会为垃圾回收做一些位图标记工作。

```go
// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    ...
    // In some cases block zeroing can profitably (for latency reduction purposes)
	// be delayed till preemption is possible; isZeroed tracks that state.
	isZeroed := true
    //判断是否为小对象，maxSmallSize当前为32KB
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
            //微小对象分配
			// Tiny allocator.
			//
			// Tiny allocator combines several tiny allocation requests
			// into a single memory block. The resulting memory block
			// is freed when all subobjects are unreachable. The subobjects
			// must be noscan (don't have pointers), this ensures that
			// the amount of potentially wasted memory is bounded.
			//
			// Size of the memory block used for combining (maxTinySize) is tunable.
			// Current setting is 16 bytes, which relates to 2x worst case memory
			// wastage (when all but one subobjects are unreachable).
			// 8 bytes would result in no wastage at all, but provides less
			// opportunities for combining.
			// 32 bytes provides more opportunities for combining,
			// but can lead to 4x worst case wastage.
			// The best case winning is 8x regardless of block size.
			//
			// Objects obtained from tiny allocator must not be freed explicitly.
			// So when an object will be freed explicitly, we ensure that
			// its size >= maxTinySize.
			//
			// SetFinalizer has a special case for objects potentially coming
			// from tiny allocator, it such case it allows to set finalizers
			// for an inner byte of a memory block.
			//
			// The main targets of tiny allocator are small strings and
			// standalone escaping variables. On a json benchmark
			// the allocator reduces number of allocations by ~12% and
			// reduces heap size by ~20%.
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = alignUp(off, 8)
			} else if sys.PtrSize == 4 && size == 12 {
				// Conservatively align 12-byte objects to 8 bytes on 32-bit
				// systems so that objects whose first field is a 64-bit
				// value is aligned to 8 bytes and does not cause a fault on
				// atomic access. See issue 37262.
				// TODO(mknyszek): Remove this workaround if/when issue 36606
				// is resolved.
				off = alignUp(off, 8)
			} else if size&3 == 0 {
				off = alignUp(off, 4)
			} else if size&1 == 0 {
				off = alignUp(off, 2)
			}
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.tinyAllocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			span = c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if !raceenabled && (size < c.tinyoffset || c.tiny == 0) {
				// Note: disabled when race detector is on, see comment near end of this function.
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
            //小对象分配
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[divRoundUp(size, smallSizeDiv)]
			} else {
				sizeclass = size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			span = c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
        //大对象分配
		shouldhelpgc = true
		// For large allocations, keep track of zeroed state so that
		// bulk zeroing can be happen later in a preemptible context.
		span, isZeroed = c.allocLarge(size, needzero && !noscan, noscan)
		span.freeindex = 1
		span.allocCount = 1
		x = unsafe.Pointer(span.base())
		size = span.elemsize
	}
	...

	return x
}
```

##### 微小对象

小于16字节的对象被划分为微小对象。划分微小对象的主要目的是处理绩效的字符串和独立的转义变量。

微小对象会被放入class为2的span中。首先对微小对象按照2、4、8的规则进行字节对齐。例如：字节为1的元素会被分配2字节，字节为7的元素会被分配8字节。

```go
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = alignUp(off, 8)
			} else if sys.PtrSize == 4 && size == 12 {
				// Conservatively align 12-byte objects to 8 bytes on 32-bit
				// systems so that objects whose first field is a 64-bit
				// value is aligned to 8 bytes and does not cause a fault on
				// atomic access. See issue 37262.
				// TODO(mknyszek): Remove this workaround if/when issue 36606
				// is resolved.
				off = alignUp(off, 8)
			} else if size&3 == 0 {
				off = alignUp(off, 4)
			} else if size&1 == 0 {
				off = alignUp(off, 2)
			}
```

查看之前分配的元素中是否有空余的空间，如果可以满足，则返回tiny+offset的地址，意味着当前地址往后的字节都是可以被分配的。分配完成后的offset位置也需要相应增加，为下一次分配做准备。如果当前要分配的元素空间不够，将尝试从mcache中查找span中下一个可用的元素。因此，tiny分配的第一步是尝试利用分配过的前一个元素的空间，达到节约内存的目的。

```go
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.tinyAllocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			span = c.alloc[tinySpanClass]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if !raceenabled && (size < c.tinyoffset || c.tiny == 0) {
				// Note: disabled when race detector is on, see comment near end of this function.
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
```

###### mcache缓存位图

在查找空闲元素空间时，首先要从mcache中照出对应级别的mspan，mspan当中拥有allocCache字段，其作为一个位图，用于标记span中的元素是否被分配。由于allocCache元素为uint64，因此其最多一次缓存64个元素。

```go
// nextFreeFast returns the next free object if one is quickly available.
// Otherwise it returns 0.
func nextFreeFast(s *mspan) gclinkptr {
	theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
	if theBit < 64 {
		result := s.freeindex + uintptr(theBit)
		if result < s.nelems {
			freeidx := result + 1
			if freeidx%64 == 0 && freeidx != s.nelems {
				return 0
			}
			s.allocCache >>= uint(theBit + 1)
			s.freeindex = freeidx
			s.allocCount++
			return gclinkptr(result*s.elemsize + s.base())
		}
	}
	return 0
}

// Ctz64 counts trailing (low-order) zeroes,
// and if all are zero, then 64.
func Ctz64(x uint64) int {
	x &= -x                       // isolate low-order bit
	y := x * deBruijn64ctz >> 58  // extract part of deBruijn sequence
	i := int(deBruijnIdx64ctz[y]) // convert to bit index
	z := int((x - 1) >> 57 & 64)  // adjustment if zero
	return i + z
}
```

allocCache中的最后1bit对应的是span中的第1个元素是否被分配。当bit位为0代表当前对应的span中的元素已经分配，这里的设计是为了sys.Ctz64函数加速计算。因此sys.Ctz64可以直接计算 allocCache 最后面有几个 0 (也就是在第几个 bit 遇到第一个 1) 来使用。

![image-20220915150312719](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202209151503792.png)

有时候，span中元素的个数大于64，因此需要专门有一个字段freeindex标识当前span元素被分配到了哪里。因此，只要从allocCache开始找到哪一位为1即可。假如X位为1，那么X+freeindex为当前span中可用的元素序号。当allocCache中的bit位全部被标记为0后，需要移动freeindex，并更新allocCache，直到span中的元素的末尾为止。

###### mcentral遍历span

如果当前的span中没有可以使用的元素，那么需要从mcentral中加锁查找。在mcentral查找时，会分别遍历mcentral的empty 链表和nonempty 链表两种类型span链表。之所以遍历empty msapnList是因为可能有些span虽然被垃圾回收器标记为空闲了，但是还没来得及清理，这些span在清扫后仍然是可以使用的。

```go
// Allocate a span to use in an mcache.
func (c *mcentral) cacheSpan() *mspan {
	...

	var s *mspan
	sl := newSweepLocker()
	sg := sl.sweepGen

	// Try partial swept spans first.
	if s = c.partialSwept(sg).pop(); s != nil {
		goto havespan
	}

	// Now try partial unswept spans.
	for ; spanBudget >= 0; spanBudget-- {
		s = c.partialUnswept(sg).pop()
		if s == nil {
			break
		}
		if s, ok := sl.tryAcquire(s); ok {
			// We got ownership of the span, so let's sweep it and use it.
			s.sweep(true)
			sl.dispose()
			goto havespan
		}
		// We failed to get ownership of the span, which means it's being or
		// has been swept by an asynchronous sweeper that just couldn't remove it
		// from the unswept list. That sweeper took ownership of the span and
		// responsibility for either freeing it to the heap or putting it on the
		// right swept list. Either way, we should just ignore it (and it's unsafe
		// for us to do anything else).
	}
	// Now try full unswept spans, sweeping them and putting them into the
	// right list if we fail to get a span.
	for ; spanBudget >= 0; spanBudget-- {
		s = c.fullUnswept(sg).pop()
		if s == nil {
			break
		}
		if s, ok := sl.tryAcquire(s); ok {
			// We got ownership of the span, so let's sweep it.
			s.sweep(true)
			// Check if there's any free space.
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				sl.dispose()
				goto havespan
			}
			// Add it to the swept list, because sweeping didn't give us any free space.
			c.fullSwept(sg).push(s.mspan)
		}
		// See comment for partial unswept spans.
	}
    ...
	return s
}
```

 ###### mheap缓存查找

如果在mcentral中找不到可以使用的span，就需要在mheap中查找。

Go1.14之后，每个逻辑处理器P中都维护了一份pageCache。

```go
// pageCache represents a per-p cache of pages the allocator can
// allocate from without a lock. More specifically, it represents
// a pageCachePages*pageSize chunk of memory with 0 or more free
// pages in it.
type pageCache struct {
	base  uintptr // base address of the chunk
	cache uint64  // 64-bit bitmap representing free pages (1 means free)
	scav  uint64  // 64-bit bitmap representing scavenged pages (1 means scavenged)
}
```

mheap会首先查找每个逻辑处理器P中的pageCache字段的cache。cache也是一个位图，每一位都代表了一个page（8KB）。由于cache为uint64，因此一共可以提供512KB的连续虚拟内存。在cache中，1代表未分配的内存，0代表已分配的内存。base代表该虚拟内存的基地址。当需要分配的内存小于512/4=128KB时，需要首先从cache中分配。例如：要分配n pages，就需要查找cache中是否有连续n个为1的位。如果存在，说明在缓存中查找到了合适的内存，用于构建span。

###### mheap基数树查找

如果要分配的page过大或者在逻辑处理器P的cache没有找到可以用的page，就需要对mheap加锁，并在整个mheap管理的虚拟地址空间的位图中查找是否有可用的page。管理线性地址空间的位图结构叫做基数树。

该树中的每个节点都对应一个pallocSum，最底层的叶子节点对应的pallocSum包含一个chunk的信息（512*8KB），除叶子节点外的节点都包含连续8个子节点的内存信息。因此越往上的节点对应的内存越多。

```go
// pallocSum is a packed summary type which packs three numbers: start, max,
// and end into a single 8-byte value. Each of these values are a summary of
// a bitmap and are thus counts, each of which may have a maximum value of
// 2^21 - 1, or all three may be equal to 2^21. The latter case is represented
// by just setting the 64th bit.
type pallocSum uint64

// start extracts the start value from a packed sum.
func (p pallocSum) start() uint {
	if uint64(p)&uint64(1<<63) != 0 {
		return maxPackedValue
	}
	return uint(uint64(p) & (maxPackedValue - 1))
}

// max extracts the max value from a packed sum.
func (p pallocSum) max() uint {
	if uint64(p)&uint64(1<<63) != 0 {
		return maxPackedValue
	}
	return uint((uint64(p) >> logMaxPackedValue) & (maxPackedValue - 1))
}

// end extracts the end value from a packed sum.
func (p pallocSum) end() uint {
	if uint64(p)&uint64(1<<63) != 0 {
		return maxPackedValue
	}
	return uint((uint64(p) >> (2 * logMaxPackedValue)) & (maxPackedValue - 1))
}

// unpack unpacks all three values from the summary.
func (p pallocSum) unpack() (uint, uint, uint) {
	if uint64(p)&uint64(1<<63) != 0 {
		return maxPackedValue, maxPackedValue, maxPackedValue
	}
	return uint(uint64(p) & (maxPackedValue - 1)),
		uint((uint64(p) >> logMaxPackedValue) & (maxPackedValue - 1)),
		uint((uint64(p) >> (2 * logMaxPackedValue)) & (maxPackedValue - 1))
}
```

pallocSum是一个简单的uint64，分为开头(start)、中间(max)、末尾(end)3部分。pallocSum的开头和末尾各占21bit，中间部分占22bit，他们分别包含了这个区域中的连续空闲内存页的信息，包括开头有多少连续内存页，最多有多少连续内存页，末尾有多少连续内存页。对于最顶层的节点，由于其max位为22bit,因此一颗完整的基数树最多代表2<sup>21</sup>pages=16GB内存。

在Go语言中，存储了一个特别的字段searchAddr，用于搜索可用内存的。searchAddr前面的地址一定是已分配过的，因此在查找时，只需要向searchAddr地址的后方查找即可跳过已经查找的节点，减少查找的时间。

在第1次查找中，会从当前searchAddr的chunk块中查找是否有对应大小的连续空间。chunk块的每个page（8KB）都有位图表明其是否已经被分配。

每个chunk都有1个pallocData结构，其中pallocBits管理器分配的位图。pallocBits是uint64，有8字节，由于其每一位对应一个page，因此pallocBits一共对应了64*8=512KB，恰好是一个chunk块的大小。

```go
// pallocData encapsulates pallocBits and a bitmap for
// whether or not a given page is scavenged in a single
// structure. It's effectively a pallocBits with
// additional functionality.
//
// Update the comment on (*pageAlloc).chunks should this
// structure change.
type pallocData struct {
	pallocBits
	scavenged pageBits
}
// pallocBits is a bitmap that tracks page allocations for at most one
// palloc chunk.
//
// The precise representation is an implementation detail, but for the
// sake of documentation, 0s are free pages and 1s are allocated pages.
type pallocBits pageBits

// pageBits is a bitmap representing one bit per page in a palloc chunk.
type pageBits [pallocChunkPages / 64]uint64 //[8]uint64

const (
	// The size of a bitmap chunk, i.e. the amount of bits (that is, pages) to consider
	// in the bitmap at once.
	pallocChunkPages    = 1 << logPallocChunkPages //512
	pallocChunkBytes    = pallocChunkPages * pageSize
	logPallocChunkPages = 9
	logPallocChunkBytes = logPallocChunkPages + pageShift
)
```

而所有的chunk pallocData都在pageAlloc结构中进行管理。

```go
type pageAlloc struct {
	// Radix tree of summaries.
	//
	// Each slice's cap represents the whole memory reservation.
	// Each slice's len reflects the allocator's maximum known
	// mapped heap address for that level.
	//
	// The backing store of each summary level is reserved in init
	// and may or may not be committed in grow (small address spaces
	// may commit all the memory in init).
	//
	// The purpose of keeping len <= cap is to enforce bounds checks
	// on the top end of the slice so that instead of an unknown
	// runtime segmentation fault, we get a much friendlier out-of-bounds
	// error.
	//
	// To iterate over a summary level, use inUse to determine which ranges
	// are currently available. Otherwise one might try to access
	// memory which is only Reserved which may result in a hard fault.
	//
	// We may still get segmentation faults < len since some of that
	// memory may not be committed yet.
	summary [summaryLevels][]pallocSum

	// chunks is a slice of bitmap chunks.
	//
	// The total size of chunks is quite large on most 64-bit platforms
	// (O(GiB) or more) if flattened, so rather than making one large mapping
	// (which has problems on some platforms, even when PROT_NONE) we use a
	// two-level sparse array approach similar to the arena index in mheap.
	//
	// To find the chunk containing a memory address `a`, do:
	//   chunkOf(chunkIndex(a))
	//
	// Below is a table describing the configuration for chunks for various
	// heapAddrBits supported by the runtime.
	//
	// heapAddrBits | L1 Bits | L2 Bits | L2 Entry Size
	// ------------------------------------------------
	// 32           | 0       | 10      | 128 KiB
	// 33 (iOS)     | 0       | 11      | 256 KiB
	// 48           | 13      | 13      | 1 MiB
	//
	// There's no reason to use the L1 part of chunks on 32-bit, the
	// address space is small so the L2 is small. For platforms with a
	// 48-bit address space, we pick the L1 such that the L2 is 1 MiB
	// in size, which is a good balance between low granularity without
	// making the impact on BSS too high (note the L1 is stored directly
	// in pageAlloc).
	//
	// To iterate over the bitmap, use inUse to determine which ranges
	// are currently available. Otherwise one might iterate over unused
	// ranges.
	//
	// TODO(mknyszek): Consider changing the definition of the bitmap
	// such that 1 means free and 0 means in-use so that summaries and
	// the bitmaps align better on zero-values.
	chunks [1 << pallocChunksL1Bits]*[1 << pallocChunksL2Bits]pallocData

	// The address to start an allocation search with. It must never
	// point to any memory that is not contained in inUse, i.e.
	// inUse.contains(searchAddr.addr()) must always be true. The one
	// exception to this rule is that it may take on the value of
	// maxOffAddr to indicate that the heap is exhausted.
	//
	// We guarantee that all valid heap addresses below this value
	// are allocated and not worth searching.
	searchAddr offAddr

	...
}
```

当内存分配过大或者当前chunk块没有连续的n pages空间时，需要从基数树中从上到下进行查找。基数树有个特性：要分配的内存越大，它能越快地查找到当前基数树是否有连续的满足需求的空间。

查找基数树过程：

从上到下、从左到右地查找每个节点是否符合要求。先计算pallocSum的开头有多少连续空间，如果大于等于npages，则说明找到了可用的空间和地址。如果小于npages，则会计算pallocSum字段的max，即中间有多少连续内存空间。如果max大于等于npages，那么需要继续向基数树当前节点对应的下一级查找，因为max大于npages,说明当前一定有连续空间大于等于npages，但是并不知道具体位置，需要向下一级查找才能找到可用的地址。如果max也不满足，那么还需要将当前pallocSum计算的end与后一节点的start加起来查看是否能够组成大于npage的连续空间。

每一次从基数树中查找到内存，或者时候从操作系统分配到内存时，都需要更新基数树中每个节点的pallocSum。

###### 操作系统内存申请

当在基数树中找不到可用的连续内存时，需要从操作系统中获取内存。从操作系统获取内存的代码是平台独立的，比如在Unix系统中，最终使用了mmap系统调用向操作系统申请内存。

```go
func sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer {
	p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
	if err != 0 {
		return nil
	}
	return p
}
```

每一次向操作系统申请的内存大小必须是heapArena的倍数。heapArena是和平台有关的内存大小。

Go语言对于heapArena有精准的管理，精准到每个指针大小的内存信息，每个page对应的mspan信息都有记录。heapArena中的bitmap用每两个bit记录一个指针(8byte)的内存信息，主要用于gc。spans将pageID对应到arena里的mspan。

```go
// A heapArena stores metadata for a heap arena. heapArenas are stored
// outside of the Go heap and accessed via the mheap_.arenas index.
//
//go:notinheap
type heapArena struct {
	// bitmap stores the pointer/scalar bitmap for the words in
	// this arena. See mbitmap.go for a description. Use the
	// heapBits type to access this.
	bitmap [heapArenaBitmapBytes]byte

	// spans maps from virtual address page ID within this arena to *mspan.
	// For allocated spans, their pages map to the span itself.
	// For free spans, only the lowest and highest pages map to the span itself.
	// Internal pages map to an arbitrary span.
	// For pages that have never been allocated, spans entries are nil.
	//
	// Modifications are protected by mheap.lock. Reads can be
	// performed without locking, but ONLY from indexes that are
	// known to contain in-use or stack spans. This means there
	// must not be a safe-point between establishing that an
	// address is live and looking it up in the spans array.
	spans [pagesPerArena]*mspan

	// pageInUse is a bitmap that indicates which spans are in
	// state mSpanInUse. This bitmap is indexed by page number,
	// but only the bit corresponding to the first page in each
	// span is used.
	//
	// Reads and writes are atomic.
	pageInUse [pagesPerArena / 8]uint8

	// pageMarks is a bitmap that indicates which spans have any
	// marked objects on them. Like pageInUse, only the bit
	// corresponding to the first page in each span is used.
	//
	// Writes are done atomically during marking. Reads are
	// non-atomic and lock-free since they only occur during
	// sweeping (and hence never race with writes).
	//
	// This is used to quickly find whole spans that can be freed.
	//
	// TODO(austin): It would be nice if this was uint64 for
	// faster scanning, but we don't have 64-bit atomic bit
	// operations.
	pageMarks [pagesPerArena / 8]uint8

	// pageSpecials is a bitmap that indicates which spans have
	// specials (finalizers or other). Like pageInUse, only the bit
	// corresponding to the first page in each span is used.
	//
	// Writes are done atomically whenever a special is added to
	// a span and whenever the last special is removed from a span.
	// Reads are done atomically to find spans containing specials
	// during marking.
	pageSpecials [pagesPerArena / 8]uint8

	// checkmarks stores the debug.gccheckmark state. It is only
	// used if debug.gccheckmark > 0.
	checkmarks *checkmarksMap

	// zeroedBase marks the first byte of the first page in this
	// arena which hasn't been used yet and is therefore already
	// zero. zeroedBase is relative to the arena base.
	// Increases monotonically until it hits heapArenaBytes.
	//
	// This field is sufficient to determine if an allocation
	// needs to be zeroed because the page allocator follows an
	// address-ordered first-fit policy.
	//
	// Read atomically and written with an atomic CAS.
	zeroedBase uintptr
}
```

##### 小对象分配

当对象不属于微小对象时，在内存分配时会继续判断其是否属于小对象，小对象指小于32KB的对象。Go语言会计算小对象对应哪一个等级的span，并在指定等级的span中查找。此后的流程就和微小对象的分配一样，经历mcache→mcentral→mheap位图查找→mheap基数树查找→操作系统分配的过程。

##### 大对象分配

大对象指的是大于32KB的对象，内存分配时不与mcache和mcentral沟通，直接通过mheap进行分配，每个大对象都是一个特殊的span，其class为0。

### 总结

Go语言运行时依靠细微的对象切割、极致的多级缓存、精准的位图管理实现了对内存的精细化管理以及快速的内存访问，同时减少了内存的碎片。

## 19. 垃圾回收（GC）初探

垃圾回收作为内存管理的一部分，包含3个重要的功能：分配和管理新对象、识别正在使用的对象、清除不再使用的对象。

### 垃圾回收的作用

* 减少错误和复杂性 具有垃圾回收的语言屏蔽了内存管理的复杂性，开发者可以更好的关注核心的业务逻辑，避免悬空指针、多次释放等手动释放管理内存时出现的问题。

* 解耦 具有垃圾回收功能的语言将垃圾收集的工作托管给了具有全局视野的运行时代码。开发者编写的业务代码将真正实现解耦，从而有利于开发、调试，并开发出更大规模的、高并发的项目。

  垃圾回收并不是在任何场景下都适用的，因为垃圾回收带来了额外的成本，需要保存内存的状态信息并扫描内存，很多时候还需要中断整个程序（STW技术）来处理垃圾回收。垃圾回收的常见指标包括程序暂停时间、空间开销、回收的及时性等，根据设计目标的侧重点不同有不同的垃圾回收算法。

### 垃圾回收的5种经典算法

* 标记-清扫

标记-清扫算法分为两个主要阶段，第1阶段是扫描并标记当前活着的对象，第2阶段是清扫没有被标记的垃圾对象。因此，标记-清扫算法是一种间接的垃圾回收算法，它不直接查找垃圾对象，而是通过活着的对象推断出垃圾对象。

扫描一般从栈上的根对象开始，只要对象引用了其他堆对象，就会一直向下扫描，因此可以采用深度优先搜索或者广度优先搜索的方式进行扫描。在扫描阶段，为了有效管理扫描对象的状态，可以通过颜色对对象的状态进行抽象。在Go语言中，使用了经典的三色标记算法。

标记-清扫算法的主要缺点在于可能产生内存碎片或者空洞，这会导致新对象分配失败。

* 标记-压缩

标记-压缩算法通过将分散的、活着的对象移动到更紧密的空间来解决内存碎片问题。标记-压缩算法分为标记与压缩两个阶段。标记过程与标记-清扫算法类似，压缩阶段需要扫描活着的对象并将其压缩到空闲的区域，这可以保证压缩后的空间更紧凑，从而解决内存碎片问题。同时，压缩后的空间能以更快的速度查找到空闲的内存区域（在已经使用内存的后方）。

标记-压缩算法的缺点在于内存对象在内存的位置是随机的，这常常会破坏缓存的局部性，并且需要一些额外空间来标记当前对象已经移动到了其他地方。在压缩阶段，如果B对象发生了转移，那么必须更新所有引用了B对象的A对象的指针，增加了实现的复杂性。

* 半空间复制

半空间复制是一种空间换时间的算法。经典的半空间复制算法只能使用一般的内存空间，保留另一半内存空间用于快速压缩内存。

半空间复制的压缩性消除了内存碎片的问题，同时其压缩时间比标记-压缩算法更短。半空间复制不分阶段，在扫描根对象时就可以直接压缩，每个扫描到的对象都会从fromspace的空间复制到tospace。一旦扫描完成，就会得到一个压缩后的副本。

* 引用计数

引用计数是一种简单直接的识别垃圾对象的算法。每个对象都包含一个引用计数，每当其他对象引用了此对象时，引用计数就会增加。反之，取消引用后，引用计数就会减少。一旦引用计数为0，就表明该对象为垃圾对象，需要被回收。

引用计数算法简单搞笑，在垃圾回收阶段不需要额外占用大量内存，但它也有自己的缺点：一些没有破坏性的操作，比如只读操作、循环迭代操作也需要更新引用计数，栈上的内存操作或寄存器操作更新引用计数是难以接受的。同时，引用计数必须原子更新，并发操作同一个对象会导致引用计数难以处理自引用的对象。

* 分代GC

分代GC指将对象按照存活时间划分。使用这种算法的前提是死去的对象一般都是刚创建不久的，因此没有必要反复的扫描旧对象，这大概率会加快垃圾回收的速度，提升处理能力和吞吐量，减少程序暂停的时间。但是这种算法没有办法及时回收老一代的对象，并且需要额外开销引用区分新老对象，特别是在有多代对象的时候。

以上五种算法在实践中都有许多微妙的变化，而不是直接使用的。

### Go语言中的垃圾回收

Go语言采用了并发三色标记算法进行回收。

三色标记法：

1. 0初始状态下所有对象都是白色的。
2. 从根节点开始遍历所有对象，把遍历到的对象变成灰色对象
3. 遍历灰色对象，将灰色对象引用的对象也变成灰色对象，然后将遍历过的灰色对象变成黑色对象。
4. 循环步骤3，直到灰色对象全部变黑色。
5. 通过写屏障(write-barrier)检测对象有变化，重复以上操作
6. 收集所有白色对象（垃圾）

Go1.0是单协程垃圾回收，在垃圾回收开始阶段需要停止所有的用户协程，并且在垃圾回收阶段只有一个协程在执行垃圾回收。

Go1.1之后，垃圾回收由多个协程并行执行，大大加快了垃圾回收的速度，但是这个阶段还是不允许用户协程执行。

Go1.5对垃圾回收进行了重大更新，允许用户协程与后台的垃圾回收同时执行，大大降低了用户协程暂停的时间（从300ms左右降低到了40ms）。

Go1.6大幅度减少了STW期间的任务，使得用户协程暂停的时间从40ms左右降到了5ms。

Go1.8使用了混合写屏障技术消除了栈重新扫描的时间，将用户协程暂停的时间降低到了0.5ms左右。这对于绝大部分场景来说几乎是无感知的。

## 20. 深入垃圾回收全流程

### 垃圾回收循环

Go语言的垃圾回收循环大致会经过以下几个阶段。当内存到达了垃圾回收的阈值后，将触发新一轮的垃圾回收。之后会先后经历标记准备阶段、并行标记阶段、标记终止阶段及垃圾清扫阶段。在并行标记阶段引入了辅助标记技术，在垃圾清扫阶段还引入了辅助清扫、系统驻留内存清除技术。

![image-20220916165835659](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202209161658743.png)

#### 标记准备阶段

标记准备阶段最主要的任务是清扫上一阶段GC遗留的需要清扫的对象，因为使用了懒清扫算法，所以当执行下一次GC时，可能还有垃圾对象没有被清扫。同时，标记准备阶段会重置各种状态和统计指标、启动专门用于标记的协程、统计需要扫描的任务数量、开启写屏障、启动标记协程等。标记准备阶段是初始阶段，执行轻量级的任务。

标记准备阶段会为每个逻辑处理器P启动一个标记协程，但并不是所有的标记协程都有执行的机会，因为在标记阶段，标记协程和正常执行用户代码的协程需要并行，以减少GC给用户程序带来的影响。

##### 计算标记协程的数量

Go语言规定后台标记协程消耗的CPU应该接近25%。

```go
// startCycle resets the GC controller's state and computes estimates
// for a new GC cycle. The caller must hold worldsema and the world
// must be stopped.
func (c *gcControllerState) startCycle() {
    ...
    // Compute the background mark utilization goal. In general,
	// this may not come out exactly. We round the number of
	// dedicated workers so that the utilization is closest to
	// 25%. For small GOMAXPROCS, this would introduce too much
	// error, so we add fractional workers in that case.
    // gcBackgroundUtilization = 0.25
	totalUtilizationGoal := float64(gomaxprocs) * gcBackgroundUtilization
	c.dedicatedMarkWorkersNeeded = int64(totalUtilizationGoal + 0.5)
	utilError := float64(c.dedicatedMarkWorkersNeeded)/totalUtilizationGoal - 1
	const maxUtilError = 0.3
	if utilError < -maxUtilError || utilError > maxUtilError {
		// Rounding put us more than 30% off our goal. With
		// gcBackgroundUtilization of 25%, this happens for
		// GOMAXPROCS<=3 or GOMAXPROCS=6. Enable fractional
		// workers to compensate.
		if float64(c.dedicatedMarkWorkersNeeded) > totalUtilizationGoal {
			// Too many dedicated workers.
			c.dedicatedMarkWorkersNeeded--
		}
		c.fractionalUtilizationGoal = (totalUtilizationGoal - float64(c.dedicatedMarkWorkersNeeded)) / float64(gomaxprocs)
	} else {
		c.fractionalUtilizationGoal = 0
	}

	// In STW mode, we just want dedicated workers.
	if debug.gcstoptheworld > 0 {
		c.dedicatedMarkWorkersNeeded = int64(gomaxprocs)
		c.fractionalUtilizationGoal = 0
	}

	// Clear per-P state
	for _, p := range allp {
		p.gcAssistTime = 0
		p.gcFractionalMarkTime = 0
	}

	// Compute initial values for controls that are updated
	// throughout the cycle.
	c.revise()

	if debug.gcpacertrace > 0 {
		assistRatio := float64frombits(atomic.Load64(&c.assistWorkPerByte))
		print("pacer: assist ratio=", assistRatio,
			" (scan ", gcController.heapScan>>20, " MB in ",
			work.initialHeapLive>>20, "->",
			c.heapGoal>>20, " MB)",
			" workers=", c.dedicatedMarkWorkersNeeded,
			"+", c.fractionalUtilizationGoal, "\n")
	}
}
```

`dedicatedMarkWorkersNeeded`代表执行完整的后台标记协程的数量。例如当P=4，`dedicatedMarkWorkersNeeded`=1。`fractionalUtilizationGoal`是一个附加的参数，它的值小于1，专门为P为1、2、3、6时而设计的。例如当P=2，它的值是0.25。代表着每个P在标记阶段需要花费25%的时间执行后台标记协程。

![image-20220919161052310](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202209191610383.png)

在总的标记周期t内，每个P都需要花费25%的时间来执行后台标记工作。当超出时间后，当前的后台协程可以被抢占，从而执行其他协程。

##### 切换到后台标记协程

在标记准备阶段执行了STW，此时暂停了所有的协程。当关闭STW准备再次启动所有协程时，每个逻辑处理器P都会进行新的一轮调度循环。

```go
func schedule(){
    ...
    //正在GC,后台标记协程
    if gp == nil && gcBlackenEnabled != 0 {
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		if gp != nil {
			tryWakeP = true
		}
	}
    ...
}

// findRunnableGCWorker returns a background mark worker for _p_ if it
// should be run. This must only be called when gcBlackenEnabled != 0.
func (c *gcControllerState) findRunnableGCWorker(_p_ *p) *g {
	if gcBlackenEnabled == 0 {
		throw("gcControllerState.findRunnable: blackening not enabled")
	}

	if !gcMarkWorkAvailable(_p_) {
		// No work to be done right now. This can happen at
		// the end of the mark phase when there are still
		// assists tapering off. Don't bother running a worker
		// now because it'll just return immediately.
		return nil
	}

	// Grab a worker before we commit to running below.
	node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
	if node == nil {
		// There is at least one worker per P, so normally there are
		// enough workers to run on all Ps, if necessary. However, once
		// a worker enters gcMarkDone it may park without rejoining the
		// pool, thus freeing a P with no corresponding worker.
		// gcMarkDone never depends on another worker doing work, so it
		// is safe to simply do nothing here.
		//
		// If gcMarkDone bails out without completing the mark phase,
		// it will always do so with queued global work. Thus, that P
		// will be immediately eligible to re-run the worker G it was
		// just using, ensuring work can complete.
		return nil
	}

	decIfPositive := func(ptr *int64) bool {
		for {
			v := atomic.Loadint64(ptr)
			if v <= 0 {
				return false
			}

			if atomic.Casint64(ptr, v, v-1) {
				return true
			}
		}
	}

	if decIfPositive(&c.dedicatedMarkWorkersNeeded) {
		// This P is now dedicated to marking until the end of
		// the concurrent mark phase.
		_p_.gcMarkWorkerMode = gcMarkWorkerDedicatedMode
	} else if c.fractionalUtilizationGoal == 0 {
		// No need for fractional workers.
		gcBgMarkWorkerPool.push(&node.node)
		return nil
	} else {
		// Is this P behind on the fractional utilization
		// goal?
		//
		// This should be kept in sync with pollFractionalWorkerExit.
		delta := nanotime() - c.markStartTime
		if delta > 0 && float64(_p_.gcFractionalMarkTime)/float64(delta) > c.fractionalUtilizationGoal {
			// Nope. No need to run a fractional worker.
			gcBgMarkWorkerPool.push(&node.node)
			return nil
		}
		// Run a fractional worker.
		_p_.gcMarkWorkerMode = gcMarkWorkerFractionalMode
	}

	// Run the background mark worker.
	gp := node.gp.ptr()
	casgstatus(gp, _Gwaiting, _Grunnable)
	if trace.enabled {
		traceGoUnpark(gp, 0)
	}
	return gp
}
```

在`findRunnableGCWorker`函数中，如果`dedicatedMarkWorkersNeeded`大于0，当前协程立即执行后台标记任务。如果`fractionalUtilizationGoal`大于0，并且当前逻辑处理器P执行标记任务的时间小于`fractionalUtilizationGoal`*当前标记周期的时间。则仍然会执行后台标记任务，但并不会在整个标记周期内一直执行。此时，后台标记协程的运行模式会切换为`gcMarkWorkerFractionalMode`。

#### 并发标记阶段

在并发标记阶段，后台标记协程可以与执行用户代码的协程并行。Go语言的目标就是后台标记协程占用CPU的时间为25%，以最大限度的避免因执行GC而中断或减慢用户协程的执行。后台标记任务有三种不同的模式：

* gcMarkWorkerDedicatedMode 代表处理器专门负责标记对象，不会被调度器抢占。

* gcMarkWorkerFractionalMode 代表协助后台标记，在标记阶段到达目标时间后，会自动退出。

* gcMarkWorkerIdleMode 代表当处理器没有查找到可以执行的协程时，执行垃圾收集的标记任务，直到被抢占。

标记阶段的核心逻辑位于`runtime.gcDrain`函数，其中第2个参数为后台标记flag。后台标记flag有4种，`gcDrainUntilPreempt`标记代表当前标记协程处于可以被抢占的状态。`gcDrainFlushBgCredit`标记会计算后台完成的标记任务量以减少并行标记期间用户协程执行辅助垃圾收集的工作流。`gcDrainIdle`标记对应`gcMarkWorkerIdleMode` 模式，表示当处理器上包含其他待执行的协程时，标记协程退出。`gcDrainFractional`标记对应`gcMarkWorkerFractionalMode` 模式，表示后台标记协程到达目标时间后退出。

```go
// gcDrain scans roots and objects in work buffers, blackening grey
// objects until it is unable to get more work. It may return before
// GC is done; it's the caller's responsibility to balance work from
// other Ps.
//
// If flags&gcDrainUntilPreempt != 0, gcDrain returns when g.preempt
// is set.
//
// If flags&gcDrainIdle != 0, gcDrain returns when there is other work
// to do.
//
// If flags&gcDrainFractional != 0, gcDrain self-preempts when
// pollFractionalWorkerExit() returns true. This implies
// gcDrainNoBlock.
//
// If flags&gcDrainFlushBgCredit != 0, gcDrain flushes scan work
// credit to gcController.bgScanCredit every gcCreditSlack units of
// scan work.
//
// gcDrain will always return if there is a pending STW.
//
//go:nowritebarrier
func gcDrain(gcw *gcWork, flags gcDrainFlags) {...}
```

在`gcMarkWorkerDedicatedMode` 模式下，会一直执行后台标记任务，这意味着当前逻辑处理器P本地队列的协程将一直得不到执行，这是不能接受的。所以在Go语言中的做法是限制性可以被抢占的后台标记任务，如果标记协程已经被其他协程抢占，那么当前的逻辑处理器P并不会执行其他协程，而是将其他协程转移到全局队列中，并取消`gcDrainUntilPreempt`标志，进入不能被抢占模式。

```go
switch pp.gcMarkWorkerMode {
			default:
				throw("gcBgMarkWorker: unexpected gcMarkWorkerMode")
			case gcMarkWorkerDedicatedMode:
				gcDrain(&pp.gcw, gcDrainUntilPreempt|gcDrainFlushBgCredit)
				if gp.preempt {
					// We were preempted. This is
					// a useful signal to kick
					// everything out of the run
					// queue so it can run
					// somewhere else.
					if drainQ, n := runqdrain(pp); n > 0 {
						lock(&sched.lock)
						globrunqputbatch(&drainQ, int32(n))
						unlock(&sched.lock)
					}
				}
				// Go back to draining, this time
				// without preemption.
				gcDrain(&pp.gcw, gcDrainFlushBgCredit)
			case gcMarkWorkerFractionalMode:
				gcDrain(&pp.gcw, gcDrainFractional|gcDrainUntilPreempt|gcDrainFlushBgCredit)
			case gcMarkWorkerIdleMode:
				gcDrain(&pp.gcw, gcDrainIdle|gcDrainUntilPreempt|gcDrainFlushBgCredit)
			}
```

`gcMarkWorkerFractionalMode`和`gcMarkWorkerIdleMode`模式都允许被抢占。`gcMarkWorkerFractionalMode`模式加上了gcDrainFractional标记表明当前标记协程会在到达目标时间后退出。`gcMarkWorkerIdleMode`模式加上了`gcDrainIdle`标记表明当发现有其他协程可以运行时退出当前标记协程。三种模式都加上了`gcDrainFlushBgCredit`标记用于计算后台完成的标记任务量，并唤醒之前由于分配内存太频繁而陷入等待的用户协程。

##### 根对象扫描

扫描的第一阶段就是扫描根对象。在最开始的标记准备阶段会统计这次GC一共要扫描多少对象。每个具体的序号都对应着要扫描的对象。

```go
			job := atomic.Xadd(&work.markrootNext, +1) - 1
```

`work.markrootNext`必须原子增加，保证多个后台标记协程能够并发执行不同任务。根对象就是最基本的对象，从根对象出发，可以找到所有的引用对象。在Go语言中，根对象包括全局变量（在.bss和.data段内存中）、span中finalizer的任务数量以及所有的协程栈。finalizer是Go语言中的对象绑定析构器，当对象的内存释放后，需要调用析构器函数，从而完整释放资源。

###### 全局变量扫描

扫描全局变量需要编译时和运行时的共同努力。在运行时才能确定全局变量分配到虚拟内存的哪一块区域，在编译时，可以确定全局变量哪些位置在包含指针。信息位于位图ptrmask字段中。ptrmask的每个bit都对应.data段中的一个指针的大小（8KB），bit位为1代表当前位置是一个指针，这时，需要求出当前的指针在堆区的哪一个对象上，并将当前对象标记为灰色。

![image-20220919172712359](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202209191727443.png)

如何通过指针找到指针对应的对象位置呢？这依赖于Go语言对内存的精细化管理，先找到指针在哪一个heapArena中，通过heapArena可以找到其对应的mspan，进而找到其位于mspan中第几个元素中。当找到此元素后，会将`gcmarkBits`位图对应的bit设置为1，表明其已经被标记，同时将该元素(对象)放入标记队列中。

###### finalizer

finalizer是特殊的对象，在对象释放后会被调用的析构器，用于资源释放。析构器不会被栈上或全局变量引用，需要单独处理。

在标记期间，后台标记协程会遍历mspan的specials链表，扫描finalizer所位于的元素（对象），并扫描当前元素（对象）。在这里并不能把finalizer所位于的span对象加入根对象中，否则我们将时区回收该对象的机会。同时需要扫描析构器字段fn，因为fn可能指向了堆中的内存，并可能被回收。

```go
//runtime/mgcmark.go:305 markrootSpans() 
			for sp := s.specials; sp != nil; sp = sp.next {
				if sp.kind != _KindSpecialFinalizer {
					continue
				}
				// don't mark finalized object, but scan it so we
				// retain everything it points to.
				spf := (*specialfinalizer)(unsafe.Pointer(sp))
				// A finalizer can be set for an inner byte of an object, find object beginning.
				p := s.base() + uintptr(spf.special.offset)/s.elemsize*s.elemsize

				// Mark everything that can be reached from
				// the object (but *not* the object itself or
				// we'll never collect it).
				scanobject(p, gcw)

				// The special itself is a root.
				scanblock(uintptr(unsafe.Pointer(&spf.fn)), sys.PtrSize, &oneptrmask[0], gcw, nil)
			}
```

使用finalizer的例子：Go语言的文件描述符使用了finalizer，这样在文件描述符不再被使用时，即便用户忘记了手动关闭文件描述符，在GC的时候也可以自动调用finalizer关闭文件描述符。finalizer可以将资源的释放托管给垃圾回收。比如在CGO场景下，Go语言调用C函数时，C函数分配的内存不受Go垃圾回收的管理，我们常常借助defer在调用结束时手动释放内存，也可以将其改成finalizer形式。

###### 栈扫描

栈扫描是根对象扫描中最重要的部分。栈扫描需要编译时和运行时共同努力，运行时能够计算出当前协程栈的所有栈帧信息，编译时能够知道栈上有哪些指针，以及对象中的哪一部分包含了指针。运行时首先计算出栈帧布局，每一个栈帧都代表一个函数，运行时可以得知当前栈帧的函数参数、函数本地变量、寄存器SP、BP等一系列信息。每个栈帧函数的参数和局部变量都需要进行扫描，确认该对象是否仍然在使用，如果在使用则需要扫描位图判断对象中是否包含指针。

```go
// Scan a stack frame: local variables and function arguments/results.
//go:nowritebarrier
func scanframeworker(frame *stkframe, state *stackScanState, gcw *gcWork) {
	...

	locals, args, objs := getStackMap(frame, &state.cache, false)

	// Scan local variables if stack frame has been allocated.
    // 扫描局部变量
	if locals.n > 0 {
		size := uintptr(locals.n) * sys.PtrSize
		scanblock(frame.varp-size, size, locals.bytedata, gcw, state)
	}
	// 扫描函数参数
	// Scan arguments.
	if args.n > 0 {
		scanblock(frame.argp, uintptr(args.n)*sys.PtrSize, args.bytedata, gcw, state)
	}

	// Add all stack objects to the stack object list.
	if frame.varp != 0 {
		// varp is 0 for defers, where there are no locals.
		// In that case, there can't be a pointer to its args, either.
		// (And all args would be scanned above anyway.)
		for i, obj := range objs {
			off := obj.off
			base := frame.varp // locals base pointer
			if off >= 0 {
				base = frame.argp // arguments and return values base pointer
			}
			ptr := base + uintptr(off)
			if ptr < frame.sp {
				// object hasn't been allocated in the frame yet.
				continue
			}
			if stackTraceDebug {
				println("stkobj at", hex(ptr), "of size", obj.size)
			}
			state.addObject(ptr, &objs[i])
		}
	}
}
```

###### 栈对象

为了解决栈扫描采取保守算法而造成的内存泄漏问题，Go语言引用了栈对象的概念。栈对象是在栈上能够被寻址的对象。不是所有对象都会存储在栈上，比如存储在寄存器中的变量就不能被寻址。编译器会在编译时将所有的栈对象都记录下来，同时编译器将追踪栈中所有可能指向栈对象的指针。在垃圾回收期间，所有栈对象都会被存储到一颗二叉搜索树中。

##### 扫描灰色对象

从根对象的收集来看，全局变量、析构器、所有协程的栈都会被扫描，从而标记目前还在使用的内存对象。下一步就是从这些被标记为灰色的内存对象触发，进一步标记整个堆内存中活着的对象。在进行根对象扫描时，会将标记的对象放入本地队列中，如果本地队列放不下，则放到全局队列中，这样可以最大限度地避免使用锁，在本地缓存的队列可以被逻辑处理器P无锁访问。在进行扫描时，先消费本地队列中找到的标记对象，如果本地队列为空，则加锁获取全局队列中存储的对象。

在标记期间，会循环往复地从本地标记队列获取灰色对象，灰色对象扫描到的白色对象仍然会被放入标记队列中，如果扫描到已经标记的对象则忽略，一直到队列中的任务为空为止。

```go
	// Drain heap marking jobs.
	// Stop if we're preemptible or if someone wants to STW.
	for !(gp.preempt && (preemptible || atomic.Load(&sched.gcwaiting) != 0)) {
		// Try to keep work available on the global queue. We used to
		// check if there were waiting workers, but it's better to
		// just keep work available than to make workers wait. In the
		// worst case, we'll do O(log(_WorkbufSize)) unnecessary
		// balances.
        //从本地标记队列中获取对象，获取不到则从全局标记队列获取
		if work.full == 0 {
			gcw.balance()
		}

		b := gcw.tryGetFast()
		if b == 0 {
            //阻塞获取
			b = gcw.tryGet()
			if b == 0 {
				// Flush the write barrier
				// buffer; this may create
				// more work.
				wbBufFlush(nil, 0)
				b = gcw.tryGet()
			}
		}
		if b == 0 {
			// Unable to get work.
			break
		}
        //扫描获取的对象
		scanobject(b, gcw)
		...
    }
```

对象的扫描过程位于scanobject函数中。在对所有对象的内存逐个进行扫描，查看对象内存中是否包含指针，如果对象中没有存储指针，则根本不需要花时间进行检查。为了实现更快的查找，Go语言在内存分配时记录了对象中是否包含指针等元信息。heapArena有一个重要的bitmap字段用位图的形式记录了每个指针大小（8KB）的内存中的信息。每个指针大小的内存都会有2个bit分别表示当前内存是否应该继续扫描及是否包含指针。bitmap位图记录了内存是否需要被扫描以及是否包含指针。bitmap中1个byte大小的空间对应了虚拟内存中4个指针大小的空间。bitmap中前4位为扫描位，后4位为指针位，分别对应指定的指针大小的空间是否需要继续进行扫描及是否包含指针。

![image-20220921143745414](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202209211437533.png)

当需要继续扫描并且发现了当前有指针时，就需要取出指针的值，并对其进行扫描。后续操作与全局变量扫描类似，根据指针查找到span中的对象，如果发现引用的是堆中的白色对象，则标记该对象并将该对象放入本地队列中。

##### 辅助标记

Go1.5引入了并发标记后，带来了许多新的问题。比如：在并发标记阶段，扫描内存的同时用户协程也不断被分配内存，当用户协程的内存分配速度快到后台标记协程来不及扫描时，GC标记阶段将永远不会结束，从而无法完成完整的GC周期，造成内存泄漏。为了解决这样的问题，引入辅助标记算法。辅助标记必须在垃圾回收的标记阶段进行，由于用户协程被分配了超过限度的内存而不得不将其暂停并切换到辅助标记工作。

扫描策略可以看成是X=assistWorkPerByte * M。X为后台标记协程需要多扫描的内存，M为新分配的内存。

在GC并发标记阶段，当用户协程分配内存时，会先检查是否已经完成了制定的扫描工作。当前协程的`gcAssistBytes`字段代表当前协程可以被分配的内存大小，类似资产池。当本地的资产池不足时（gcAssistBytes<0），会尝试从全局的资产池中获取。用户协程一开始是没有资产的，所有资产都来自后台标记协程。用户协程中的本地资产来自后台标记协程的扫描工作。后台标记协程的扫描工作会增加全局资产池的大小。 如果标记协程已经扫描完成的内存为X，那么意味着全局资产池可以容忍用户协程分配的内存数量为M=X/assistWorkPerByte。这种机制保证了在GC并发标记时，工作协程分配的内存数量不会太多也不会太少。

```go
	// assistG is the G to charge for this allocation, or nil if0
	// GC is not currently active.
	var assistG *g
	// gcBlackenEnabled在GC的标记阶段会开启
	if gcBlackenEnabled != 0 {
		// Charge the current user G for this allocation.
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		// Charge the allocation against the G. We'll account
		// for internal fragmentation at the end of mallocgc.
		assistG.gcAssistBytes -= int64(size)
		// 需要辅助标记
		if assistG.gcAssistBytes < 0 {
			// This G is in debt. Assist the GC to correct
			// this before allocating. This must happen
			// before disabling preemption.
            // 尝试从全局资产池获取资产
			gcAssistAlloc(assistG)
		}
	}
```

如果工作内存在分配内存时，既无法从本地资产池也无法从全局资产池获取资产，那么需要停止工作协程，并执行辅助标记协程。辅助标记协程需要额外扫描的内存大小为assistWorkPerByte * M，当扫描完成指定工作或被抢占时会退出。当辅助标记完成后，如果本地仍然没有足够的资产，则可能是因为当前协程被抢占，也可能是因为当前逻辑处理器的工作池没有多余的标记工作。前一种情况会调用Gosched函数让渡当前辅助标记的执行权利，后一种情况会陷入休眠状态，当后台标记协程扫描了足够的任务后，会刷新全局资产池并将等待中的协程唤醒。

##### 屏障技术

辅助标记解决的是垃圾回收正常结束和循环的问题，屏障技术解决的则是准确性的问题。 保证并发标记准确性需要遵守的原则是强、弱三色不变性。

* 强三色不变性 黑色节点不允许引用白色节点。
* 弱三色不变性 黑色节点允许引用白色节点，但是该白色节点有其他灰色节点间接的引用（确保不会被遗漏） 

在并发标记写入和删除对象时，可能破坏三色不变性，因此需要屏障技术来维护三色不变性。屏障技术的原则是在写入或删除对象时将可能活着的对象标记为灰色。

Dijistra风格插入屏障：当黑色节点新增了白色节点的引用时，将对应的白色节点改为灰色。

Yuasa删除写屏障：当白色节点被删除了一个引用时，悲观地认为它一定会被一个黑色节点新增引用，所以将它置为灰色。

插入屏障和删除屏障通过在写入和删除时重新标记颜色保证了三色不变性，解决了并发标记期间的准确性问题。但是它们都存在浮动垃圾的问题，插入屏障在删除引用的时候可能标记一个已经变成垃圾的对象，而删除屏障在删除引用时可能把一个垃圾对象标记为灰色。这些是垃圾回收的精度问题，不会影响准确性，因为浮动垃圾会在下一次垃圾回收中被回收。

插入屏障和删除屏障独立存在并能良好工作的前提是并发标记期间所有的写入操作都应用了屏障技术。为了解决重复扫描的问题，Go1.8之后使用了混合写屏障技术，结合了Dijistra和Yuasa两种风格。伪代码如下

```pseudocode
writePoint(slot,ptr):
	shade(*slot)
	shade(ptr)
	*slot=ptr
```

在Go语言中，混合写屏障技术的实现依赖编译时和运行时的共同努力。在标记准备阶段的STW阶段会打开写屏障，具体做法是将全局变量writeBarrier.enabled设置为true。

```go
// The compiler knows about this variable.
// If you change it, you must change builtin/runtime.go, too.
// If you change the first four bytes, you must also change the write
// barrier insertion code.
var writeBarrier struct {
	enabled bool    // compiler emits a check of this before calling write barrier
	pad     [3]byte // compiler uses 32-bit load for "enabled" field
	needed  bool    // whether we need a write barrier for current GC phase
	cgo     bool    // whether we need a write barrier for a cgo check
	alignme uint64  // guarantee alignment so that compiler can use a 32 or 64-bit load
}
```

编译器会在所有堆写入或者删除操作前判断当前是否为垃圾回收阶段，如果是则会执行对应的混合写屏障标记对象。Go语言中构建了写屏障指针缓存池，gcWriteBarrier先将所有被标记的指针放入缓存池中，并在容量满后，一次性全部刷新到扫描任务池中。最终这些被标记的指针都将被扫描。汇编代码如下：

```assembly
CMPL	runtime.writeBarrier(SB),$0
CALL	runtime.gcWriteBarrier(SB)
```

#### 标记终止阶段

完成并发标记阶段所有灰色对象的扫描和标记后进入标记终止阶段，标记终止阶段主要完成一些指标，比如统计用时、统计强制开始GC的次数、更新下一次触发GC需要达到的堆目标、关闭写屏障等，并唤醒后台清扫的协程，开始下一阶段的清扫工作。标记终止阶段会再次进入STW。

标记终止阶段的重要任务是计算下一次触发GC时需要达到的堆目标，这叫作垃圾回收的调步算法。调步算法最重要的任务是估计下一次触发GC的最佳时机，这依赖本次GC的阶段差额(GC完成后占用内存和目标内存之间的差距)。如果GC完成后占用内存远小于目标内存，意味着触发GC时间过早，如果占用内存远大于目标内存，则意味着触发GC时间太迟。因此调度算法的第1个目标是min(|目标占用内存-GC完成后的占用内存|)，第2个目标是预计执行标记的CPU占用率接近25%。如果用户协程执行了过多的辅助标记，则会导致GC完成后的占用内存偏小，因为用户协程本来应该用来分配内存的时间用来执行了辅助标记。所以算法将先计算目标内存和GC完成后的内存的偏差：
$$
偏差率=(目标增长率-触发率)-(实际增长率-触发率)
$$
其中，

* 目标增长率=目标内存/上一次GC完成后的标记内存-1 
* 触发率=触发GC时的占用内存/上一次GC完成后的标记内存-1 
* 完成率=GC完成后的内存/上一次GC完成后的标记内存-1

这其实是偏差=(目标内存-触发GC时的占用内存)-(GC完成后的占用内存-触发GC时的占用内存)的变形，为了修复辅助标记带来的偏差，计算辅助标记所用的时间，从而调整(GC完成后的占用内存-触发GC时的占用内存)的大小，因此最终的偏差率为：
$$
偏差率=(目标增长率-触发率)-调整率\times(实际增长率-触发率)
$$
其中：调整率=GC标记阶段的CPU占用率/目标CPU占用率

```go
// endCycle computes the trigger ratio for the next cycle.
// userForced indicates whether the current GC cycle was forced
// by the application.
func (c *gcControllerState) endCycle(userForced bool) float64 {
	if userForced {
		// Forced GC means this cycle didn't start at the
		// trigger, so where it finished isn't good
		// information about how to adjust the trigger.
		// Just leave it where it is.
		return c.triggerRatio
	}

	// Proportional response gain for the trigger controller. Must
	// be in [0, 1]. Lower values smooth out transient effects but
	// take longer to respond to phase changes. Higher values
	// react to phase changes quickly, but are more affected by
	// transient changes. Values near 1 may be unstable.
	const triggerGain = 0.5

	// Compute next cycle trigger ratio. First, this computes the
	// "error" for this cycle; that is, how far off the trigger
	// was from what it should have been, accounting for both heap
	// growth and GC CPU utilization. We compute the actual heap
	// growth during this cycle and scale that by how far off from
	// the goal CPU utilization we were (to estimate the heap
	// growth if we had the desired CPU utilization). The
	// difference between this estimate and the GOGC-based goal
	// heap growth is the error.
	goalGrowthRatio := c.effectiveGrowthRatio()
	actualGrowthRatio := float64(c.heapLive)/float64(c.heapMarked) - 1
	assistDuration := nanotime() - c.markStartTime

	// Assume background mark hit its utilization goal.
	utilization := gcBackgroundUtilization
	// Add assist utilization; avoid divide by zero.
	if assistDuration > 0 {
		utilization += float64(c.assistTime) / float64(assistDuration*int64(gomaxprocs))
	}

	triggerError := goalGrowthRatio - c.triggerRatio - utilization/gcGoalUtilization*(actualGrowthRatio-c.triggerRatio)

	// Finally, we adjust the trigger for next time by this error,
	// damped by the proportional gain.
	triggerRatio := c.triggerRatio + triggerGain*triggerError

	if debug.gcpacertrace > 0 {
		// Print controller state in terms of the design
		// document.
		H_m_prev := c.heapMarked
		h_t := c.triggerRatio
		H_T := c.trigger
		h_a := actualGrowthRatio
		H_a := c.heapLive
		h_g := goalGrowthRatio
		H_g := int64(float64(H_m_prev) * (1 + h_g))
		u_a := utilization
		u_g := gcGoalUtilization
		W_a := c.scanWork
		print("pacer: H_m_prev=", H_m_prev,
			" h_t=", h_t, " H_T=", H_T,
			" h_a=", h_a, " H_a=", H_a,
			" h_g=", h_g, " H_g=", H_g,
			" u_a=", u_a, " u_g=", u_g,
			" W_a=", W_a,
			" goalΔ=", goalGrowthRatio-h_t,
			" actualΔ=", h_a-h_t,
			" u_a/u_g=", u_a/u_g,
			"\n")
	}

	return triggerRatio
}
```

实际增长率和辅助标记的时长都会影响最终的偏差率。目标内存和GC完成后的占用内存偏离越大，偏差率越大。这时，下一次GC的触发率会渐进调整，即每次只调整偏差的一半，公式如下：
$$
下次GC触发率=上次GC触发率+1/2\times偏差率
$$
计算出下次GC偏差率后，需要计算出目标内存大小，目标内存的计算如下：

```go
	// Compute the next GC goal, which is when the allocated heap
	// has grown by GOGC/100 over the heap marked by the last
	// cycle.
	goal := ^uint64(0)
	if c.gcPercent >= 0 {
		goal = c.heapMarked + c.heapMarked*uint64(c.gcPercent)/100
	}
```

goal为下次GC完成后的目标啊内存，它的大小取决于本次GC扫描后的占用内存以及gcpercent的大小。gcpercent可以由用户调用debug.SetGCPercent()函数动态设置。gcpercent默认值为100，代表目标内存是上次GC内存的2倍。当gcpercent小于0，将禁用Go的垃圾回收。还可以通过设置GOGC环境变量来修改gcpercent的大小。

在明确了目标内存后，触发内存的大小可以简单定义如下：
$$
触发内存=触发率\times目标内存
$$
其中触发率不能大于0.95，也不能小于0.6。

#### 垃圾清扫

垃圾标记工作完成意味着已经追踪到内存中所有活着的对象，之后进入垃圾清扫阶段，将垃圾对象内存回收重用或返还给操作系统。标记结束阶段会调用`gcSweep`函数，该函数会将sweep.g清扫协程的状态变为running，在结束STW阶段并开始重新调度循环时优先清扫协程。

```go
// gcSweep must be called on the system stack because it acquires the heap
// lock. See mheap for details.
//
// The world must be stopped.
//
//go:systemstack
func gcSweep(mode gcMode) {
	assertWorldStopped()

	if gcphase != _GCoff {
		throw("gcSweep being done but phase is not GCoff")
	}

	lock(&mheap_.lock)
	mheap_.sweepgen += 2
	mheap_.sweepDrained = 0
	mheap_.pagesSwept = 0
	mheap_.sweepArenas = mheap_.allArenas
	mheap_.reclaimIndex = 0
	mheap_.reclaimCredit = 0
	unlock(&mheap_.lock)

	sweep.centralIndex.clear()

	if !_ConcurrentSweep || mode == gcForceBlockMode {
		// Special case synchronous sweep.
		// Record that no proportional sweeping has to happen.
		lock(&mheap_.lock)
		mheap_.sweepPagesPerByte = 0
		unlock(&mheap_.lock)
		// Sweep all spans eagerly.
		for sweepone() != ^uintptr(0) {
			sweep.npausesweep++
		}
		// Free workbufs eagerly.
		prepareFreeWorkbufs()
		for freeSomeWbufs(false) {
		}
		// All "free" events for this mark/sweep cycle have
		// now happened, so we can make this profile cycle
		// available immediately.
		mProf_NextCycle()
		mProf_Flush()
		return
	}

	// Background sweep.
	lock(&sweep.lock)
	if sweep.parked {
		sweep.parked = false
		ready(sweep.g, 0, true)
	}
	unlock(&sweep.lock)
}
```

清扫阶段在程序启动时调用的gcenable函数中启动。程序只有一个垃圾清扫协程，并在清扫阶段与用户协程同时进行。

```go
// gcenable is called after the bulk of the runtime initialization,
// just before we're about to start letting user code run.
// It kicks off the background sweeper goroutine, the background
// scavenger goroutine, and enables GC.
func gcenable() {
	// Kick off sweeping and scavenging.
	gcenable_setup = make(chan int, 2)
    //启动后台清扫器，与用户态代码并发被调度，归还从内存分配器中申请的内存
	go bgsweep()
    //启动后台清扫器，与用户态代码并发被调度，归还从操作系统中申请的内存
	go bgscavenge()
	<-gcenable_setup
	<-gcenable_setup
	gcenable_setup = nil
	memstats.enablegc = true // now that runtime is initialized, GC is okay
}
```

当清扫协程被唤醒后，会开始垃圾清扫。垃圾清扫采取了懒清扫的策略（执行少量清扫工作后，通过Gosched函数让渡自己的执行权利，不需要一直执行）。因此当触发下一阶段的垃圾回收后，可能有没有被清理的内存，需要先将它们清理完。

```go
// sweepone sweeps some unswept heap span and returns the number of pages returned
// to the heap, or ^uintptr(0) if there was nothing to sweep.
func bgsweep() {
	sweep.g = getg()

	lockInit(&sweep.lock, lockRankSweep)
	lock(&sweep.lock)
	sweep.parked = true
	gcenable_setup <- 1
	goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceEvGoBlock, 1)
	//等待唤醒
	for {
        //清扫一个span 然后进入调度（一次只进行少量工作）
		for sweepone() != ^uintptr(0) {
			sweep.nbgsweep++
			Gosched()
		}
        //可抢占地释放一些workbufs到堆中
        //释放一些未使用的标记队列缓冲区到heap
		for freeSomeWbufs(true) {
			Gosched()
		}
		lock(&sweep.lock)
		if !isSweepDone() {
			// This can happen if a GC runs between
			// gosweepone returning ^0 above
			// and the lock being acquired.
			unlock(&sweep.lock)
			continue
		}
		sweep.parked = true
        //休眠
		goparkunlock(&sweep.lock, waitReasonGCSweepWait, traceEvGoBlock, 1)
	}
}
```

##### 懒清扫逻辑

清扫是以span为单位进行的，sweepone函数的作用是找到一个span并进行相应的清扫工作。

先从mheap中的sweepSpans队列中取出需要清扫的span：

sweepSpans数组的长度为2，sweepSpans[sweepgen/2%2]保存当前正在使用的span列表，sweepSpans[1-sweepgen/2%2]保存等待清扫的span列表。sweepgen每次清扫时加2，所以sweepSpans[0]、sweepSpans[1]每次清扫时互相交换身份，本次正在使用的span列表为下一次GC待清扫的列表。

```go
    // partial and full contain two mspan sets: one of swept in-use
	// spans, and one of unswept in-use spans. These two trade
	// roles on each GC cycle. The unswept set is drained either by
	// allocation or by the background sweeper in every GC cycle,
	// so only two roles are necessary.
	//
	// sweepgen is increased by 2 on each GC cycle, so the swept
	// spans are in partial[sweepgen/2%2] and the unswept spans are in
	// partial[1-sweepgen/2%2]. Sweeping pops spans from the
	// unswept set and pushes spans that are still in-use on the
	// swept set. Likewise, allocating an in-use span pushes it
	// on the swept set.
	//
	// Some parts of the sweeper can sweep arbitrary spans, and hence
	// can't remove them from the unswept set, but will add the span
	// to the appropriate swept list. As a result, the parts of the
	// sweeper and mcentral that do consume from the unswept list may
	// encounter swept spans, and these should be ignored.
	partial [2]spanSet // list of spans with a free object
	full    [2]spanSet // list of spans with no free objects

s := mheap_.nextSpanForSweep()

// nextSpanForSweep finds and pops the next span for sweeping from the
// central sweep buffers. It returns ownership of the span to the caller.
// Returns nil if no such span exists.
func (h *mheap) nextSpanForSweep() *mspan {
	sg := h.sweepgen
	for sc := sweep.centralIndex.load(); sc < numSweepClasses; sc++ {
		spc, full := sc.split()
		c := &h.central[spc].mcentral
		var s *mspan
		if full {
			s = c.fullUnswept(sg).pop()
		} else {
			s = c.partialUnswept(sg).pop()
		}
		if s != nil {
			// Write down that we found something so future sweepers
			// can start from here.
			sweep.centralIndex.update(sc)
			return s
		}
	}
	// Write down that we found nothing.
	sweep.centralIndex.update(sweepClassDone)
	return nil
}

// split returns the underlying span class as well as
// whether we're interested in the full or partial
// unswept lists for that class, indicated as a boolean
// (true means "full").
func (s sweepClass) split() (spc spanClass, full bool) {
	return spanClass(s >> 1), s&1 == 0
}

// fullUnswept returns the spanSet which holds unswept spans without any
// free slots for this sweepgen.
func (c *mcentral) fullUnswept(sweepgen uint32) *spanSet {
	return &c.full[1-sweepgen/2%2]
}

// partialUnswept returns the spanSet which holds partially-filled
// unswept spans for this sweepgen.
func (c *mcentral) partialUnswept(sweepgen uint32) *spanSet {
	return &c.partial[1-sweepgen/2%2]
}

// gcSweep must be called on the system stack because it acquires the heap
// lock. See mheap for details.
//
// The world must be stopped.
//
//go:systemstack
func gcSweep(mode gcMode) {
	assertWorldStopped()

	if gcphase != _GCoff {
		throw("gcSweep being done but phase is not GCoff")
	}

	lock(&mheap_.lock)
	mheap_.sweepgen += 2
	mheap_.sweepDrained = 0
	mheap_.pagesSwept = 0
	mheap_.sweepArenas = mheap_.allArenas
	mheap_.reclaimIndex = 0
	mheap_.reclaimCredit = 0
	unlock(&mheap_.lock)
    ...
}
```

在清扫span期间，最重要的一步是将gcmarkBits位图赋值给allocBits位图。

```go
    // gcmarkBits becomes the allocBits.
	// get a fresh cleared gcmarkBits in preparation for next GC
	s.allocBits = s.gcmarkBits
	s.gcmarkBits = newMarkBits(s.nelems)
```

当前gcmarkBits是GC标记后的最新的对象位图，当gcmarkBits中的bit位为1时，代表当前对象是活着的。所以当gcmarkBits中的某一个bit位为1，但是对应的allocBits位图中的bit位为0时，代表这个对象是会被回收的垃圾对象。完成这一切换后，就可以通过位图使用已经是垃圾对象的内存了。如果GC后gcmarkBits的全部bit位都为0，那么意味着当前所有span中的对象都不会再被其他对象引用（大对象特殊，因为其span内部只有一个对象）。

这时，整个span将会被mheap回收，并更新整个基数树，表明当前span的整个空间都可以被程序再次使用。如果当前span的整个空间并不完全为空，那么span会被重新放入sweepSpans正在使用的span列表中。

```go
			// Return span back to the right mcentral list.
			if uintptr(nalloc) == s.nelems {
				mheap_.central[spc].mcentral.fullSwept(sweepgen).push(s)
			} else {
				mheap_.central[spc].mcentral.partialSwept(sweepgen).push(s)
			}

// fullSwept returns the spanSet which holds swept spans without any
// free slots for this sweepgen.
func (c *mcentral) fullSwept(sweepgen uint32) *spanSet {
	return &c.full[sweepgen/2%2]
}

// partialSwept returns the spanSet which holds partially-filled
// swept spans for this sweepgen.
func (c *mcentral) partialSwept(sweepgen uint32) *spanSet {
	return &c.partial[sweepgen/2%2]
}
```

可以看出，这种回收方式并没有直接将内存释放到操作系统中，而是再次组织内存以便能在下次内存分配时利用已经被回收的内存。

##### 辅助清扫

为了规避懒清扫导致的剩余的未清扫span过多而拖后下一次GC开始时间的问题，Go语言采用了辅助清扫的手段。即工作协程必须在适当的时机执行辅助清扫工作，以避免下一次GC发生时还有大量的未清扫span。判断是否需要清扫的最好时机是在工作协程分配内存时。

目前Go语言会在两个时机判断是否需要辅助扫描。一个是在需要向mcentral申请内存时，一个是在大对象分配时。在这两个时间会判断当前已经清扫的page数是否大于清理的目标page数，如果否就会进行辅助清扫知道条件成立。sweepPagesPerByte是一个重要参数，代表工作协程每分配1byte需要辅助清扫的page数。

```go
	sweptBasis := atomic.Load64(&mheap_.pagesSweptBasis)

	// Fix debt if necessary.
	newHeapLive := uintptr(atomic.Load64(&gcController.heapLive)-mheap_.sweepHeapLiveBasis) + spanBytes
	pagesTarget := int64(mheap_.sweepPagesPerByte*float64(newHeapLive)) - int64(callerSweepPages)
	for pagesTarget > int64(atomic.Load64(&mheap_.pagesSwept)-sweptBasis) {
		if sweepone() == ^uintptr(0) {
			mheap_.sweepPagesPerByte = 0
			break
		}
		if atomic.Load64(&mheap_.pagesSweptBasis) != sweptBasis {
			// Sweep pacing changed. Recompute debt.
			goto retry
		}
	}
	...
		//未清扫页数=使用中的页数-已清扫的页数
		sweepDistancePages := int64(pagesInUse) - int64(pagesSwept)
		if sweepDistancePages <= 0 {
			mheap_.sweepPagesPerByte = 0
		} else {
			mheap_.sweepPagesPerByte = float64(sweepDistancePages) / float64(heapDistance)
			mheap_.sweepHeapLiveBasis = heapLiveBasis
			// Write pagesSweptBasis last, since this
			// signals concurrent sweeps to recompute
			// their debt.
			atomic.Store64(&mheap_.pagesSweptBasis, pagesSwept)
		}
```

可以看出，辅助标记策略会尽可能地保证在下次触发GC时，已经扫描了所有带扫描的span。

##### 系统驻留内存清除

驻留内存(RSS)是主内存(RAM)保留的进程占用的内存部分，是从操作系统中分配的内存。为了将系统分配的内存保持在适当的大小，同时回收不再被使用的内存，Go语言使用了一个单独的后台清扫协程来清除内存，目前后台清扫协程是在程序开始时启动的，并且只启动一个。

```go
// gcenable is called after the bulk of the runtime initialization,
// just before we're about to start letting user code run.
// It kicks off the background sweeper goroutine, the background
// scavenger goroutine, and enables GC.
func gcenable() {
	// Kick off sweeping and scavenging.
	gcenable_setup = make(chan int, 2)
    //启动后台清扫协程，与用户态代码并发被调度，归还从内存分配器中申请的内存。
	go bgsweep()
    //启动后台清扫协程，与用户态代码并发被调度，归还从操作系统中申请的内存。
	go bgscavenge()
	<-gcenable_setup
	<-gcenable_setup
	gcenable_setup = nil
	memstats.enablegc = true // now that runtime is initialized, GC is okay
}
```

清除策略占用当前线程CPU1%的时间进行清除，所以大部分时间里，该协程处于休眠状态。bgscavenge保证实现1%CPU执行时间的目标。

在默认情况下，当占用内存达到上一次GC标记内存的2倍后将触发垃圾回收。

## 21. 调试：特征分析和事件追踪

Go语言中的pprof指对于指标或特征的分析，通过分析不仅可以查到程序中的错误（内存泄漏，race冲突，协程泄露），也能对程序进行优化。有标准库net/http/pprof（提供了通过http访问的便利方式）和runtime/pprof用于对外交互。

### pprof的使用方式

通过pprof进行特征分析时需要执行两个步骤：收集样本和分析样本。

收集样本可以通过两种方式，一个是引用net/http/pprof并在程序中开启http服务器，net/http/pprof会在初始化init函数时注册路由。另一种方式是直接在代码中需要分析的位置嵌入runtime/pprof分析函数。

### trace

trace工具可以提供指定时间内程序发生的时间的完整信息。这些信息包括：协程的创建、开始和结束，协程堵塞——系统调用、通道、锁，网络I/O相关事件，系统相关事件，垃圾回收相关事件。

收集trace文件的方式和pprof类似，一个是runtime/trace包，一个是net/http/pprof库集成了trace接口。
