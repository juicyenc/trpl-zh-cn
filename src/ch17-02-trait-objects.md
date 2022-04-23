## 使用顾及不同类型值的 trait 对象

> [ch17-02-trait-objects.md](https://github.com/rust-lang/book/blob/main/src/ch17-02-trait-objects.md)
> <br>
> commit 727ef100a569d9aa0b9da3a498a346917fadc979

 在第八章中，我们提到了 vector 的一个局限是它只能存储同种类型元素。在示例 8-10 中提供了一个定义 `SpreadsheetCell` 枚举来储存整型、浮点型和文本成员的替代方案。这意味着可以在每个单元中储存不同类型的数据，并仍能拥有一个代表一排单元的 vector。这在编译代码时那些可替换项的类型就是我们所知的一个固定集合时是一个非常好的方案。

然而有时我们希望库用户在具体情况下能够扩展有效的类型集合。为了展示或许我们如何实现这一点，这里将创建一个图形用户接口（Graphical User Interface， GUI）工具的例子，它通过遍历列表并调用每一个项目的 `draw` （绘制）方法来将其绘制到屏幕上 —— 一个 GUI 工具中的常用技术。我们将要创建一个包含GUI库结构，叫做 `gui` 的库 crate。这个 GUI 库可能包含一些可供开发者使用的类型，比如 `Button` （按钮）或 `TextField`（文本框）。在此之上，`gui` 的用户会希望创建他们自己的可绘制类型：比如，一个程序员可能会添加 `Image`（图片），另一个可能会添加 `SelectBox`（选择框）。

这个例子中并不会实现一个功能完善的 GUI 库，不过会展示其中各个部分是如何结合在一起的。当编写库的时候，我们不可能得知并定义所有其他程序员希望创建的类型。但我们所明晰的是 `gui` 需要追踪一系列不同类型的值，并且得调用这每一个不同类型值的 `draw` 方法。库无需知道调用 `draw` 方法时具体会发生什么，只是那个值会有那个方法可供我们调用。

在拥有继承的语言中，我们或许会定义一个名为 `Component` （组件）的类，该类上有一个 `draw` 方法。其他的类比如 `Button`、`Image` 和 `SelectBox` 会从 `Component` 派生并因此继承 `draw` 方法。它们可以各自覆盖 `draw` 方法来定义它们自己的行为，但是框架将会把所有这些类型当作是 `Component` 的实例，并在其上调用 `draw`。不过因为 Rust 并没有继承，我们得另寻出路去架构 `gui` 库以允许用户扩展新类型。

### 定义通用行为的 trait

为了实现 `gui` 所期望的行为，让我们定义一个叫做 `Draw` 的 trait，其中包含一个名为 `draw` 的方法。接着可以定义一个存放 **trait 对象**（*trait object*） 的 vector。trait 对象指向一个实现了我们指定 trait 的类型的实例同一个用于在运行时查找该类型的trait方法的表。我们通过指定某种指针来创建 trait 对象，例如 `&` 引用或  `Box<T>` 智能指针，还有 `dyn`  关键字， 然后指定关联的 trait（第十九章  [““动态大小类型和 `Sized` trait”][dynamically-sized] 部分会介绍 trait 对象必须使用指针的原因）。我们可以使用 trait 对象代替泛型或具体类型。无论何地我们使用 trait 对象，Rust 的类型系统会在编译期确保任何在该上下文中使用的值会实现其 trait 对象的 trait。如此便无需在编译时就得知所有可能的类型。

之前提到过，Rust 刻意避免结构体与枚举称为 “对象”，以便与其他语言中的对象相区别。在结构体或枚举中，结构体字段中的数据和 `impl` 块中的行为是分开的，不同于其他语言中将数据和行为组合进一个称为对象的概念中。然而， trait **对象实例**更接近其他语言中的对象，因其在某种意义上结合了数据和行为。不过 trait 对象不同于传统的对象，因为不能向 trait 对象增加数据。trait 对象并不像其他语言中的对象那么有用：它们的特定作用是允许抽象各种通用行为。

示例 17-3 展示了如何定义一个叫做 `Draw` , 带有 `draw` 方法的 trait：

<span class="filename">文件名: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-03/src/lib.rs}}
```

<span class="caption">示例 17-3：`Draw` trait 的定义</span>

这语法应该看起来和第10章我们讨论如何定义 trait 时的比较眼熟。接下来就是新内容了：示例 17-4 定义了名为 `Screen` 的结构体，存放着一个名为 `components` 的 vector 。这个 vector 属于 `Box<dyn Draw>` 类型，是一个 trait 对象：它是 `Box` 中任何实现了 `Draw` trait 的类型的替身。

<span class="filename">文件名: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-04/src/lib.rs:here}}
```

