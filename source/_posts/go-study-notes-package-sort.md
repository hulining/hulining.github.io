---
title: go 学习笔记之 sort 包
tags:
  - go
  - 学习笔记
categories:
  - go
abbrlink: 16328
description: '本文章主要包含 Go sort 包提供的内置类型,接口,函数方法的使用.sort 包提供了对于切片和用户定义的集合进行排序的原始函数'
---

`sort` 包提供了对于切片和用户定义的集合进行排序的原始函数,实现了 3 种基本的排序算法: 插入排序,快速排序和堆排序.它们只在 `sort` 包内部使用.用户无需考虑使用哪种排序方式. `sort` 包会根据实际数自动选择高效的排序算法.

在使用 `sort` 包对数据进行排序时,必须要求集合的元素由整数索引来枚举,那么我们只需要对该类型实现 `sort.Interface` 定义的三个方法即可.如下:

```go
type Interface interface {
    // 集合中元素的数量
    Len() int
    // 决定了如何进行排序,排序后索引为 i 的元素是否应该在 索引为 j 的元素之前
    Less(i, j int) bool
    // 交换索引为 i 和 j  的元素
    Swap(i, j int)
}
```

# 常用类型定义

```go
// sort 包中提供了三种实现了 sort.Interface 接口的类型
type Float64Slice []float64
type IntSlice []int
type StringSlice []string

type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

# 常用函数定义

```go
// 对浮点型,整型,字符串切片进行升序排序
func Float64s(a []float64)
func Ints(a []int)
func Strings(a []string)

// 判断浮点型,整型,字符串是否已经升序排序
func Float64sAreSorted(a []float64) bool
func IntsAreSorted(a []int) bool
func StringsAreSorted(a []string) bool

// 判断一个实现了 sort.Interface 接口的对象是否已经排序
func IsSorted(data Interface) bool

// 使用二分查找并返回 [0, n) 中 f(i) 为 true 的最小索引.
// 多用于查找升序排序(运算符: >=)或降序排序(运算符: <=)的数据中查找指定元素的索引
func Search(n int, f func(int) bool) int

// 在已排序切片中搜索 x,若查到,则返回查到的索引;否则返回将 x 有序插入 a 后,x 的索引
func SearchFloat64s(a []float64, x float64) int
func SearchInts(a []int, x int) int
func SearchStrings(a []string, x string) int

// 使用给定的 less 函数对 slice 进行排序. SliceStable 可保证排序是稳定的.
// 如果 slice 不是切片,则会引发 panics
func Slice(slice interface{}, less func(i, j int) bool)
func SliceStable(slice interface{}, less func(i, j int) bool)
// 测试切片是否已经使用 less 排序
func SliceIsSorted(slice interface{}, less func(i, j int) bool) bool

// 对实现了 sort.Interface  接口的类型对象进行排序. Stable 可保证排序是稳定的
func Sort(data Interface)
func Stable(data Interface)
// 对数据进行反向排序,常用于 sort.Sort(sort.Reverse(data)) 对数据进行反向排序
func Reverse(data Interface) Interface
```

# 示例

## 排序

### 使用 `sort.Sort` 或 `sort.Slice` 进行排序

```go
import (
	"fmt"
	"sort"
)

type Person struct {
	Name string
	Age  int
}

func (p Person) String() string {
	return fmt.Sprintf("%s: %d", p.Name, p.Age)
}

// 实现 sort.Interface 接口,可对 []Person 切片按照指定方式进行排序
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
// 使用 Person 的 Age 属性做比较,降序排序
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
	people := []Person{
		{"Bob", 31},
		{"John", 42},
		{"Michael", 17},
		{"Jenny", 26},
	}
	fmt.Println(people)

	// 使用 sort.Sort 方法进行排序
	sort.Sort(ByAge(people))
	fmt.Println(people)

	// 使用 sort.Slice 按指定 less 函数进行降序排序
	sort.Slice(people, func(i, j int) bool {
		return people[i].Age > people[j].Age
	})
	fmt.Println(people)
}
```

### 使用指定的属性进行排序

```go
import (
	"fmt"
	"sort"
)

