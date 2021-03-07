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


## 一. 基本数据结构

在上一篇 Go interface 中，可以了解到普通对象在内存中的存在形式，一个变量值得我们关注的无非是两部分，一个是类型，一个是它存的值。变量的类型决定了底层 tpye 是什么，支持哪些方法集。值无非就是读和写。去内存里面哪里读，把 0101 写到内存的哪里，都是由类型决定的。这一点在解析不同 Json 数据结构的时候深有体会，如果数据类型用错了，解析出来得到的变量的值是乱码。Go 提供反射的功能，是为了支持在运行时动态访问变量的类型和值。

在运行时想要动态访问类型的值，必然应用程序存储了所有用到的类型信息。"reflect" 库提供了一套供开发者使用的访问接口。Go 中反射的基础是接口和类型，Go 很巧妙的借助了对象到接口的转换时使用的数据结构，先将对象传递给内部的空接口，即将类型转换成空接口 eface。然后反射再基于这个 eface 来访问和操作实例对象的值和类型。



## 二. 反射的内部实现



## 三. 反射三定律






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