<span class="caption">示例 17-4: 一个 `Screen` 结构体的定义，它带有一个字段 `components`，其包含实现了 `Draw` trait 的 trait 对象的 vector</span>

在 `Screen` 结构体上，我们将定义一个 `run` 方法，该方法会调用 `components` 上每一项的 `draw` 方法，如示例 17-5 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-05/src/lib.rs:here}}
```

<span class="caption">示例 17-5：在 `Screen` 上实现一个 `run` 方法，该方法在每个 component 上调用 `draw` 方法</span>

这与定义使用了带有 trait bound 的泛型类型参数的结构体不同。泛型类型参数一次只能代入一个具体类型，然而 trait 对象允许其在运行时代入多种具体类型。例如，我们本可以定义 `Screen` 结构体去使用泛型以及 trait bound，如示例 17-6 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-06/src/lib.rs:here}}
```

<span class="caption">示例 17-6: 一种 `Screen` 结构体的替代实现，其 `run` 方法使用泛型和 trait bound</span>

这限制了 `Screen` 实例必须拥有一个全是 `Button` 类型或者全是 `TextField` 类型的组件列表。如果只需要同质（相同类型）集合，则倾向于使用泛型和 trait bound，因为其定义会在编译期采用具体类型进行单态化。

另一方面，通过使用 trait 对象的方法，一个 `Screen` 实例可以存放一个包含 `Box<Button>`，同时包含 `Box<TextField>` 的 `Vec<T>`。让我们来看看它是如何工作的，接着会讲解对运行时性能的影响。

### 实现所需的 trait

现在来增加一些实现了 `Draw` trait 的类型。我们会提供 `Button` 类型。再一次重申，真正实现 GUI 库超出了本书的范畴，所以 `draw` 方法体中不会有任何有意义的实现。为了想象实现会看起来像什么，一个 `Button` 结构体可能会拥有 `width`、`height` 和 `label` 字段，如示例 17-7 所示：

<span class="filename">文件名: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch17-oop/listing-17-07/src/lib.rs:here}}
```

<span class="caption">示例 17-7: 一个实现了 `Draw` trait 的 `Button` 结构体</span>

在 `Button` 上的 `width`、`height` 和 `label` 字段会和其他组件不同，比如 `TextField` ，后者除此之外还加上一个 `placeholder` 字段。每一个我们希望能在屏幕上绘制的类型都会实现 `Draw` trait 的 `draw` 方法但使用不同的代码来定义如何绘制特定的类型，就像 `Button` 类型的一样（并不包含任何实际的 GUI 代码，这超出了本章的范畴）。除了实现 `Draw` trait 之外，比如 `Button` 类型，作为一个例子，还可能有另一个包含与按钮点击时会发生什么相关的方法的 `impl` 块。这类方法不会适用于类似 `TextField` 的类型。

如果一些库的使用者决定实现一个包含 `width`、`height` 和 `options` 字段的结构体 `SelectBox`，他们也会为其实现 `Draw` trait，如示例 17-8 所示：

<span class="filename">文件名: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch17-oop/listing-17-08/src/main.rs:here}}
```

<span class="caption">示例 17-8: 另一个使用了 `gui` 的 crate ，其中 `SelectBox` 结构体实现了 `Draw` trait</span>

