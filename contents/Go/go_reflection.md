![](https://img.halfrost.com/Blog/ArticleImage/148_0.png)

# Go reflection 三定律与最佳实践

在计算机学中，反射式编程 reflective programming 或反射 reflection，是指计算机程序在运行时 runtime 可以访问、检测和修改它本身状态或行为的一种能力。用比喻来说，反射就是程序在运行的时候能够“观察”并且修改自己的行为。

> Wikipedia: In computer science, reflective programming or reflection is the ability of a process to examine, introspect, and modify its own structure and behavior.


“反射”和“内省”（type introspection）在概念上有区别。内省（或称“自省”）机制仅指程序在运行时对自身信息（称为元数据）的检测；反射机制不仅包括要能在运行时对程序自身信息进行检测，还要求程序能进一步根据这些信息改变程序状态或结构。所以反射的概念范畴要大于内省。


在类型检测严格的面向对象的编程语言如 Java 中，一般需要在编译期间对程序中需要调用的对象的具体类型、接口（interface）、字段（fields）和方法的合法性进行检查。反射技术则允许将对需要调用的对象的消息检查工作从编译期间推迟到运行期间再现场执行。这样一来，可以在编译期间先不明确目标对象的接口（interface）名称、字段（fields），即对象的成员变量、可用方法，然后在运行根据目标对象自身的消息决定如何处理。它还允许根据判断结果进行实例化新对象和相关方法的调用。

反射主要用途就是使给定的程序，动态地适应不同的运行情况。利用面向对象建模中的多态（多态性）也可以简化编写分别适用于多种不同情形的功能代码，但是反射可以解决多态（多态性）并不适用的更普遍情形，从而更大程度地避免硬编码（即把代码的细节“写死”，缺乏灵活性）的代码风格。

**反射也是元编程的一个关键策略**。

最常见的代码如下：

```go
import "reflect"

func main() {
	// Without reflection
	f := Foo{}
	f.Hello()

	// With reflection
	fT := reflect.TypeOf(Foo{})
	fV := reflect.New(fT)

	m := fV.MethodByName("Hello")
	if m.IsValid() {
		m.Call(nil)
	}
}
```

反射看似代码更加复杂，但是能实现的功能更加灵活了。究竟什么时候用反射？最佳实践是什么？这篇文章好好讨论一下。


## 一. 基本数据结构和方法

在上一篇 Go interface 中，可以了解到普通对象在内存中的存在形式，一个变量值得我们关注的无非是两部分，一个是类型，一个是它存的值。变量的类型决定了底层 tpye 是什么，支持哪些方法集。值无非就是读和写。去内存里面哪里读，把 0101 写到内存的哪里，都是由类型决定的。这一点在解析不同 Json 数据结构的时候深有体会，如果数据类型用错了，解析出来得到的变量的值是乱码。Go 提供反射的功能，是为了支持在运行时动态访问变量的类型和值。

在运行时想要动态访问类型的值，必然应用程序存储了所有用到的类型信息。"reflect" 库提供了一套供开发者使用的访问接口。Go 中反射的基础是接口和类型，Go 很巧妙的借助了对象到接口的转换时使用的数据结构，先将对象传递给内部的空接口，即将类型转换成空接口 eface。然后反射再基于这个 eface 来访问和操作实例对象的值和类型。

那么笔者就从数据结构开始梳理 Go 是如何实现反射的。在 reflect 包中，有一个描述类型公共信息的通用数据结构 rtype。从源码的注释上看，它和 interface 里面的 \_type 是同一个数据结构。它们俩只是因为包隔离，加上为了避免循环引用，所以在这边又复制了一遍。


```go
// rtype is the common implementation of most values.
// It is embedded in other struct types.
//
// rtype must be kept in sync with ../runtime/type.go:/^type._type.
type rtype struct {
	size       uintptr // 类型占用内存大小
	ptrdata    uintptr // 包含所有指针的内存前缀大小
	hash       uint32  // 类型 hash
	tflag      tflag   // 标记位，主要用于反射
	align      uint8   // 对齐字节信息
	fieldAlign uint8   // 当前结构字段的对齐字节数
	kind       uint8   // 基础类型枚举值
	equal func(unsafe.Pointer, unsafe.Pointer) bool // 比较两个形参对应对象的类型是否相等
	gcdata    *byte    // GC 类型的数据
	str       nameOff  // 类型名称字符串在二进制文件段中的偏移量
	ptrToThis typeOff  // 类型元信息指针在二进制文件段中的偏移量
}
```

相同的，所有类型的元信息也都复制了一遍：

```go
type arraytype struct {
	typ   _type
	elem  *_type
	slice *_type
	len   uintptr
}

type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}
```

所有基础类型都不再赘述，详情可见上一篇《深入研究 Go interface 底层实现》。在 reflect 包中有一个重要的方法 TypeOf()，利用这个方法可以获得一个 Type 的 interface。通过 Type interface 可以获取对象的类型信息。

```go
// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}

// toType converts from a *rtype to a Type that can be returned
// to the client of package reflect. In gc, the only concern is that
// a nil *rtype must be replaced by a nil Type, but in gccgo this
// function takes care of ensuring that multiple *rtype for the same
// type are coalesced into a single Type.
func toType(t *rtype) Type {
	if t == nil {
		return nil
	}
	return t
}
```

上述方法实现非常简单，就是将形参转换成 Type interface。这里设计成返回 interface 而不是返回 rtype 类型的数据结构是有讲究的。一是设计者不希望调用者拿到 rtype 滥用。毕竟类型信息这些都是只读的，在运行时被任意篡改太不安全了。二是设计者将调用者的需求的所有需求用 interface 这一层屏蔽了，Type interface 下层可以对应很多种类型，利用这个接口统一抽象成一层。

值得说明的一点是 TypeOf() 入参，入参类型是 i interface{}，可以是 2 种类型，一种是 interface 变量，另外一种是具体的类型变量。如果 i 是具体的类型变量，TypeOf() 返回的具体类型信息；如果 i 是 interface 变量，并且绑定了具体类型对象实例，返回的是 i 绑定具体类型的动态类型信息；如果 i 没有绑定任何具体的类型对象实例，返回的是接口自身的静态类型信息。例如下面这段代码：


```go
import (
	"fmt"
	"reflect"
)

func main() {
	ifa := new(Person)
	var ifb Person = Student{name: "halfrost"}
    // 未绑定具体变量的接口类型 
	fmt.Println(reflect.TypeOf(ifa).Elem().Name())
	fmt.Println(reflect.TypeOf(ifa).Elem().Kind().String())
    // 绑定具体变量的接口类型 
	fmt.Println(reflect.TypeOf(ifb).Name())
	fmt.Println(reflect.TypeOf(ifb).Kind().String())
}
```

在第一组输出中，reflect.TypeOf() 入参未绑定具体变量的接口类型，所以返回的是接口类型本身 Person。对应的 Kind 是 interface。在第二组输出中，reflect.TypeOf() 入参绑定了具体变量的接口类型，所以返回的是绑定的具体类型 Student。对应的 Kind 是 struct。

```go
Person
interface

Student
struct
```


下面来看看 Type interface 究竟涵盖了哪些有用的方法：


### 1. reflect.Type 通用方法

以下这些方法是通用方法，可以适用于任何类型。

```go
// Type 是 Go 类型的表示。
//
// 并非所有方法都适用于所有类型。
// 在调用 kind 具体方法之前，先使用 Kind 方法找出类型的种类。因为调用一个方法如果类型不匹配会导致 panic
//
// Type 类型值是可以比较的，比如用 == 操作符。所以它可以用做 map 的 key
// 如果两个 Type 值代表相同的类型，那么它们一定是相等的。
type Type interface {
	
	// Align 返回该类型在内存中分配时，以字节数为单位的字节数
	Align() int
	
	// FieldAlign 返回该类型在结构中作为字段使用时，以字节数为单位的字节数
	FieldAlign() int
	
	// Method 这个方法返回类型方法集中的第 i 个方法。
	// 如果 i 不在[0, NumMethod()]范围内，就会 panic。
	// 对于一个非接口类型 T 或 *T，返回的 Method 的 Type 和 Func。
	// fields 字段描述一个函数，它的第一个参数是接收方，而且只有导出的方法可以访问。
	// 对于一个接口类型，返回的 Method 的 Type 字段给出的是方法签名，没有接收者，Func字段为nil。
	// 方法是按字典序顺序排列的。
	Method(int) Method
	
	// MethodByName 返回类型中带有该名称的方法。
	// 方法集和一个表示是否找到该方法的布尔值。
	// 对于一个非接口类型 T 或 *T，返回的 Method 的 Type 和 Func。
	// fields 字段描述一个函数，其第一个参数是接收方。
	// 对于一个接口类型，返回的 Method 的 Type 字段给出的是方法签名，没有接收者，Func字段为nil。
	MethodByName(string) (Method, bool)

	// NumMethod 返回使用 Method 可以访问的方法数量。
	// 请注意，NumMethod 只在接口类型的调用的时候，会对未导出方法进行计数。
	NumMethod() int

	// 对于定义的类型，Name 返回其包中的类型名称。
	// 对于其他(非定义的)类型，它返回空字符串。
	Name() string

	// PkgPath 返回一个定义类型的包的路径，也就是导入路径，导入路径是唯一标识包的类型，如 "encoding/base64"。
	// 如果类型是预先声明的(string, error)或者没有定义(*T, struct{}, []int，或 A，其中 A 是一个非定义类型的别名），包的路径将是空字符串。
	PkgPath() string

	// Size 返回存储给定类型的值所需的字节数。它类似于 unsafe.Sizeof.
	Size() uintptr

	// String 返回该类型的字符串表示。
	// 字符串表示法可以使用缩短的包名。
	// (例如，使用 base64 而不是 "encoding/base64")并且它并不能保证类型之间是唯一的。如果是为了测试类型标识，应该直接比较类型 Type。
	String() string

	// Kind 返回该类型的具体种类。
	Kind() Kind

	// Implements 表示该类型是否实现了接口类型 u。
	Implements(u Type) bool

	// AssignableTo 表示该类型的值是否可以分配给类型 u。
	AssignableTo(u Type) bool

	// ConvertibleTo 表示该类型的值是否可转换为 u 类型。
	ConvertibleTo(u Type) bool

	// Comparable 表示该类型的值是否具有可比性。
	Comparable() bool
}
```

### 2. reflect.Type 专有方法

以下这些方法是某些类型专有的方法，如果类型不匹配会发生 panic。在不确定类型之前最好先调用 Kind() 方法确定具体类型再调用类型的专有方法。


|Kind|Methods applicable|
|:------|:-----------|
|Int\*| Bits |
|Uint\* | Bits |
|Float\*| Bits |
|Complex\*| Bits |
|Array|Elem, Len|
|Chan|ChanDir, Elem|
|Func|In, NumIn, Out, NumOut, IsVariadic|
|Map| Key, Elem|
|Ptr| Elem|
|Slice| Elem|
|Struct| Field, FieldByIndex, FieldByName,FieldByNameFunc, NumField|


对专有方法的说明如下：


```go
type Type interface {

	// Bits 以 bits 为单位返回类型的大小。
	// 如果类型的 Kind 不属于：sized 或者 unsized Int, Uint, Float, 或者 Complex，会 panic。
	//大小不一的Int、Uint、Float或Complex类型。
	Bits() int

	// ChanDir 返回一个通道类型的方向。
	// 如果类型的 Kind 不是 Chan，会 panic。
	ChanDir() ChanDir


	// IsVariadic 表示一个函数类型的最终输入参数是否为一个 "..." 可变参数。如果是，t.In(t.NumIn() - 1) 返回参数的隐式实际类型 []T.
	// 更具体的，如果 t 代表 func(x int, y ... float64)，那么：
	// t.NumIn() == 2
	// t.In(0)是 "int" 的 reflect.Type 反射类型。
	// t.In(1)是 "[]float64" 的 reflect.Type 反射类型。
	// t.IsVariadic() == true
	// 如果类型的 Kind 不是 Func.IsVariadic，IsVariadic 会 panic
	IsVariadic() bool

	// Elem 返回一个 type 的元素类型。
	// 如果类型的 Kind 不是 Array、Chan、Map、Ptr 或 Slice，就会 panic
	Elem() Type

	// Field 返回一个结构类型的第 i 个字段。
	// 如果类型的 Kind 不是 Struct，就会 panic。
	// 如果 i 不在 [0, NumField()] 范围内，也会 panic。
	Field(i int) StructField

	// FieldByIndex 返回索引序列对应的嵌套字段。它相当于对每一个 index 调用 Field。
	// 如果类型的 Kind 不是 Struct，就会 panic。
	FieldByIndex(index []int) StructField

	// FieldByName 返回给定名称的结构字段和一个表示是否找到该字段的布尔值。
	FieldByName(name string) (StructField, bool)

	// FieldByNameFunc 返回一个能满足 match 函数的带有名称的 field 字段。布尔值表示是否找到。
	// FieldByNameFunc 先在自己的结构体的字段里面查找，然后在任何嵌入结构中的字段中查找，按广度第一顺序搜索。最终停止在含有一个或多个能满足 match 函数的结构体中。如果在该深度上满足条件的有多个字段，这些字段相互取消，并且 FieldByNameFunc 返回没有匹配。
	// 这种行为反映了 Go 在包含嵌入式字段的结构的情况下对名称查找的处理方式
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// In 返回函数类型的第 i 个输入参数的类型。
	// 如果类型的 Kind 不是 Func 类型会 panic。
	// 如果 i 不在 [0, NumIn()) 的范围内，会 panic。
	In(i int) Type

	// Key 返回一个 map 类型的 key 类型。
	// 如果类型的 Kind 不是 Map，会 panic。
	Key() Type

	// Len 返回一个数组类型的长度。
	// 如果类型的 Kind 不是 Array，会 panic。
	Len() int

	// NumField 返回一个结构类型的字段数目。
	// 如果类型的 Kind 不是 Struct，会 panic。
	NumField() int

	// NumIn 返回一个函数类型的输入参数数。
	// 如果类型的 Kind 不是Func.NumIn()，会 panic。
	NumIn() int

	// NumOut 返回一个函数类型的输出参数数。
	// 如果类型的 Kind 不是 Func.NumOut()，会 panic。
	NumOut() int

	// Out 返回一个函数类型的第 i 个输出参数的类型。
	// 如果类型的类型不是 Func.Out，会 panic。
	// 如果 i 不在 [0, NumOut()) 的范围内，会 panic。
	Out(i int) Type

	common() *rtype
	uncommon() *uncommonType
}
```

### 3. reflect.Value 数据结构

在 reflect 包中，并非所有的方法都适用于所有类型的值。具体的限制在方法说明注释里面有写。在调用特定种类的方法之前，最好使用 Kind 方法找出 Value 的种类。和 reflect.Type 一样，调用类型不匹配的方法会导致 panic。需要特殊说明的是 zero Value，zero Value 代表没有值。它的 IsValid() 方法返回 false，Kind() 方法返回 Invalid，String() 方法返回 “<invalid Value>”，而剩下的所有其他方法均会 panic。大多数函数和方法从不返回 invalid value。如果确实返回了 invalid value，则其文档会明确说明特殊条件。
	
一个 Value 可以由多个 goroutine 并发使用，前提是底层的 Go 值可以同时用于等效的直接操作。 要比较两个 Value，请比较 Interface 相关方法的结果。 在两个 Value 上使用 ==，并不会比较它们表示的底层的值。	

reflect 包里的 Value 很简单，数据结构如下：

```go
type Value struct {
	// typ 包含由值表示的值的类型。
	typ *rtype

	// 指向值的指针，如果设置了 flagIndir，则是指向数据的指针。只有当设置了 flagIndir 或 typ.pointers（）为 true 时有效。
	ptr unsafe.Pointer

	// flag 保存有关该值的元数据。最低位是标志位：
	//	- flagStickyRO: 通过未导出的未嵌入字段获取，因此为只读
	//	- flagEmbedRO:  通过未导出的嵌入式字段获取，因此为只读
	//	- flagIndir:    val保存指向数据的指针
	//	- flagAddr:     v.CanAddr 为 true (表示 flagIndir)
	//	- flagMethod:   v 是方法值。
    // 接下来的 5 个 bits 给出 Value 的 Kind 种类，除了方法 values 以外，它会重复 typ.Kind（）。其余 23 位以上给出方法 values 的方法编号。如果 flag.kind（）!= Func，代码可以假定 flagMethod 没有设置。如果 ifaceIndir(typ)，代码可以假定设置了 flagIndir。
	flag
}
```

一个方法的 Value 表示一个相关方法的调用，就像一些方法接收者 r 调用 r.Read。typ + val + flag bits 位描述了接收者r，但是 Kind 标记位表示 Func（方法是函数），并且该标志的高位给出 r 的类型的方法集中的方法编号。



## 二. 反射的内部实现



## 三. 反射三定律

著名的 [《The laws of Reflection》](https://blog.golang.org/laws-of-reflection) 这篇文章里面归纳了反射的三定律。

### 1. 从接口值中得到反射对象


![](https://img.halfrost.com/Blog/ArticleImage/148_1.png)


- 通过实例获取 Value 对象，使用 reflect.ValueOf() 函数。

```go
// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}
	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}
```

- 通过实例获取反射对象 Type，使用 reflect.TypeOf() 函数。


```go
// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```

### 2. 从反射对象中获得接口值

从 reflect.Value 数据结构可知，它包含了类型和值的信息，所以将 Value 转换成实例对象很容易。

![](https://img.halfrost.com/Blog/ArticleImage/148_2.png)


- 将 Value 转换成空的 interface，内部存放具体类型实例。使用 interface() 函数。

```go
// Interface returns v's current value as an interface{}.
// It is equivalent to:
//	var i interface{} = (v's underlying value)
// It panics if the Value was obtained by accessing
// unexported struct fields.
func (v Value) Interface() (i interface{}) {
	return valueInterface(v, true)
}
```

- Value 也包含很多成员方法，可以将 Value 转换成简单类型实例，注意如果类型不匹配会 panic。

```go
// Int returns v's underlying value, as an int64.
// It panics if v's Kind is not Int, Int8, Int16, Int32, or Int64.
func (v Value) Int() int64 {
	k := v.kind()
	p := v.ptr
	switch k {
	case Int:
		return int64(*(*int)(p))
	case Int8:
		return int64(*(*int8)(p))
	case Int16:
		return int64(*(*int16)(p))
	case Int32:
		return int64(*(*int32)(p))
	case Int64:
		return *(*int64)(p)
	}
	panic(&ValueError{"reflect.Value.Int", v.kind()})
}

// Uint returns v's underlying value, as a uint64.
// It panics if v's Kind is not Uint, Uintptr, Uint8, Uint16, Uint32, or Uint64.
func (v Value) Uint() uint64 {
	k := v.kind()
	p := v.ptr
	switch k {
	case Uint:
		return uint64(*(*uint)(p))
	case Uint8:
		return uint64(*(*uint8)(p))
	case Uint16:
		return uint64(*(*uint16)(p))
	case Uint32:
		return uint64(*(*uint32)(p))
	case Uint64:
		return *(*uint64)(p)
	case Uintptr:
		return uint64(*(*uintptr)(p))
	}
	panic(&ValueError{"reflect.Value.Uint", v.kind()})
}

// Bool returns v's underlying value.
// It panics if v's kind is not Bool.
func (v Value) Bool() bool {
	v.mustBe(Bool)
	return *(*bool)(v.ptr)
}

// Float returns v's underlying value, as a float64.
// It panics if v's Kind is not Float32 or Float64
func (v Value) Float() float64 {
	k := v.kind()
	switch k {
	case Float32:
		return float64(*(*float32)(v.ptr))
	case Float64:
		return *(*float64)(v.ptr)
	}
	panic(&ValueError{"reflect.Value.Float", v.kind()})
}
```

### 3. 要修改反射对象，其值必须可以修改



![](https://img.halfrost.com/Blog/ArticleImage/148_3.png)

- 指针类型 Type 转成值类型 Type。指针类型必须是 \*Array、\*Slice、\*Pointer、\*Map、\*Chan 类型，否则会发生 panic。Type 返回的是内部元素的 Type。

```go
// Elem returns element type of array a.
func (a *Array) Elem() Type { return a.elem }

// Elem returns the element type of slice s.
func (s *Slice) Elem() Type { return s.elem }

// Elem returns the element type for the given pointer p.
func (p *Pointer) Elem() Type { return p.base }

// Elem returns the element type of map m.
func (m *Map) Elem() Type { return m.elem }

// Elem returns the element type of channel c.
func (c *Chan) Elem() Type { return c.elem }
```

- 值类型 Type 转成指针类型 Type。PtrTo 返回的是指向 t 的指针类型 Type。

```go
// PtrTo returns the pointer type with element t.
// For example, if t represents type Foo, PtrTo(t) represents *Foo.
func PtrTo(t Type) Type {
	return t.(*rtype).ptrTo()
}
```

### 4. Type 和 Value 相互转换

![](https://img.halfrost.com/Blog/ArticleImage/148_4.png)


### 5. Value 指针转换成值

![](https://img.halfrost.com/Blog/ArticleImage/148_5.png)

### 6. 总结

![](https://img.halfrost.com/Blog/ArticleImage/148_6.png)

## 四. 优缺点与最佳实践

优点：  
支持反射的语言提供了一些在早期高级语言中难以实现的运行时特性。

- 可以在一定程度上避免硬编码，提供灵活性和通用性。
- 可以作为一个第一类对象发现并修改源代码的结构（如代码块、类、方法、协议等）。
- 可以在运行时像对待源代码语句一样动态解析字符串中可执行的代码（类似JavaScript的eval()函数），进而可将跟class或function匹配的字符串转换成class或function的调用或引用。
- 可以创建一个新的语言字节码解释器来给编程结构一个新的意义或用途。

劣势：

- 此技术的学习成本高。面向反射的编程需要较多的高级知识，包括框架、关系映射和对象交互，以实现更通用的代码执行。  
- 同样因为反射的概念和语法都比较抽象，过多地滥用反射技术会使得代码难以被其他人读懂，不利于合作与交流。
- 由于将部分信息检查工作从编译期推迟到了运行期，此举在提高了代码灵活性的同时，牺牲了一点点运行效率。

通过深入学习反射的特性和技巧，它的劣势可以尽量避免，但这需要许多时间和经验的积累。