type Person struct {
	Name  string
	Age   int
	Score float64
}

func (p Person) String() string {
	return fmt.Sprintf("%s: %d, %.2f", p.Name, p.Age, p.Score)
}

// 定义一个 "less" 函数类型,用于定义 Person 排序的方式
type By func(p1, p2 *Person) bool

// 排序的入口函数,传入 people 后构建 personSorter 对象,用于排序
func (by By) Sort(people []Person) {
	ps := &personSorter{
		people: people,
		by:     by, // 使用函数的闭包将 by 对象(实际上是一个函数)作为 personSorter 的成员传入,
	}
	sort.Sort(ps)
}

// personSorter 类型封装了要排序的对象和如何进行排序的函数
type personSorter struct {
	people []Person
	by     func(p1, p2 *Person) bool // Closure used in the Less method.
}

func (s *personSorter) Len() int      { return len(s.people) }
func (s *personSorter) Swap(i, j int) { s.people[i], s.people[j] = s.people[j], s.people[i] }
// Less 方法滴啊用 by 成员(by 是一个函数),决定如何进行排序
func (s *personSorter) Less(i, j int) bool {
	return s.by(&s.people[i], &s.people[j])
}

func main() {
	people := []Person{
		{"Bob", 31, 83.5},
		{"John", 42, 86.0},
		{"Michael", 17, 79.6},
		{"Jenny", 26, 89.3},
	}
	// 定义排序的方式,按照 Person 各种属性进行排序,
	name := func(p1, p2 *Person) bool {
		return p1.Name < p2.Name
	}
	age := func(p1, p2 *Person) bool {
		return p1.Age < p2.Age
	}
	score := func(p1, p2 *Person) bool {
		return p1.Score < p2.Score
	}
	// 调用 score 方法, p1,p2 调换位置,实现倒序
	decreasingScore := func(p1, p2 *Person) bool {
		return score(p2, p1)
	}

	// 按照既定的方式进行排序
	By(name).Sort(people)
	fmt.Println("By Name:", people)
	By(age).Sort(people)
	fmt.Println("By Age:", people)
	By(score).Sort(people)
	fmt.Println("By Score:", people)
	By(decreasingScore).Sort(people)
	fmt.Println("By decreasing Score:", people)
}
```

### 使用多个属性进行排序

```go
import (
	"fmt"
	"sort"
)

// 定义待排序对象
type Change struct {
	user     string
	language string
	lines    int
}

// 定以 "less" 函数,用于定义排序的方式
type lessFunc func(p1, p2 *Change) bool

// multiSorter 实现了 sort.Interface 接口,按照指定方式对 changes 属性进行排序
type multiSorter struct {
	changes []Change
	less    []lessFunc
}

func (ms *multiSorter) Len() int      { return len(ms.changes) }
func (ms *multiSorter) Swap(i, j int) { ms.changes[i], ms.changes[j] = ms.changes[j], ms.changes[i] }

// 实现 Less 方法
func (ms *multiSorter) Less(i, j int) bool {
	p, q := &ms.changes[i], &ms.changes[j]
	// 依次按照 ms 中 less 中包含的函数进行判断,不包括最后一个
	var k int
	for k = 0; k < len(ms.less)-1; k++ {
		less := ms.less[k]
		switch {
		case less(p, q):
			// p < q, so we have a decision.
			return true
		case less(q, p):
			// p > q, so we have a decision.
			return false
		}
		// p == q; 进行下次比较
	}
	// 除最后一个,之前的函数都相等,此时 k = len(ms.less)-1,则按照最后一个函数进行判断
	return ms.less[k](p, q)
}

// 排序的入口函数,调用此方法对 changes 进行排序
func (ms *multiSorter) Sort(changes []Change) {
	ms.changes = changes
	sort.Sort(ms)
}

