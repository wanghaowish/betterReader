# go语言底层原理剖析

## go语言编译器

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
## 浮点数

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

## 类型推断

`:=`用于变量的类型推断。类型推断依赖于编译器的处理能力。

## 常量与隐式类型转换

const关键字声明常量，声明时可以指定或者忽略类型。等号左边的叫命名常量，等号右边的叫未命名常量，未命名常量只会在编译期间存在，因此不会存在在内存中。命名常存在内存静态只读区， 不能被修改。go语言禁止对常量进行取地址操作。

隐式类型转换的规则是有类型常量优先于无类型。无 类型常量运算时的优先级为：复数(Imag)>浮点数(float)>符文数(rune)>整数(int)

## 字符串的本质与实现

go语言中字符常量存储于静态编译区，字符串不能被修改，只能被访问。字符串本质是一个定长的字符数组。字符常量的拼接发生在编译时，字符串常量的拼接发生在运行时。字节数组和字符串的相互转换并不是无损的指针引用，而是涉及了复制。

字符串的struct：

```go
type StringHeader struct{
    Data uintptr //指向底层的字符数组
    Len int //代表字符串的长度
}
```

字母占据1个字节，中文一般占据3个字节。strings库内包含很多字符处理函数。strconv包含很多字符串转换的函数。``可以换行，""不能换行。字符串拼接的原理并不是简单的将一个字符串合并到另一个字符串中，而是找一个更大的空间，通过内存复制的形式将字符串复制到其中。

## 数组

三种声明方式

```go
var arr [3]int
var arr2 =[3]int{1,2,3}
arr3:=[...]int{1,2,3} //语法糖 ... 这种声明方式在编译时自动推断长度
```

数组在赋值和函数调用时的形参都是值复制。数组在编译时会进行重要的优化，当数组长度小于4时，运行时数组会被放置在栈中，大于4则会放在内存的静态只读区。数组一般在Go语言中比较少用。

## 切片

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

![img](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202207191423469.png)

```go
//nil切片
var slice []int
//空切片
silce := make( []int , 0 )
slice := []int{ }
```

![img](https://raw.githubusercontent.com/wanghaowish/picGo/main/img/202207191423045.png)

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



## 哈希表（map）

哈希表的原理是将多个键值对分散存储在buckets中。给定一个keyhash算法会计算出键值对存储的位置。map是o(1)时间复杂度的操作。

### 哈希碰撞

哈希碰撞就是不同的键通过哈希算法可能产生相同的值。避免哈希碰撞的策略主要有两种：拉链法和开放寻址法。

- 拉链法

  将同一个桶中的元素通过链表的形式形成链接。随着桶中元素的增加，可以不断链接新的元素，同时不用预先为元素分配内存。它的不足之处在于它需要存储额外的指针用于链接元素，增加了整个哈希表的大小。同时由于链表存储的地址不连续，所以无法高效地利用CPU高速缓存。

- 开放寻址法

  所有的元素都存储在桶的数组中。当必须插入新的条目时，按照某种探测策略操作，直到找到使用的数组插槽为止。当搜索元素时，将按相同的循序扫描存储桶，直到查找到目标记录或者找到未使用的插槽为止。Go采用的开放寻址法的线性探测策略，线性探测策略是顺序（每次探测间隔为1）的。

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

         - count代表map中的元素数量
         - flags代表当前map的状态（是否处在正在写入的状态等等）
         - 2的B次幂标识当前map中桶的数量，2^B=Buckets size
         - noverflow为map中溢出桶的数量。当溢出桶太多时，map会进行same-size map growth。其实质是避免桶过大导致内存泄露。
         - hash0代表生成hash的随机数种子
         - buckets指向当前map对应的桶的指针。
         - oldbuckets是在map扩容时存储旧桶的。当所有旧桶中的数据都已经转移到了新桶中时，则清空
         - nevacuate在扩容时使用，用于标记当前旧桶中小于nevacuate的数据已经被转移到了新桶中
         - extra存储map中的溢出桶

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

  - 第三步：根据hint=10，并根据算法规则来创建 B，当前B应该为1。

    ```
    hint            B
    0~8				0
    9~13            1
    14~26           2
    ...
    ```

  - 第四步：根据B去创建去创建桶（bmap对象）并存放在buckets数组中，当前bmap的数量应为2.只有在map的数量大于24，才会生成溢出桶。

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

  - map中数据总个数 / 桶个数 > 6.5 ，引发翻倍扩容。
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

## 函数和栈

函数是程序中为了执行特定任务而存在的一系列执行代码。在Go语言中，函数是一等公民，这意味着可以将它看作变量，并且它可以作为参数传递，返回及赋值。它还可以具有多返回值。

### 函数闭包



## defer的延迟调用

## 异常和异常捕获

## 接口和程序设计模式

## 反射

## 协程初探

## 协程设计和调度原理

## 通道和协程通信

## 并发

## 内存分配管理

## 垃圾回收（GC）初探

## 深入垃圾回收全流程

## 调试：特征分析和事件追踪