库使用者现在可以在他们的 `main` 函数中创建一个 `Screen` 实例。至此可以通过将 `SelectBox` 和 `Button` 放入 `Box<T>` 转变为 trait 对象来增加组件。然后调用 `Screen` 的 `run` 方法，它会调用每个组件的 `draw` 方法。示例 17-9 展示了这个实现：

<span class="filename">文件名: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch17-oop/listing-17-09/src/main.rs:here}}
```

<span class="caption">示例 17-9: 使用 trait 对象来存储实现了相同 trait 的不同类型的值</span>

当编写库的时候，我们不知道可能会有人添加 `SelectBox` 类型，不过 `Screen` 的实现能够在这个新类型上运作并绘制它，这是因为 `SelectBox` 实现了 `Draw` trait，意味着它实现了 `draw` 方法。

这个概念 —— 只关心值返回的信息而不是其具体类型 —— 类似于动态类型语言中称为 **鸭子类型**（*duck typing*）的概念：如果它走起来像一只鸭子，叫起来像一只鸭子，那么它就是一只鸭子！在示例 17-5 中 `Screen` 上的 `run` 实现中，`run` 并不需要知道各个组件的具体类型是什么。它并不检查组件是 `Button` 还是 `SelectBox` 的实例，它只是调用了组件上的 `draw` 方法。通过指定 `Box<dyn Draw>` 作为 `components` vector 中值的类型，我们就定义了 `Screen` 需要可以在其上调用 `draw` 方法的值。

使用 trait 对象和 Rust 类型系统来进行类似鸭子类型操作的优势是永远不需要在运行时检查一个值是否实现了特定方法或者担心在调用时因为值没有实现方法而产生错误。如果值没有实现 trait 对象所需的 trait 则 Rust 不会编译这些代码。

例如，示例 17-10 展示了当创建一个使用 `String` 做为其组件的 `Screen` 时发生的情况：

<span class="filename">文件名: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-oop/listing-17-10/src/main.rs}}
```

<span class="caption">示例 17-10: 尝试使用一种没有实现 trait 对象的 trait 的类型</span>

我们会遇到这个错误，因为 `String` 没有实现 `Draw` trait：

```console
{{#include ../listings/ch17-oop/listing-17-10/output.txt}}
```

这个错误告诉我们，要么是我们传递了并不意图传递给 `Screen` 的类型且应该传递其他类型，要么应该在 `String` 上实现 `Draw` 以便 `Screen` 可以调用其上的 `draw`。

### trait 对象执行动态派发

回忆一下第十章 [“泛型代码的性能”][performance-of-code-using-generics] 部分讨论过的，当对泛型使用 trait bound 时编译器所进行单态化处理：编译器为每一个被泛型类型参数代替的具体类型生成了非泛型的函数和方法实现。单态化所产生的代码进行 **静态派发**（*static dispatch*）。静态分发发生于编译器在编译时就知晓调用了什么方法的时候。这与 **动态派发** （*dynamic dispatch*）相反，这时编译器在编译时无法知晓调用了什么方法。在动态分发的情况下，编译器会生成在运行时确定调用了什么方法的代码。

当使用 trait 对象时，Rust 必须使用动态分发。编译器无法知晓所有可能用于 trait 对象代码的类型，所以它也不知道应该调用哪个类型的哪个方法实现。为此，在运行时，Rust 使用 trait 对象中的指针来知晓需要调用哪个方法。这个不存在于静态派发中的查找也有运行时消耗。动态分发也阻止编译器选择去内联方法代码，这会相应地禁用一些优化。但是我们编写的示例17-5确实获得了额外的灵活性，并且得以支持示例17-9。所以这是一个需要思考的权衡。


[performance-of-code-using-generics]:
ch10-01-syntax.html#泛型代码的性能
[dynamically-sized]: ch19-04-advanced-types.html#动态大小类型和-sized-trait
[Rust RFC 255 ref]: https://github.com/rust-lang/rfcs/blob/master/text/0255-object-safety.md
[Rust Reference ref]: https://doc.rust-lang.org/reference/items/traits.html#object-safety