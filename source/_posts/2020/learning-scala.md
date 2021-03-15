---
title: Learning Scala
date: 2020-08-25 09:07:05
categories: Tech
tags: 
 - Scala
---

<div class="banner-img">
    <img src="/images/2020/scala-banner.png">
</div>

近日的生活略显平淡，被困于校内而不允许随意出行，听Dr. Xinkang Jia说Coursera上使用edu邮箱可以在9月底前free enroll，那正好想要了解一下JVM平台的语言，又顺便想尝试一下真正的函数式语言，于是选择了Functional Programming Principles in Scala这门课。Martin Odersky的教学方式与过去接触到的语言入门课程有点不太类似，并非以语法、类型等入手，而是以高阶函数这一FP最突出的特点逐渐熟悉Scala。虽然这门课时间不长，但对于暂且只熟悉了OOP的人而言，已经能领略到Scala的优雅与魅力。
<!--more-->

# 课后作业
说实话对于大部分人而言，这门课的课后作业难度略显不够平滑。在介绍了基本结构与（尾）递归之后，随之而来的作业是三个算法：帕斯卡三角，括号平衡，找零的所有组合，有人在forum里吐槽课后题就像是在做Leetcode。也许有人具备Java或C#背景，我看到forum里还有PHP背景的，他们的思考方式会受到OOP的约束，会首先以循环的结构去思考问题，这是多年面向对象编程的练习所导致的，也就很难以一个递归的想法来思考出这个函数应该怎么实现。并且与其他课程不同的是，即使你通过了测验，也没有答案提供给你，这其实对于学习一种新的编程范式不够友好，对于一个问题会显得比较茫然，无从下手，又不知如何改进自己的代码。起初我也有这种困难，虽然有使用过LINQ中高阶函数的经验，但是对于递归的写法还是不太适应。

# 高阶函数
Week 2介绍了高阶函数和Currying的概念，凭借对C#中Func的了解这些概念都是非常类似的，问题还是在于课后作业，要求实现一种集合并具备并集、差集、补集的功能。难点在于这个集合的定义是函数Int => Boolean的alias，并且没有那种listOf的功能。那么实现这一的一个集合就完全不能够用OOP的想法，存到一个array里这样去实现，而是要不停地复用高阶函数，每个操作得到的集合实际上也是lazy的。以setOf(1, 2)为例子，写法就是这样的：
```scala
type FunSet = Int => Boolean

def singletonSet(elem: Int): FunSet = 
  x => x == elem

def union(s: FunSet, t: FunSet): FunSet =
  x => s(x) || t(x)

def contains(s: FunSet, elem: Int): Boolean = s(elem)

val set1_2 = singletonSet(1) union singletonSet(2)
```
在调用contains方法的时候，OOP里就会把array中的每一个element都进行一次比较，而Scala里则是按组合的顺序执行一连串的方法。

之后作业要求实现forall与exists方法，功能和Collection.allMatch与anyMatch是类似的，不过也是用尾递归来实现，不能依赖流程控制：
```scala
def forall(s: FunSet, p: Int => Boolean): Boolean = {
  def iter(a: Int): Boolean = {
    if (a > BOUND) true
    else if (s(a) && !p(a)) false
    else iter(a + 1)
  }

  iter(-BOUND)
}

def exists(s: Set, p: Int => Boolean): Boolean =
  !forall(s, x => !p(x))
```
当时这里思考了很久应该如何去写，因为还不熟悉递归那一套，总是忘记考虑终止条件。尾递归有一点点像reverse的数学归纳法，首先给定一个基准，然后写出一个递推规则，在不断地根据基准进行递推就得到了所有的结果。在写这个函数的时候，其实并没有意识到去把函数拆成几个部分：终止条件，递归规则，内部递归函数，只是觉得这么写好复杂，还要引入一个local function，因为写C#的时候很少会把函数写成这种范式的。后来学完了整个课程之后才是豁然开朗，体会到了什么叫把状态保存在函数参数中，引入local function也是为了实现这一点，不过这就要求outer function要向其传入一个初始状态来启动“循环”。而exists也就是forall的逆否命题，通过把两个条件同时取反，就复用了已有的实现，非常巧妙。