// 返回一个 multiSorter 对象调用其 Sort 方法进行排序
// 调用 Sort 方法后,该对象按照 less 函数定义的排序方式进行排序
func OrderedBy(less ...lessFunc) *multiSorter {
	return &multiSorter{
		less: less,
	}
}

var changes = []Change{
	{"gri", "Go", 100},
	{"ken", "C", 150},
	{"glenda", "Go", 200},
	{"rsc", "Go", 200},
	{"r", "Go", 100},
	{"ken", "Go", 200},
	{"dmr", "C", 100},
	{"r", "C", 150},
	{"gri", "Smalltalk", 80},
}

func main() {

	// 定义排序的方式
	user := func(c1, c2 *Change) bool { return c1.user < c2.user }
	language := func(c1, c2 *Change) bool { return c1.language < c2.language }
	increasingLines := func(c1, c2 *Change) bool { return c1.lines < c2.lines }
	decreasingLines := func(c1, c2 *Change) bool { return increasingLines(c2, c1) }

	OrderedBy(user).Sort(changes)
	fmt.Println("By user:", changes)
	// 使用多键进行排序
	OrderedBy(user, increasingLines).Sort(changes)
	fmt.Println("By user,<lines:", changes)
	OrderedBy(user, decreasingLines).Sort(changes)
	fmt.Println("By user,>lines:", changes)
	OrderedBy(language, increasingLines).Sort(changes)
	fmt.Println("By language,<lines:", changes)
	OrderedBy(language, increasingLines, user).Sort(changes)
	fmt.Println("By language,<lines,user:", changes)
}
```

### 使用继承对数据进行排序

```go
import (
	"fmt"
	"sort"
)

type Grams int

func (g Grams) String() string { return fmt.Sprintf("%dg", int(g)) }

type Organ struct {
	Name   string
	Weight Grams
}

type Organs []*Organ

func (s Organs) Len() int      { return len(s) }
func (s Organs) Swap(i, j int) { s[i], s[j] = s[j], s[i] }

// 继承 Organs 并实现 Less 方法,从而实现 Interface 接口
type ByName struct{ Organs }

func (s ByName) Less(i, j int) bool { return s.Organs[i].Name < s.Organs[j].Name }

// 继承 Organs 并实现 Less 方法,从而实现 Interface 接口
type ByWeight struct{ Organs }

func (s ByWeight) Less(i, j int) bool { return s.Organs[i].Weight < s.Organs[j].Weight }

func main() {
	s := []*Organ{
		{"brain", 1340},
		{"heart", 290},
		{"liver", 1494},
		{"pancreas", 131},
		{"prostate", 62},
		{"spleen", 162},
	}
	sort.Sort(ByWeight{s})
	fmt.Println("Organs by weight:")
	printOrgans(s)
	sort.Sort(ByName{s})
	fmt.Println("Organs by name:")
	printOrgans(s)
}

func printOrgans(s []*Organ) {
	for _, o := range s {
		fmt.Printf("%-8s (%v)\n", o.Name, o.Weight)
	}
}
```

## 使用 `sort.Search` 查找指定元素

- 升序

```go
import (
	"fmt"
	"sort"
)

func main() {
	a := []int{1, 3, 6, 10, 15, 21, 28, 36, 45, 55}
	x := 6

	i := sort.Search(len(a), func(i int) bool { return a[i] >= x })
	if i < len(a) && a[i] == x {
		fmt.Printf("found %d at index %d in %v\n", x, i, a)
	} else {
		fmt.Printf("%d not found in %v\n", x, a)
	}
}
```

- 降序

```go
import (
	"fmt"
	"sort"
)

func main() {
	a := []int{55, 45, 36, 28, 21, 15, 10, 6, 3, 1}
	x := 6

	i := sort.Search(len(a), func(i int) bool { return a[i] <= x })
	if i < len(a) && a[i] == x {
		fmt.Printf("found %d at index %d in %v\n", x, i, a)
	} else {
		fmt.Printf("%d not found in %v\n", x, a)
	}
}
```
