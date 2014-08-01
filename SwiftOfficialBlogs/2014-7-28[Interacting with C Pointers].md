-  本文翻译自Swift官方博客，原文地址：[Interacting with C Pointers](https://developer.apple.com/swift/blog/?id=6)
-  译者：[JaceFu](http://www.devtalking.com/)

# 在Swift中使用C语言的指针
Objective-C和C语言经常需要使用到指针。Swift中的数据类型由于良好的设计，使其可以和基于指针的C语言API无缝混用。同时Swift也可以自动处理大多数将指针作为参数的情况。在这篇文章里，我们可以看到在Swift语言中如何将变量、数组、字符串当做C语言中的指针参数来使用。

## 将输入输出参数作为指针参数
C和Objective-C不支持多类型的返回值。所以Cocoa API就使用指针作为函数的输入输出参数，以用来传递多类型的数据。Swift允许使用指针参数进行类似`inout`参数的处理，所以你可以使用`&`语法将一个`var`变量的引用作为指针参数进行传递。比如说，`UIColor`的`getRed(_:green:blue:alpha:)`方法，使用4个`CGFloat*`指针用来接收颜色的组成元素。我们可以使用`&`将这几个颜色组成部分装配在本地变量中。

```swift
var r: CGFloat = 0, g: CGFloat = 0, b: CGFloat = 0, a: CGFloat = 0
color.getRed(&r, green: &g, blue: &b, alpha: &a)
```

另外一个常见的情况出现在Cocoa `NSError`类的使用中。很多方法都使用一个`NSError**`参数来保存异常信息。比如说，我们可以通过`NSFileManager`类的`contentsOfDirectoryAtPath(_:error:)`方法，挪列出指定目录中的信息，一旦出现疑似异常信息，就将其保存在`NSError?`类型的变量中。

```swift
var maybeError: NSError?
if let contents = NSFileManager.defaultManager()
	.contentsOfDirectoryAtPath("/usr/bin", error: &maybeError) {
	// Work with the directory contents
} else if let error = maybeError {
	// Handle the error
}
```

为安全起见，Swift要求在使用`&`传值时，变量必须是已经被初始化的。这是因为Swift无法知道也无法判断在操作指针之前，该指针是否确实在内存有指向的地址。

## 将数组作为指针参数
在C语言中，指针与数组是水乳交融，纠缠不清的。那么为了在Swift中能无缝的使用C语言中基于数组的一些API，Swift允许将`Array`作为指针参数。一个不可变数组的值可以作为一个`const`指针参数直接传递，可变数组可以使用`&`作为一个非`const`指针参数进行传递，就`inout`参数一样。比如，我们使用`Accelerate`框架中的`vDSP_vadd`函数对数组`a`和数组`b`进行相加，将结果写入`result`数组：

```swift
import Accelerate

let a: [Float] = [1, 2, 3, 4]
let b: [Float] = [0.5, 0.25, 0.125, 0.0625]
var result: [Float] = [0, 0, 0, 0]

vDSP_vadd(a, 1, b, 1, &result, 1, 4)

// result now contains [1.5, 2.25, 3.125, 4.0625]
```

## 将字符串作为指针参数
C语言中，传递字符串的主要方式是通过`const char*`指针。在Swift中，`String`也可以被用作`const char*`指针，用它可以向函数传递空字符串或UTF-8编码的字符串。比如，我们可以在标准的C语言和POSIX的库函数中直接使用字符串作为参数传递：

```swift
puts("Hello from libc")
let fd = open("/tmp/scratch.txt", O_WRONLY|O_CREAT, 0o666)

if fd < 0 {
	perror("could not open /tmp/scratch.txt")
} else {
	let text = "Hello World"
	write(fd, text, strlen(text))
	close(fd)
}
```

## 指针参数转换的安全性
Swift一直在努力让我们可以方便的、无缝的使用C语言中的指针，因为在Cocoa中已经使用的非常普遍了。虽然Swift是一个类型安全的语言，对指针参数的转换的安全性也有保障，但是相比Swift原生的其他代码来说，还是存在着一定的不安全性。所以我们在使用时要格外小心。比如说：

- 如果调用者在指针返回之后保存了指针指向的对象，那么再去使用这个对象时是不安全的。这些被转换的指针参数只能在调用过程中或者发送消息过程中保证其有效性。即时你使用相同的变量、数组或者字符串作为多指针参数进行传递，你每次接收到的指针都是不同的。除非是全局或者静态变量。你可以安全的使用全局或静态变量的指针的参数，比如KVO上下文参数。
- 当将数组或字符串作为指针参数传递时，Swift不会检查其边界值。在C语言中，数组和字符串的大小是不能增长的，所以当你将数组或字符串作为指针参数传递时，要确保它们有足够的大小，或者适合当前场景的大小。

如果你使用的基于指针的API不在这篇指导内，或者你需要重写接收指针参数的Cocoa方法，那么你可以直接使用Swift原始内存中的不安全的指针。我们会在以后的文章中介绍更多Swift的特性。