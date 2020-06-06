---
title: go 学习笔记之反射
date: 2020/04/30
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 15536
description: '记录在学习 Go 过程中容易出错, 容易忘记的知识点, 主要包含 Go 中反射相关使用, 包括 reflect.Type 和 reflect.Value'
---

反射是值在程序运行期间对程序本身进行访问和修改的能力. 程序在编译时, 变量被转换为内存地址, 程序运行时, 程序无法获取自身的信息.

支持反射的语言可以在程序编译期将变量的反射信息, 如字段名称, 类型信息, 结构体信息等整合到可执行文件中, 并给程序提供接口访问反射信息, 这样就可以在程序运行期获取类型的反射信息, 并且有能力修改它们.


任意接口值在反射中都可以理解为由 Type 和 Value 组成的. Go 语言的反射是由 `reflect` 包提供的, 它定义了两个重要的类型 `reflect.Type` 和 `reflect.Value`, 并提供了 `reflect.TypeOf` 和 `reflect.ValueOf` 函数来获取任意对象的 Value 和 Type.

## 反射的类型接口 `reflect.Type`

在 Go 语言程序中, 使用 `reflect.TypeOf()` 函数可以获得任意值的 `reflect.Type` 接口对象, 程序通过类型对象可以访问对象的类型信息.

```go
import (
    "fmt"
    "reflect"
)

func main() {
    var n int = 20
    typeOfn := reflect.TypeOf(n)  // 获取 n 的 反射的类型对象 `reflect.Type`
    fmt.Println("type:", typeOfn.Name(), "kind:", typeOfn.Kind())  // 输出为 type: int kind: int
}
```

### 类型名称(Name)与种类(Kind)

- Name(类型名称) 返回反射类型的的名称, 包括 Go 语言中原生数据类型及通过 type 关键字自定义的数据类型. 获取方式为 `reflect.Type` 的 `Name()` 方法
- Kind(种类) 指反射类型所属的种类, 它仅包含 Go 语言中原生数据种类. 获取方式为 `reflect.Type` 的 `Kind()` 方法, 返回 `reflect.Kind` 类型的常量

在如下示例中, `stu` 对象的反射类型名称为自定义的 `Student`, 而其所属的反射种类为 `struct`.

```go
import (
    "fmt"
    "reflect"
)

type Student struct {
	Name  string
	Age   int
}

func main() {
    stu := Student{
    	Name:  "tom",
    	Age:   20,
    }
    typeOfstu := reflect.TypeOf(stu)
    fmt.Println("type:", typeOfstu.Name(), "kind:", typeOfstu.Kind())
}
// 输出
// type: Student kind: struct
```

### 特殊反射种类的类型名称

获取反射种类(Kind)为 `Array`, `Chan`, `Map`, `Ptr` 或 `Slice` 的对象的反射类型名称时, 通过 `Name()` 方法获得的类型名称为空字符串("")

要想获取以上对象的反射类型名称, 需要通过 `Elem()` 方法获取反射类型的元素类型, 然后再通过 `Name()` 方法获取其反射类型的名称

```go
import (
    "fmt"
    "reflect"
)

type Student struct {
	Name  string
	Age   int
}

func main() {
    stu := Student{
    	Name:  "tom",
    	Age:   20,
    }
    ptrOfstu := &stu
    a:=[3]int{1,3,4}
    typeOfptr := reflect.TypeOf(a)  // 获取 &stu 的反射的类型对象 `reflect.Type`
    fmt.Println("type:", typeOfptr.Name(), "kind:", typeOfptr.Kind())  // 输出为 type:  kind: ptr
    elemType := typeOfptr.Elem()
    fmt.Println("elem's type:", elem.Name(), "elem's kind:", elem.Kind())  // 输出为 elem's type: Student elem's kind: struct
}
// 输出
// type:  kind: ptr
// type: Student kind: struct
```

`reflect.Type` 的 `Elem()` 方法仅当反射类型的种类是 `Array`, `Chan`, `Map`, `Ptr` 或 `Slice` 时才可以获取其元素类型, 否则 `Elem()` 方法会引发 panics.


