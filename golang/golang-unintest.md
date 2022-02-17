# 单元测试框架（GoConvey+sqlmock+httptest）

# 前言

Go语言中的测试依赖go test命令。go test命令是一个按照一定约定和组织的测试代码的驱动程序。在包目录内，所有以_test.go为后缀名的源代码文件都是go test测试的一部分，不会被go build编译到最终的可执行文件中。go test命令会遍历所有的`_test.go`文件中符合上述命名规则的函数，然后生成一个临时的main包用于调用相应的测试函数，然后构建并运行、报告测试结果，最后清理测试中生成的临时文件。

Golang单元测试对文件名和方法名，参数都有很严格的要求。

```
    1、文件名必须以xx_test.go命名
    2、方法必须是TestXxxx开头
    3、方法参数必须 t *testing.T
    4、使用go test执行单元测试 -test.v 输出全部单元测试用例
```

如果测试文件包含函数:`func TestMain(m *testing.M)`那么生成的测试会先调用 TestMain(m)，然后再运行具体测试。TestMain运行在主goroutine中, 可以在调用 m.Run前后做任何设置（setup）和拆卸（teardown）。退出测试的时候应该使用m.Run的返回值作为参数调用os.Exit。

示例如下：

```go
func TestMain(m *testing.M) {
    fmt.Println("write setup code here...") // 测试之前的做一些设置
    // 如果 TestMain 使用了 flags，这里应该加上flag.Parse()
    retCode := m.Run()                         // 执行测试
    fmt.Println("write teardown code here...") // 测试之后做一些拆卸工作
    os.Exit(retCode)                           // 退出测试
}
```

# GoConvey

## 1.GoConvey介绍

GoConvey是一款针对Golang的测试框架，可以管理和运行测试用例，同时提供了丰富的断言函数，并支持很多 Web 界面特性。

## 2.具体使用

```bash
go get github.com/smartystreets/goconvey
```

```go
import(
. "github.com/smartystreets/goconvey/convey"
)
```

实现一个简单的斐波那契数列方法。

```go
package demotest

func fib(n int) int {
	if n < 2 {
		return n
	}
	return fib(n-1) + fib(n-2)
}
```

单元测试：

```go
package demotest

import (
	"testing"

	. "github.com/smartystreets/goconvey/convey"
	"github.com/stretchr/testify/assert"
)

func Test_fibByGolang(t *testing.T) {
	var (
		in       = 7
		expected = 13
	)
	actual := fib(in)
	if actual != expected {
		t.Errorf("fib(%d) = %#v; expected %#v", in, actual, expected)
	}
}

func Test_fibByGoConvey(t *testing.T) {
	var (
		in1       = 6
		expected1 = 8
		in2       = 7
		expected2 = 13
	)
	Convey("通过", t, func() {
		Convey("测试第六项", func() {
			So(fib(in1), ShouldEqual, expected1)
		})
		Convey("测试第七项", func() {
			So(fib(in2), ShouldEqual, expected2)
		})
	})
}

func Test_fibByTestify(t *testing.T) {
	var (
		in       = 7
		expected = 13
	)
	assert.Equal(t, expected, fib(in), "通过")
}
```

使用 GoConvey 书写单元测试，每个测试用例需要使用 Convey 函数包裹起来。它接受的第一个参数为 string 类型的描述；第二个参数一般为 *testing.T，即本例中的变量 t；第三个参数为不接收任何参数也不返回任何值的函数（习惯以闭包的形式书写）。

Convey 语句同样可以无限嵌套，以体现各个测试用例之间的关系，例如 `Test_fibByGoConvey`函数就采用了嵌套的方式体现它们之间的关系。需要注意的是，只有最外层的 Convey 需要传入变量 t，内层的嵌套均不需要传入。GoConvey 底层是借助了 `jtolds/gls` 这个库实现了 goroutine 的管理，也实现了多个子Convey 的并发执行。

最后，需要使用 So 语句来对条件进行判断。

针对想忽略但又不想删掉或注释掉某些断言操作，GoConvey提供了Convey/So的Skip方法：

- SkipConvey函数表明相应的闭包函数将不被执行
- SkipSo函数表明相应的断言将不被执行

当存在SkipConvey或SkipSo时，测试日志中会显式打上"skipped"形式的标记：

- 当测试代码中存在SkipConvey时，相应闭包函数中不管是否为SkipSo，都将被忽略，测试日志中对应的符号仅为一个"⚠"
- 当测试代码Convey语句中存在SkipSo时，测试日志中每个So对应一个"✔"或"✘"，每个SkipSo对应一个"⚠"，按实际顺序排列
- 不管存在SkipConvey还是SkipSo时，测试日志中都有字符串"`{n} total assertions (one or more sections skipped)`"，其中{n}表示测试中实际已运行的断言语句数

