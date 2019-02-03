---
layout: post
title: 浅谈 Kotlin 中的 apply、let、run 与 also 等函数
author: DuckSoft
categories: [编程]
tags: [Kotlin, JVM]
image: mountains.jpg
---

## 写在前面
> Updated: `2019-02-02`

时隔两年，整理自己的旧博客，俨然发现自己接触 Kotlin 已有足足两年时间。还记得，我在一次无意的操作中将一个 Java 文件使用 IntelliJ IDEA 的“一键转换 Kotlin”功能转成了自己根本不认识的代码。

当时，Kotlin 给我的印象是：比起繁琐至极的 Java 代码，Kotlin 在保留了可读性的前提下更加精炼和简洁。诚然当时我只看到了 Kotlin 的冰山一角——自动生成的 `getter` 与 `setter`。同时 JetBrains 公司在 2016 年 2 月释出的 [Kotlin v1.0 版本](https://blog.jetbrains.com/kotlin/2016/02/kotlin-1-0-released-pragmatic-language-for-jvm-and-android/)也在网上吸引了大量的开发者的关注，于是在百度（没错，当时的百度还能看）一番之后便对 Kotlin 一见钟情，开始了 Kotlin 之旅。

Kotlin 给我的感觉，用一句话来概括：在用 Kotlin 写代码的时候，你真的会感觉到编程是幸福的。Kotlin 语言丰富的语法糖、强大的扩展函数、省心的自动类型推导、简练强大的空值安全机制让受够了 Java/C++ 等语言毒害的人几乎不能抵御。

最让人舒心的是 JetBrains 公司的 IDE 对 Kotlin 的支持做的可谓面面俱到，从一开始就扫清了开发的障碍。天时地利人和俱备的语言，在 Google I/O 2017 大会上[被宣布为 Android 开发官方支持语言](https://blog.jetbrains.com/kotlin/2017/05/kotlin-on-android-now-official/)。（不过毕竟不是谷歌爸爸亲生的，后来被 Dart 和 Flutter 替代了，然而在后者生态环境缺乏的情况下，Kotlin 仍然可以占有一席之地）

两年之后重新整理这篇文章，加上了近些年来的一些想法，希望对大家有帮助。最后，Kotlin 神教万岁！喷气脑神教万岁！

![kotlin-huaji](https://user-images.githubusercontent.com/7822648/52177753-cd3dc800-2800-11e9-8d54-39bd415121c1.jpg)

## 看本质：定义层面上的分析
利用 IntelliJ IDEA 自带的定义查询功能（快捷键 `Ctrl+Q`，或者使用 `Ctrl+LClick` 直接跳转到相对应的代码中查看），我们看一下这几个函数的定义：

| 名称 | 定义 |
| ---- | -------- |
| `apply` | `public inline fun <T> T.apply(block: T.() -> Unit): T` |
| `run` | `public inline fun <T, R> T.run(block: T.() -> R): R`|
| `also` | `public inline fun <T> T.also(block: (T) -> Unit): T` |
| `let` | `public inline fun <T, R> T.let(block: (T) -> R): R` |
| `run`* | `public inline fun <R> run(block: () -> R): R` |
| `with`* | `public inline fun <T, R> with(receiver: T, block: T.() -> R): R` |

通过简单的观察我们可以看到，前四个函数 `apply`、`run`、`also` 和 `let` 同属 Kotlin 中的**扩展函数**（[extension function](https://kotlinlang.org/docs/reference/extensions.html)）一类，而带有 `*` 符号的两个函数则不是扩展函数。这样就把我们的六大函数分成了两类：**扩展函数**与**非扩展函数**。

### 四个扩展函数: `apply`、`run`、`also` 与 `let`
通过观察四个扩展函数的定义，我们不难发现其中有两组正交的异同点：

* `apply` 和 `run` 的 `block` 函数使用 `T.() ->`，能捕获 `T` 对象作为函数的 `this` 作用域；而 `also` 和 `let` 的 `block` 函数使用 `(T) ->`，能捕获 `T` 对象作为函数的单个参数 `it`。
* `apply` 和 `also` 均返回原 `T` 对象的引用（reference）；而 `run` 和 `let` 则返回 `block` 函数返回值 `R` 的引用。

如果觉得文字过于繁琐，我们可以简单地用一张表来概括：

| 函数名称 | 捕获输入为 | 返回 |
|---------| ----| ---- |
| `apply` | `this` | 原引用 |
| `run` | `this` | `block` 返回值 |
| `also` | `it` | 原引用 |
| `let` | `it` | `block` 返回值 |

### 两个非扩展函数: `run` 与 `with`
我们首先来观察 `with` 的定义，不难发现这与前节所提**扩展函数**中的 `run` 非常相似，唯一不同的是 `with` 将扩展函数的 `T.` “拿进”了参数列表里面。

而 `run` 的非扩展函数版本在定义上则没有什么可圈可点之处，与扩展函数版本相比，只是彻底的少了 `T.` 而已。

---

## 看实际：试图用一堆例子来启发大家的懒人作者
> “自强不息，知行合一。”——东北大学校训

> **超长代码警告**
> 
> 若您有“看到大段文字或代码就会面露异常”的特性，敬请立即关闭本页面。

这些函数的普遍用途很浅显，无非是将一段代码进行“提取公因子”而已。例如下面的代码片段：

```kotlin
fun foo(): String {
    val sb = StringBuilder()

    sb.append("Hello")
    sb.append(' ')
    sb.append("World!")

    return sb.toString()
}
```

我们可以使用 `apply` 函数将其改写为下面的、更为简洁的形式：

```kotlin
fun foo(): String {
    val sb = StringBuilder().apply {
        append("Hello")
        append(' ')
        append("World!")
    }

    return sb.toString()
}
```

其实如果你对 Kotlin 足够了解的话，我们还可以借助 Kotlin 的函数表达式（即将函数写成表达式）的特性，将上述代码进行再次简化：

```kotlin
fun foo() = StringBuilder().apply {
    append("Hello")
    append(' ')
    append("World!")
}.toString()
```

无疑，Kotlin 中这样的简单暴力的操作使得 Java 中的构造者模式繁杂的写法颜面无存。举一个使用 Google GSON 库时 Kotlin 的例子我们就能略窥一二：

```kotlin
// 此处使用了 Kotlin Ktor 库
call.respond(JsonObject().apply {
    addProperty("capacity", ParkingLot.capacity)
    add("inside", JsonArray().apply {
        ParkingLot.forEach { vehicle -> add(JsonObject().apply {
            addProperty("plateNumber", vehicle.plateNumber)
            addProperty("timeArrive", vehicle.timeArrive.toString())
            addProperty("timeAccumulated", vehicle.timeAccumulated.toString())
        })}
    })
    add("outside", JsonArray().apply {
        ParkingLot.sideway.forEach { vehicle -> add(JsonObject().apply {
            addProperty("plateNumber", vehicle.plateNumber)
            addProperty("timeArrive", vehicle.timeArrive.toString())
            addProperty("timeAccumulated", vehicle.timeAccumulated.toString())
        })}
    })
})
```

不仅如此，Kotlin 的 `apply` 等函数还能用在变量的初值上。这可以将大量与对象中某几个变量有关的逻辑完全地从构造函数体中抽离出来，保持构造函数体的简洁性。例如下面的代码：

```kotlin
// 此处使用了 Vert.x + Jackson

/**
 * 外部配置文件。
 */
val configFileExternal = File("config.json")

/**
 * 外部配置文件不存在时，复制并使用内部配置文件；
 * 外部配置文件存在时，优先使用外部配置文件。
 */
val configFile = if (!configFileExternal.exists()) {
    mainLogger.info("Configuration file does not exist. Creating from default configuration.")
    configFileExternal.apply {
        bufferedWriter().also {
            val resource = Startup.javaClass.classLoader.getResource("config.json")
            resource.openConnection().getInputStream().bufferedReader().copyTo(it)
        }.also { it.flush() }.also { it.close() }
    }
} else configFileExternal

/**
 * 从配置文件中加载的配置对象。
 */
val configObject = JsonObject(ObjectMapper().readTree(configFile)!!.toString())
```

在上面的代码中，我们交替地使用了 `apply` 和 `also` 等函数来提取公共变量，虽有炫技之嫌，无形之中的确省却了使用 `val` 关键字来声明变量的痛事（炫技要适度，该提取变量还是要提取变量呀）。而后面的两组独立的 `also` 则用于分离或强调几组逻辑。

总体而言，在使用这些函数时风格大多比较自由。大体上只要把握住使用的限度，应该不会被开发团队里的人打死（逃）。

---

各位看官看到这里是否发现一个问题：`let`、`with` 和 `run` 还没有出现呢！正如上面所说的，Kotlin 的这些函数的使用风格比较自由，而在我的代码风格里这些函数恰恰很少出现。不过为了能够使大家更全面地了解这些函数的用法，我还是从代码库里翻出了一些祖传代码，供大家吐槽之用：

下面是一段使用到 `let` 的代码：
```kotlin
fun ByteArray.toHexStringLCase() = "0123456789abcdef".let { hexChars ->
    StringBuilder(this.size * 2).also { s ->
        this.forEach { byte ->
            byte.toInt().also { int ->
                s.append(hexChars[int shr 4 and 0x0f])
                s.append(hexChars[int and 0x0f])
            }
        }
    }.toString()
}
```


下面是一段使用到 `run` 的代码：
```kotlin
call.receive<JsonObject>().run {
    val spotName = this["name"].asString ?: return@run null
    val spotDescription = this["description"].asString ?: return@run null
    val location = Location.randomize()

    Spot(spotName, spotDescription, location)
}?.apply {
    scene.addSpotSafely(this)
    call.respond(HttpStatusCode.Accepted)
} ?: call.respond(HttpStatusCode.InternalServerError)
```

需要特殊说明的是，这段代码借助 Kotlin 的空值机制和 `run` 配合，做到了类似于函数式编程中 Monad 范式的效果。这也是作者后续想要给大家分享的内容之一，详细内容看官们敬请期待。

未完待续！