### 使用反射获取结构体的成员及方法信息

通过 `reflect.TypeOf()` 获得反射对象信息后, 如果它所属种类是结构体，可通过 `reflect.Type` 的 `NumField()` 和 `Field()` 方法获得结构体成员的详细信息, 可通过`NumMethod()` 和 `Method()` 方法获取结构体的方法详细信息

`reflect.Type` 中与成员及方法获取相关的方法如下表所示

方法 | 参数 | 返回值 |说明
:---: | :---: | :---: | :---:
`NumField()` | - | int | 返回结构体成员字段数量
`Field(i int)` | 字段索引,从 0 开始 | reflect.StructField | 根据索引返回索引对应的结构体字段的信息
`NumMethod()` | - | int | 返回结构体方法数量
`Method(i int)` | 字段索引,从 0 开始 | reflect.Method | 根据索引返回索引对应的结构体方法的信息, 方法的排序是根据 ascii 码的先后顺序进行排序的

需要注意的是, 方法相关信息可直接通过 `reflect.TypeOf(&object).Method()` 方法获取, 且通过这种方法获取的方法包括 receiver 为 `&object` 的方法. `reflect.TypeOf(object).Method()` 仅返回 receiver 为 `object` 的方法. 而字段相关信息只能通过 `Elem()` 获取元素类型后获取字段相关信息

```go
import (
	"fmt"
	"reflect"
)

type Student struct {
	Name  string  `json: "name"`
	Age   int     `json: "age"`
}

func (stu Student) GetSum(n1, n2 int) int {
    return n1 + n2
}

func (stu *Student) Print() {
	fmt.Println(*stu)
}

func (stu *Student) Set(name string, age int) {
	stu.Name = name
	stu.Age = age
}

func main() {
	stu := Student{
		Name: "tom",
		Age:  20,
	}
	typeOfstu := reflect.TypeOf(stu)
	kindOfstu := typeOfstu.Kind()
	typeOfptr := reflect.TypeOf(&stu)

	if kindOfstu != reflect.Struct {
		fmt.Println("except struct...")
		return
	}

	fieldNum := typeOfstu.NumField()
	fmt.Println("字段个数为: ", fieldNum)  // 2
	for i := 0; i < fieldNum; i++ {
		field := typeOfstu.Field(i)
		fmt.Println(field)
		fmt.Printf("字段名: %v, 字段类型: %v, 字段的 json 标签: %v\n", field.Name, field.Type, field.Tag.Get("json"))  // Tag.Get(key) 方法可以返回键值对格式的的标签指定键的值
	}

	methodNumOfstu := typeOfstu.NumMethod()
	fmt.Println("stu方法个数为: ", methodNumOfstu)  // 1
	for i := 0; i < methodNumOfstu; i++ {
		method := typeOfstu.Method(i)
		fmt.Println(method)
		fmt.Printf("方法名: %v, 方法类型: %v\n", method.Name, method.Type)
	}

	methodNumOfptr := typeOfptr.NumMethod()
	fmt.Println("&stu方法个数为: ", methodNumOfptr)   // 3
	for i := 0; i < methodNumOfptr; i++ {
		method := typeOfptr.Method(i)
		fmt.Println(method)
		fmt.Printf("方法名: %v, 方法类型: %v, 方法对象\n", method.Name, method.Type, method.Func)
	}
}
```

## 反射的值对象 `reflect.Value`

使用 `reflect.ValueOf()` 函数可以获得对象的反射值对象 `reflect.Value`. `reflect.Value` 对象可以通过调用 `Type()` 方法返回 `reflect.Type` 对象, 完成值到类型对象的转换.

