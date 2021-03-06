## Java 8 Stream 编程

#### 流处理

* 什么是流处理？

  流是一系列数据项，一次只能生成一项。程序可以从输入流中一个一个读取数据项，然后以同样的方式将数据写入输出流。一个程序的输出流很可能是另一个程序的输入流。

#### 用行为参数化把代码传递给方法

Java 8 增加了把方法(你的代码)作为参数传递给另一个方法的能力。我们把这一概念化称为行为参数化。

* 行为参数化重要在哪呢？

  StreamAPI 就是构建在通过传递代码使操作行为实现参数化的思想上的，当把 compareUsingCustomerId 传递进去，就能把 sort 的行为参数化了。

#### 并行与共享的可变数据

没有共享的可变数据，以及将方法和函数(即代码)作为参数传递给其他方法的能力，这两个要点是函数式编程范式的基石，与此相反，在**命令式编程**范式中，你写的程序则是一些列改变状态的指令。“不能有共享的可变数据”意味着，一个方法可以通过它将参数值转换为结果的方式来完整描述，换句话说，它的行为就像一个数学函数，没有可见的副作用。

#### Java中的函数

编程语言中的**函数**一词通常称为方法，尤其是静态方法，这是在**数学函数**，也就是没有副作用的函数之外的另一含义。

Java 8 中新增加了函数，作为值的一种新形势。有了它，Java 8 可以在多核处理器上进行并行编程。

- 作为值的函数本身有什么用处呢？

  - Java 程序能操作的值有哪些？

    1. 原始值(基本类型)、对象(更严格来说是对象的引用)。
    2. 以上的值可以称为一等值，编程语言中的其他结构也许有帮助于表示值的结构，但在程序执行期间不能传递、Java概念（比如方法和类等）可以称为二等值。

    