事后推测week 2这里的题目意图是引导按照这种范式来编写递归函数，但是无奈天分不足，不像学习ReactiveX一样看的很清晰，虽然ReactiveX operator的内部实现也是以这种范式来实现的，但是这门课并没有直接在视频中讲明，使得你逐渐去真正地思考函数式编程，很有趣。

## 模式匹配
多年以来，在各种各样的C++，Java，C#课程中，说到匹配就是switch case，但是这个东西的功能实在太弱，不能包含条件呀，只能匹配常量呀，用起来还要和if组合，十分的蹩脚。随着语言的进化，真正的匹配 Pattern Matching 有了越来越多的关注，kotlin直接实现了完整的模式匹配，C# 9.0也对已有的switch进行了改进。第一次看到这个东西的时候其实并没有看懂，却在怀疑这样做的意义是什么，今天在Scala中找到了它的意义。

Scala中似乎不推荐使用if else与switch，实际上match就能实现所有的条件跳转，当它与递归结合在一起的时候，就展示了它的威力，将代码从繁复的if else中解放出来，展现出清晰的逻辑并且避免了隐藏在if中没有被cover的bug。

Scala中对List的处理也有一个很甜的语法糖，可以把List拆分为head和tail，这就是广义表的概念。并且List具有一个表尾节点Nil，它不会被计算在List内，而只是作为表尾的代表。举几个例子：
```
List("a", "b") = "a" :: "b" :: Nil
List(List(1, 0), List(0, 1)) = (1 :: 0 :: Nil) :: (0 :: 1 :: Nil) :: Nil
```
然后使用这种广义表的语法，可以将一个List拆分为head和tail，类似元组结构。当这种语法与递归和模式匹配的结合的时候，将使得代码无比优雅，不会迷失在去判断各种终止条件的面条代码里
```scala
  def combinations(occurrences: Occurrences): List[Occurrences] = {
    def combinationInner(o: Occurrences, state: List[Occurrences]): List[Occurrences] = {
      o match {
        case Nil => state
        case x :: xs => {
          val list2 = 1.to(x._2).flatMap(y => state.map(li => (x._1, y) :: li))
          val list3 = 1.to(x._2).map(y => List((x._1, y)))
          combinationInner(xs, state concat list2 concat list3)
        }
      }
    }

    combinationInner(occurrences.reverse, List(List()))
  }
```
如果是归并排序的话，这种优雅会体现的更明显，在C实现的归并排序中，要判断边界，要在参数中传递上下界，但是在Scala中做到了专注于业务，从重复劳动中解放出来。
```scala
  def mergedOccurrences(lhs: Occurrences, rhs: Occurrences): Occurrences = {
    def merge(lhs: Occurrences, rhs: Occurrences, state: Occurrences): Occurrences = {
      (lhs, rhs) match {
        case (Nil, Nil) => state.reverse
        case (_, Nil) => state.reverse ::: lhs
        case (Nil, _) => state.reverse ::: rhs
        case (x :: _, y :: _) =>
          if (x._1 < y._1) merge(lhs.tail, rhs, x :: state)
          else if (x._1 > y._1) merge(lhs, rhs.tail, y :: state)
          else merge(lhs.tail, rhs.tail, (x._1, x._2 + y._2) :: state)
      }
    }

    merge(lhs, rhs, List())
  }
```

Scala的初学体验好吗？说实话，很不好，因为要换一种思维模式是很困难的，尤其是在目前而言，函数式编程确实具有一定的门槛，并且sbt这个构建工具也难以上手，编译速度堪比Spring的启动速度。但是，在这几周内对Scala几个小小的方面进行了解后，逐渐地喜欢上了这种精致而没有脱离主流的感觉，看到了Kotlin中存在着许多Scala的影子。这门课对于介绍FP确实不错，不过缺少了一点点编写真正的应用程序的练习，但是它让我喜欢上了Scala，这就足够了。