```go
import (
    "fmt"
    "reflect"
)

func main() {
    var n int = 20
    valueOfN := reflect.ValueOf(n)  // 获取 n 的 反射的值对象 `reflect.Value`
    typeOfN := valueOfN.Type() // 值对象可以转换为类型对象, 等价于 reflect.TypeOf(n)
    fmt.Printf("值为: %v, 实际类型为 %T", valueOfN, valueOfN)  // 输出为 值为: 20, 实际类型为 reflect.Value
    n2, ok := valueOfN.Interface().(int) // 先将 reflect.Value 转换为 interface, 然后通过类型断言来转换为 int 类型
    if ok {
    	fmt.Printf("转换后值为: %v, 类型为 %T", n2, n2)  // 输出 转换后值为: 20, 类型为 int
    }
}
```

![实际类型与反射间的转换](https://raw.githubusercontent.com/hulining/hulining.github.io/hexo/source/_posts/images/go-study-notes-reflect/conversion_between_actual_type_and_reflection.jpg)

通过以上可以看得出, 实际类型对象将自身通过接口方式传入 `reflect.ValueOf(i interface{})` 方法, 返回 `reflect.Value` 对象. `reflect.Value` 对象通过调用自身 `Interface()` 方法, 返回接口类型, 该接口类型可通过类型断言转化为实际类型对象.

### 使用反射获取结构体的成员值及方法信息

通过 `reflect.ValueOf()` 获得反射对象值信息后, 如果它所属种类是结构体，可通过 `reflect.Value` 的 `NumField()` 和 `Field()` 方法获得结构体成员的值信息, 可通过`NumMethod()` 和 `Method()` 方法获取结构体的方法等信息

`reflect.Value` 中与成员及方法获取相关的方法如下表所示:

方法 | 参数 | 返回值 |说明
:---: | :---: | :---: | :---:
`NumField()` | - | int | 返回结构体成员字段数量
`Field(i int)` | 字段索引,从 0 开始 | reflect.Value | 根据索引返回结构体字段对应的值对象
`NumMethod()` | - | int | 返回结构体方法数量
`Method(i int)` | 字段索引,从 0 开始 | reflect.Value | 根据索引返回结构体方法对应的值对象, 输出为内存地址(尚不清楚是什么地址), 方法的排序是根据 ascii 码的先后顺序进行排序的

需要注意的是, 方法相关信息可直接通过 `reflect.ValueOf(&object).Method()` 方法获取, 且通过这种方法获取的方法包括 receiver 为 `&object` 的方法. `reflect.ValueOf(object).Method()` 仅返回 receiver 为 `object` 的方法. 而字段值对象相关信息只能通过 `Elem()` 获取值元素后获取字段段值对象相关信息

```go
import (
	"fmt"
	"reflect"
)

type Student struct {
	Name  string  `json: "name"`
	Age   int     `json: "age"`
}

func (stu Student) GetSum(n1, n2 int) int {
    return n1 + n2
}

func (stu *Student) Print() {
	fmt.Println(*stu)
}

func (stu *Student) Set(name string, age int) {
	stu.Name = name
	stu.Age = age
}

func main() {
	stu := Student{
		Name: "tom",
		Age:  20,
	}
	valueOfstu := reflect.ValueOf(stu)
	kindOfstu := valueOfstu.Kind()
	valueOfptr := reflect.ValueOf(&stu)
    typeOfStu := valueOfstu.Type()  // 返回反射的类型

	if kindOfstu != reflect.Struct {
		fmt.Println("except struct...")
		return
	}

	fieldNum := typeOfstu.NumField()
	fmt.Println("字段个数为: ", fieldNum)  // 2
	for i := 0; i < fieldNum; i++ {
		field := typeOfstu.Field(i)
		fmt.Println(field)  // 直接输出各个字段的值
        
	}

	methodNumOfstu := typeOfstu.NumMethod()
	fmt.Println("stu方法个数为: ", methodNumOfstu)  // 1
	for i := 0; i < methodNumOfstu; i++ {
		method := typeOfstu.Method(i)
		fmt.Println(method)  // 直接输出一个形如 0x4878c0 的地址, 但是不确定是什么, 而且所有都是相同的
	}

	methodNumOfptr := typeOfptr.NumMethod()
	fmt.Println("&stu方法个数为: ", methodNumOfptr)   // 3
	for i := 0; i < methodNumOfptr; i++ {
		method := typeOfptr.Method(i)
		fmt.Println(method) // 直接输出一个形如 0x4878c0 的地址, 但是不确定是什么, 而且所有都是相同的
	}
}
```

### 使用反射调用结构体的方法

继续看上一个例子, 通过反射如何调用 `Student` 中的方法呢?

```go
import (
	"fmt"
	"reflect"
)

type Student struct {
	Name  string  `json: "name"`
	Age   int     `json: "age"`
}

func (stu Student) GetSum(n1, n2 int) int {
    return n1 + n2
}

func (stu *Student) Print() {
	fmt.Println(*stu)
}

func (stu *Student) Set(name string, age int) {
	stu.Name = name
	stu.Age = age
}

func main() {
	stu := Student{
		Name: "tom",
		Age:  20,
	}
	valueOfStu := reflect.ValueOf(&stu)  // 需要注意这里传入的是 &stu, 使反射能够调用所有的方法, 并修改原始对象的成员变量

    // 使用 reflect.Value 的 Call() 方法反射调用 GetSum(n1, n2 int) int 方法, Call()方法需传入 []reflect.Value 类型的值
	var paramsGetSum []reflect.Value
	paramsGetSum = append(paramsGetSum, reflect.ValueOf(10))
	paramsGetSum = append(paramsGetSum, reflect.ValueOf(20))
	resGetSum := valueOfStu.Method(0).Call(paramsGetSum)  // 调用后返回值为 []reflect.Value, 因此需要根据返回值个数使用索引来获取返回值
    fmt.Printf("调用结果为 %v, 类型为 %T", resGetSum, resGetSum)  // 输出为 :调用结果为 30, 类型为 []reflect.Value
    resGetSum0 := resGetSum[0]
    fmt.Printf("第一个返回值为 %v, 类型为 %T\n", resGetSum0, resGetSum0)  // 输出为 :第一个返回值为 30, 类型为 reflect.Value
	realRes0 := resGetSum[0].Interface().(int)
	fmt.Printf("实际结果为 %v, 类型为 %T\n", realRes, realRes)  // 输出为 :实际结果为 30, 类型为 int
    
    // 使用 reflect.Value 的 Call() 方法反射调用 Print() 方法, Call()方法需传入 []reflect.Value 类型的值
    var paramsPrint []reflect.Value
	resPrint := valueOfStu.Method(1).Call(paramsPrint)
	fmt.Printf("调用结果为 %v, 类型为 %T", resPrint, resPrint)  // 输出为 :调用结果为 [], 类型为 []reflect.Value
    
    // 使用 reflect.Value 的 Call() 方法反射调用 Set(name string, age int) 方法, Call()方法需传入 []reflect.Value 类型的值
    var paramsSet []reflect.Value
	paramsSet = append(paramsSet, reflect.ValueOf("jack"))
	paramsSet = append(paramsSet, reflect.ValueOf(30))
	resSet := valueOfStu.Method(2).Call(paramsSet)
	fmt.Printf("调用结果为 %v, 类型为 %T\n", resSet, resSet) // 输出为 :调用结果为 [], 类型为 []reflect.Value
	fmt.Println(stu)  // 可以看到 stu 已被修改为 {jack 30}
}
```

## 其它类型结构体

### StructField 结构体字段

```go
type StructField struct {
    Name string          // 字段名
    PkgPath string       // 字段路径
    Type      Type       // 字段反射类型对象
    Tag       StructTag  // 字段的结构体标签
    Offset    uintptr    // 字段在结构体中的相对偏移
    Index     []int      // Type.FieldByIndex中的返回的索引值
    Anonymous bool       // 是否为匿名字段
}
type StructTag string
```

### Method 方法

```go
type Method struct {
	Name    string // 方法名
	PkgPath string // 方法路径
	Type    Type   // 方法类型
	Func    Value  // 以接收者为第一个参数的 func
	Index   int    // Type.Method 的索引
}
```