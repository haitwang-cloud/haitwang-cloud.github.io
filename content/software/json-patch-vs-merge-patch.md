+++
title = 'JSON Patch and JSON Merge Patch'
date = 2024-06-23T09:55:47+08:00
draft = false
+++

>  [JSON Patch and JSON Merge Patch](https://erosb.github.io/post/json-patch-vs-merge-patch/) 的中文翻译版本。

作为近年来受到关注HTTP的 “PATCH” 方法的带动，人们开始提出关于表示 JSON 驱动的 PATCH 格式的想法，它以声明方式描述了两个 JSON 文档之间的差异。 目前已经有许多解决方案被提出，不计其数。而其中IETF 已经发布了两种格式作为 RFC 文档来解决这个问题：[RFC 6902 (JSON Patch)](https://tools.ietf.org/html/rfc6902) 和 [RFC 7396 (JSON Merge Patch)](https://tools.ietf.org/html/rfc7386)。两者都有优缺点，会根据不同的用例因人而异，所以让我们快速比较，看看你会使用哪一个。

JSON Patch
----------

JSON Patch format 和数据库事务类似：它是JSON 文档上的更改操作，由适当的实现自动执行。它基本上是一系列 `"add"`, `"remove"`, `"replace"`, `"move"` 和 `"copy"` 操作。


让我们通过一个简单的例子来看看JSON 文档：

```json
{
	"users" : [
		{ "name" : "Alice" , "email" : "alice@example.org" },
		{ "name" : "Bob" , "email" : "bob@example.org" }
	]
}
```

我们可以通过PATCH操作更新它，这次操作更改了Alice的邮件地址，然后添加了一个新的元素到数组中：

```json
[
	{
		"op" : "replace" ,
		"path" : "/users/0/email" ,
		"value" : "alice@wonderland.org"
	},
	{
		"op" : "add" ,
		"path" : "/users/-" ,
		"value" : {
			"name" : "Christine",
			"email" : "christine@example.org"
		}
	}
]
```

然后结果会变成:

```json
{
	"users" : [
		{ "name" : "Alice" , "email" : "alice@wonderland.org" },
		{ "name" : "Bob" , "email" : "bob@example.org" },
		{ "name" : "Christine" , "email" : "christine@example.org" }
	]
}
```

所以 JSON Patch 中的操作大纲是：
*  `"op"` 代表的操作类型
* 由其他键描述的操作参数
*  `"path"` 参数指向操作的目标文档片段



一个有趣的选项是 JSON Patch 的 `"test"` 操作符：它的执行不会产生任何副作用，所以它不是一个数据操作符。而是用来描述文档的断言，如果 `"test"` 返回 false，那么会发生错误，后续的操作不会执行，文档会回到初始状态。我觉得 `"test"` 对于在补丁执行之前检查先决条件很有用，用来检查执行后的文档是否正常。JSON Patch 是一个原子性的操作，所以如果 `"test"` 发现文档不一致，那么你可以认为文档已经回到初始状态。

JSON Merge Patch
----------------

除了 JSON Patch 之外，还有另一种 JSON 格式，[JSON Merge Patch - RFC 7386](https://tools.ietf.org/html/rfc7386)，也可以用于相同的目的。**JSON Merge Patch 和 JSON Patch 的主要区别是，JSON Merge Patch 类似于 diff 文件**，它只包含文档中应该变化的节点。


请看一个简单的例子[数据源](https://tools.ietf.org/html/rfc7386#section-1)，假设我们有以下文档：


```json
{
	"a": "b",
	"c": {
		"d": "e",
		"f": "g"
	}
}
```

我们可以通过下面的内容patch它

```json
{
	"a":"z",
	"c": {
		"f": null
	}
}
```

它将改变 `"a"` 的值为 `"z"`，并将 `"f"` 键删除。

乍一看，这个格式可能会看起来很更简便一些，因为大多数人都能够直接理解原始文档的格式。因为很可能任何了解原始文档架构的人也会立即理解merge patch文档。 它只是一个标准化的一个可以自然地称为一个 JSON 文档的PATCH。


但这种简单性也有一些限制：
*  删除是通过设置键的值为 `null` 来实现的。这意味着不能通过改变键的值来删除键，因为这种改变不能被merge patch描述。
*  数组不能被merge patch操作。如果你想添加一个元素到数组，或者改变数组中的任何元素，那么你必须包含整个数组在merge patch文档中，即使是微小的改动。
* merge patch不会导致错误。任何不正确的patch都会被合并，所以它是一个很严格的格式。它不一定是好的，因为你可能需要在合并后进行程序性检查，或者在合并后运行JSON Schema验证。


总结（Summary）
-------

JSON Merge Patch是一个简易的patch操作，但是它也有明显的局限性。如果你正在建造一个小的项目，使用简单的JSON Schema，它可能是一个很好的选择。但你希望提供一个可以被客户端读取的快速易懂，较为可用的方法来更新JSON文档。面向公共消费而设计但没有适当的客户端库的 REST API 可能是一个更好的选择。

对于更加复杂的使用场景，我会偏向于JSON Patch，因为它适用于任何JSON文档（不论是merge patch，它不能设置键为 `null`）。该规范也保证了原子执行和robust的错误报告。