当测试中有需要时，可以定制断言函数

```go
type assertion func(actual interface{}, expected ...interface{}) string
```

默认 一个Convey 下的多个 So 断言，是**失败后就终止**的策略。如果想要调整，在Convey 参数中加上 失败策略即可，比如设置 失败后继续，就用 `FailureContinues`。但是要注意：这里的失败后策略是针对 一个Convey 下的多个So 断言来说的，而不是一个Convey 下的多个子Convey。

```go
  	// FailureContinues is a failure mode which prevents failing
	// So()-assertions from halting Convey-block execution, instead
	// allowing the test to continue past failing So()-assertions.
	FailureContinues FailureMode = "continue"

	// FailureHalts is the default setting for a top-level Convey()-block
	// and will cause all failing So()-assertions to halt further execution
	// in that test-arm and continue on to the next arm.
	FailureHalts FailureMode = "halt"

	// FailureInherits is the default setting for failure-mode, it will
	// default to the failure-mode of the parent block. You should never
	// need to specify this mode in your tests..
	FailureInherits FailureMode = "inherits"
```

```bash

$GOPATH/bin/goconvey
http://localhost:8080
```

# sqlmock

## 1.sqlmock介绍

在数据库应用开发过程中，会在数据库上执行各种 SQL 语句。在做单元测试的时候，一般不会与实际数据库交互，这时就需要mock 数据库操作。即在不建立真实连接的情况下，模拟 sql driver 中的各种操作。

## 2.具体使用

```bash
go get github.com/DATA-DOG/go-sqlmock
```

```go
package global

import (
	"fmt"

	"github.com/DATA-DOG/go-sqlmock"

	_ "github.com/go-sql-driver/mysql"
	"github.com/go-xorm/xorm"
	. "github.com/smartystreets/goconvey/convey"
)

var (
	Xb *xorm.Engine
)

func InitXorm() *xorm.Engine {
	var err error
	Xb, err = xorm.NewEngine("mysql", "root:123456@(127.0.0.1:3306)/gounittest?charset=utf8&parseTime=True&loc=Local")
	if err != nil {
		fmt.Println(err)
		return Xb
	}
	Xb.ShowSQL(true)
	fmt.Println("xorm init success")
	return Xb
}

func InitMockXorm() (*xorm.Engine, sqlmock.Sqlmock) {
	var err error
	Xb, err = xorm.NewEngine("mysql", "root:123456@(127.0.0.1:3306)/gounittest?charset=utf8&parseTime=True&loc=Local")
	So(err, ShouldBeNil)
	Xb.ShowSQL(true)
	fmt.Println("xorm init success")
	db, mock, err := sqlmock.New()
	So(err, ShouldBeNil)
	Xb.DB().DB = db
	return Xb, mock
}
```

```go
package demotest

import (
   "apusic/golang-unit-test/global"
   "apusic/golang-unit-test/model"
)

func GetUserById(id int64) (*model.User, error) {
   var u model.User
   _, err := global.Xb.ID(id).Get(&u)
   if err != nil {
      return nil, err
   }
   return &u, nil
}

func InsertUser(user *model.User) (int64, error) {
   result, err := global.Xb.Insert(user)
   if err != nil {
      return 0, err
   }
   return result, nil
}

```

