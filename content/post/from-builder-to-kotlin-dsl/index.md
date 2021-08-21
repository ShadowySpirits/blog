---
title: 使用 Kotlin DSL 代替 Builder 模式
slug: from-builder-to-kotlin-dsl
date: 2019-04-29T23:22:00+08:00
image: kotlin-logo.png
tags:
  - Kotlin
categories:
  - tech
---

本文旨在介绍如何用 Kotlin DSL 来代替 Builder 模式，如果你不知道什么是 DSL 或者不了解 Kotlin 中的 DSL 可以阅读我的上一篇文章：[Kotlin DSL 简介]({{< ref "introduction-to-kotlin-dsl" >}})

 <!--more-->

## 举个例子

我们想编写一个 HTML 构建器输出如下：

```html
<html>
  <head>
    <title>HTML encoding with Kotlin</title>
  </head>
  <body>
    <h1>HTML encoding with Kotlin</h1>
    <p>this format can be used as an alternative markup to HTML</p>
    <a href="http://jetbrains.com/kotlin"> Kotlin </a>
    <p>some text</p>
  </body>
</html>
```

如果使用 Builder 模式代码类似这样：

```kotlin
val html = HTMLBuilder().addHead(
                HeadBuilder().addTitle(TitleBuilder("HTML encoding with Kotlin").build())
            ).addBody(
                BodyBuilder().addH1(H1Builder("HTML encoding with Kotlin"))
                    .addP(PBuilder("this format can be used as an alternative markup to HTML").build())
                    .addA(ABuilder("Kotlin")
                                .setHerf("http://jetbrains.com/kotlin")
                                .build()
                    ).addP(PBuilder("some text").build())
            ).build()
println(result)
```

写这些 XXXBuilder 就很麻烦了，而且代码十分臃肿，可读性很差。再来看看 Kotlin DSL 的写法：

```kotlin
fun main(args: Array<String>) {
    val result =
            html {
                head {
                    title { +"HTML encoding with Kotlin" }
                }
                body {
                    h1 { +"HTML encoding with Kotlin" }
                    p { +"this format can be used as an alternative markup to HTML" }

                    // an element with attributes and text content
                    a(href = "http://jetbrains.com/kotlin") { +"Kotlin" }
                    p { +"some text" }
                }
            }
    println(result)
}
```

显然更加清晰明了，并且代码量要少的多。

## 具体实现

我们来分析一下这段 Kotlin DSL 实现：构造标签使用的是类似 html { ... } 这样的函数，这个函数接收一个 Lambda 表达式作为参数，并返回一个 HTML 对象：

```kotlin
inline fun html(init: HTML.() -> Unit): HTML {
    val html = HTML()
    html.init()
    return html
}
```
{{< tip info >}}
这里使用内联函数来避免 Lambda 表达式的开销
{{< /tip >}}

其他标签同理，所以我们可以构建一个泛型函数：

```kotlin
abstract class Tag() {
    val children = arrayListOf<Tag>()

    inline protected fun <T: Tag> initTag(tag: T, init: T.() -> Unit): T {
        tag.init()
        children.add(tag)
        return tag
    }

    ....

}
```

然后传入对应的标签类型即可创建不同的标签：

```kotlin
class HTML(): Tag() {
    fun head(init: Head.() -> Unit) = initTag(Head(), init)

    fun body(init: Body.() -> Unit) = initTag(Body(), init)
}
```

这样我们就完成了一个简单的 HTML Builder，完整的代码见此：[HTML Builder](https://try.kotlinlang.org/#/Examples/Longer%20examples/HTML%20Builder/HTML%20Builder.kt)。

## 限制作用域

思考如下代码：

```kotlin
html {
    +"html scope"
    head {
        +"head scope"
        head {
            +"head scope"
        }
    }
}
```

显然 `head` 标签中嵌套另一个 `head` 标签是没有意义的，而且编译却不会报错。这是因为 Kotlin 会隐式推断接收者为最顶层 Lambda 表达式中 this 指向的 HTML 对象。所以我们要修正这个问题就要禁止这种隐式推断：

```kotlin
@DslMarker
annotation class HtmlTagMarker

@HtmlTagMarker
abstract class Tag() {
    ....
}
```

使用 `@DslMarker` 声明一个注解 `@HtmlTagMarker`，然后为所有标签的基类 `Tag` 加上这个注解即可。再次尝试使用这种不规范的写法就会报错：

```plaintext
'fun head(init: Head.() -> Unit): Head' can't be called in this context by implicit receiver.
```

但是我们仍通过指定的接收者进行调用：

```kotlin
html {
    +"html scope"
    head {
        +"head scope"
        this@html.head {
            +"head scope"
        }
    }
}
```
