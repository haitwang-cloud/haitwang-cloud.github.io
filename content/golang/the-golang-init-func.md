+++
title = 'Go init 函数使用介绍'
date = 2024-06-19T16:44:28+08:00
draft = false
+++
> [The Go init Function](https://tutorialedge.net/golang/the-go-init-function/) 的中文翻译版本。

当使用 Go 创建应用程序时，有的时候您需要能够在程序启动时初始化某些资源。例如涉及创建数据库的连接，或从本地存储的配置文件加载配置。


在本教程中，我们将研究如何使用这个 `init()` 函数来实现初始化，我们还将看看为什么这不一定是实例化组件的最佳方法。

### 替代方案（Alternative Approaches ）
----------------------

现在，使用 `init` 函数的典型用例可能类似于“我想实例化与数据库的连接”。但是这实际上可能是 Go 应用程序中设计不佳的副作用。

实例化数据库连接之类的更好方法可能是使用`New`函数，该函数返回指向包含数据库连接对象的结构的指针。

```Go
func New() (*Database, error) {
    conn, err := sql.Open("postgres", "connectionURI")
    if err != nil {
        return &Database{}, err
    }
    return &Database{
        Connection: conn,
    }, nil
}
```

使用这种方法，您可以将数据库传递给系统中可能需要调用数据库的任何其他组件。

它还使您可以在启动期间如何更好地处理故障，而不是简单地终止您的应用程序。

总的来说，我会尽量避免使用 `init` 函数，并使用上面概述的方法来实例化数据库连接，构建Go 应用程序。

### init 函数（The init Function）
-----------------

在 Go 中，`init()` 函数非常强大。同时与其他一些语言相比，在 Go 程序中更容易使用。这些 `init()` 函数可以在 `package` 块中使用，并且无论该包被导入多少次，`init()` 函数只会被调用一次。

**现在你需要知道的是，`init()` 函数只会被调用一次**。当我们想要建立数据库连接，或向各种服务注册中心注册，或执行您通常只想执行一次的任何数量的其他任务，它会是十分有效的。

```Go
package main

func init() {
  fmt.Println("This will get called on main initialization")
}

func main() {
  fmt.Println("My Wonderful Go Program")
}
```

请注意在上面的示例中，我们没有在程序中的任何地方显式调用 `init()` 函数。 Go 隐式地为我们处理执行，因此上面的程序输出如下：

```Shell
$ go run test.go
This will get called on main initialization
My Wonderful Go Program
```
通过这个工作，我们可以开始做一些很酷的事情，比如变量初始化。

```Go
package main

import "fmt"

var name string

func init() {
    fmt.Println("This will get called on main initialization")
    name = "Elliot"
}

func main() {
    fmt.Println("My Wonderful Go Program")
    fmt.Printf("Name: %s\n", name)
}
```

在这个例子中，我们可以开始了解为什么使用 `init()` 函数比显式调用您自己的setup函数相比会更受欢迎。


当我们运行上面的程序时，你应该会看到我们的 `name` 变量已经正确赋值。虽然它不是这个星球上最有用的变量😊，但我们当然仍然可以在整个 Go 程序中使用它。

```Shell
$ go run test.go
This will get called on main initialization
My Wonderful Go Program
Name: Elliot
```

多个包的情况（Multiple Packages）
-----------------

接下来是更接近在生产 Go 系统中更复杂的场景，比如我们的应用程序中有 4 个不同的 Go 包，`main`, `broker`和`database`。

在这些包中，我们可以指定一个 `init()` 函数，它将执行以下操作：各种第三方服务（如 Kafka 或 MySQL）设置连接池的任务。

当你调用了了 `database` 包中任何函数的时候，它都会使用我们在 `init()` 函数中设置的连接池。

> **Note -** 请注意，你不能依赖于你的 `init()` 函数的执行顺序。相反，最好专注于以顺序无关紧要的方式编写系统。

初始化顺序（Order of Initialization）
-----------------------

对于更加复杂的系统，你可能有多个文件组成一个包。这些文件可能有自己的 `init()` 函数。所以 Go 如何确定这些包的初始化顺序呢？

当涉及到初始化的顺序时，需要考虑一些事情。Go 通常是按照声明顺序初始化，但是如果它们依赖于其他文件，那么它们必定将会先初始化。这意味着如果你在同一个包中有 2 个文件 `a.go` 和 `b.go` ，如果 `a.go` 中的任何东西依赖于 `b.go` 中的东西，它们（`b.go`）将会先初始化。


> **Note -** 你可以在官方文档中找到有关 Go 中初始化顺序的更深入概述：: [Package Initialization](https://golang.org/ref/spec#Package_initialization)


需要注意的是，这个顺序可能会导致这样的场景：

```
// source: https://stackoverflow.com/questions/24790175/when-is-the-init-function-run
var WhatIsThe = AnswerToLife()

func AnswerToLife() int {
    return 42
}

func init() {
    WhatIsThe = 0
}

func main() {
    if WhatIsThe == 0 {
        fmt.Println("It's all a lie.")
    }
}
```
在这个场景中，你会看到 `AnswerToLife()` 在我们的 `init()` 函数之前被调用。因为我们的 `WhatIsThe` 变量被声明在我们的 `init()` 函数之前，所以它的值是 `0`。

```Shell
$ go run main.go
                                        
It's all a lie.
```



### 在同一个文件中有多个初始化函数（Multiple Init Functions in the Same File）
----------------------------------------


如果我们在同一个 Go 文件中有多个 `init()` 函数会发生什么？首先我不会认为这是可能的，但是 Go 却实际上可以支持有 2 个不同的 `init()` 函数在同一个文件中。

这些 `init()` 函数仍然会按照声明顺序在文件中被调用：

```Go
package main

import "fmt"

// this variable is initialized first due to
// order of declaration
var initCounter int

func init() {
    fmt.Println("Called First in Order of Declaration")
    initCounter++
}

func init() {
    fmt.Println("Called second in order of declaration")
    initCounter++
}

func main() {
    fmt.Println("Does nothing of any significance")
    fmt.Printf("Init Counter: %d\n", initCounter)
}
```


现在当你运行上面的程序时，你会看到输出如下：

```Shell
$ go run test.go
Called First in Order of Declaration
Called second in order of declaration
Does nothing of any significance
Init Counter: 2
```

很酷吧？但这是为了什么？为什么我们允许这样做？好吧，对于更复杂的系统，Go 允许我们将复杂的初始化过程分解为多个更易于理解的 `init()` 函数。Go本质上允许我们避免在单个 `init()` 函数中有一个单一的代码块，这是一件好事。但是，这种方式的一个缺点是，你需要注意声明顺序。


结论（Conclusion）
----------
至此，对 `init()` 函数世界的基本介绍到此结束。一旦你掌握了包初始化的使用，可能会对你掌握 Go 项目的结构化有所帮助。