```go
package demotest

import (
	"apusic/golang-unit-test/global"
	"apusic/golang-unit-test/model"
	"errors"
	"regexp"
	"testing"

	"github.com/DATA-DOG/go-sqlmock"
	. "github.com/smartystreets/goconvey/convey"
)

func Test_GetUserByIdByGoConvey(t *testing.T) {
	Convey("查询", t, func() {
		var mock sqlmock.Sqlmock
		global.Xb, mock = global.InitMockXorm()
		Convey("根据ID查询", func() {
			rows := sqlmock.NewRows([]string{"id", "name"}).AddRow(1, "wanghao")
			mock.ExpectQuery(regexp.QuoteMeta("SELECT `id`, `name` FROM `user` WHERE `id`=? LIMIT 1")).WithArgs(1).WillReturnRows(rows)
			u, sqlErr := GetUserById(1)
			So(sqlErr, ShouldBeNil)
			So(u, ShouldResemble, &model.User{Id: 1, Name: "wanghao"})
		})
		Convey("查询报错", func() {
			mock.ExpectQuery(regexp.QuoteMeta("SELECT `id`, `name` FROM `user` WHERE `id`=? LIMIT 1")).WithArgs(1).WillReturnError(errors.New("sql报错"))
			_, sqlErr := GetUserById(1)
			So(sqlErr, ShouldBeError)
		})
		Reset(func() {
			So(mock.ExpectationsWereMet(), ShouldBeNil)
		})
	})
}

func TestInsertUser(t *testing.T) {
	Convey("插入", t, func() {
		var mock sqlmock.Sqlmock
		global.Xb, mock = global.InitMockXorm()
		Convey("插入数据", func() {
			mock.ExpectExec(regexp.QuoteMeta("INSERT INTO `user` (`id`,`name`) VALUES (?, ?)")).WithArgs(2, "wanghao2").WillReturnResult(sqlmock.NewResult(1, 1))
			u := &model.User{
				Id:   2,
				Name: "wanghao2",
			}
			result, sqlErr := InsertUser(u)
			So(sqlErr, ShouldBeNil)
			So(result, ShouldNotBeZeroValue)
		})
		Convey("插入报错err", func() {
			mock.ExpectExec(regexp.QuoteMeta("INSERT INTO `user` (`id`,`name`) VALUES (?, ?)")).WithArgs(2, "wanghao2").WillReturnError(errors.New("插入报错"))
			u := &model.User{
				Id:   2,
				Name: "wanghao2",
			}
			_, sqlErr := InsertUser(u)
			So(sqlErr, ShouldBeError)
		})
		Reset(func() {
			So(mock.ExpectationsWereMet(), ShouldBeNil)
		})
	})
}
```

mock 对象提供了一组方法，实现 Sql mock。首先是`ExpectQuery`方法，指定查询的 Sql 语句，可以提供正则表达式，默认通过正则匹配。`WithArgs`指定 Sql 的参数，`WillReturnRows`设置期待返回的查询结果。每次执行完 case，都会执行`ExpectationsWereMet`判断所有的 Sql mock 是否被满足。如果想使用sql匹配，应当使用`regexp.QuoteMeta`将sql转义。

mock对象的常用方法：

```go
// ExpectPrepare expects Prepare() to be called with expectedSQL query.
	// the *ExpectedPrepare allows to mock database response.
	// Note that you may expect Query() or Exec() on the *ExpectedPrepare
	// statement to prevent repeating expectedSQL
	ExpectPrepare(expectedSQL string) *ExpectedPrepare

	// ExpectQuery expects Query() or QueryRow() to be called with expectedSQL query.
	// the *ExpectedQuery allows to mock database response.
	ExpectQuery(expectedSQL string) *ExpectedQuery

	// ExpectExec expects Exec() to be called with expectedSQL query.
	// the *ExpectedExec allows to mock database response
	ExpectExec(expectedSQL string) *ExpectedExec

	// ExpectBegin expects *sql.DB.Begin to be called.
	// the *ExpectedBegin allows to mock database response
	ExpectBegin() *ExpectedBegin

	// ExpectCommit expects *sql.Tx.Commit to be called.
	// the *ExpectedCommit allows to mock database response
	ExpectCommit() *ExpectedCommit

	// ExpectRollback expects *sql.Tx.Rollback to be called.
	// the *ExpectedRollback allows to mock database response
	ExpectRollback() *ExpectedRollback
```

# httptest

## 1.httptest介绍

Go 标准库专门提供了 `net/http/httptest` 包专门用于进行 http Web 开发测试。

## 2.具体使用
```go
package demotest

import (
   "apusic/golang-unit-test/model"
   "strconv"

   "github.com/gin-gonic/gin"
)

func HandleUser(c *gin.Context) {
   reqId, _ := strconv.ParseInt(c.Param("id"), 10, 64)
   c.JSON(200, model.User{Id: reqId, Name: "wanghao"})
}

```
```go
package demotest

import (
	"apusic/golang-unit-test/model"
	"encoding/json"
	"io"
	"net/http/httptest"
	"testing"

	. "github.com/smartystreets/goconvey/convey"

	"github.com/gin-gonic/gin"
)

func HttpTestUtil(method, uri string, body io.Reader, router *gin.Engine) *httptest.ResponseRecorder {
	// 构造get请求
	req := httptest.NewRequest(method, uri, body)
	// 初始化响应
	w := httptest.NewRecorder()
	// 调用相应的handler接口
	router.ServeHTTP(w, req)
	return w
}

func TestHttpDemoByConvey(t *testing.T) {
	router := gin.Default()
	router.GET("/:id/getUser", HandleUser)
	Convey("http测试", t, func() {
		expected := model.User{Id: 1, Name: "wanghao"}
		response := HttpTestUtil("GET", "/1/getUser", nil, router)
		So(response.Code, ShouldEqual, 200)
		actual := model.User{}
		_ = json.Unmarshal(response.Body.Bytes(), &actual)
		So(actual, ShouldResemble, expected)
	})
}
```