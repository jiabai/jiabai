---
layout: post
title: Method Resolution Order – Python类的方法解析顺序
description: 在支持多重继承的编程语言中，查找方法具体来自那个类时的基类搜索顺序通常被称为方法解析顺序(Method Resolution Order)，简称MRO。
category: blog
---

在支持多重继承的编程语言中，查找方法具体来自那个类时的基类搜索顺序通常被称为方法解析顺序(Method Resolution Order)，简称**MRO**。(Python中查找其它属性也遵循同一规则。)对于只支持单重继承的语言，MRO十分简单；但是当考虑多重继承的情况时，MRO算法的选择非常微妙。Python先后出现三种不同的MRO：经典方式、Python2.2 新式算法、Python2.3 新式算法(也称作C3)。Python 3中只保留了最后一种，即C3算法。

经典类采用了一种简单MRO机制：查找一个方法时，搜索基类简单的深度优先从左到右。搜索过程中第一个匹配的对象作为返回结果。例如，考虑如下类：

    class A:
      def save(self): pass

    class B(A): pass

    class C:
      def save(self): pass

    class D(B, C): pass

对于类D的实例x，它的经典MRO结果是类D，B，A，C顺序。因此查找方法x.save()将会得到A.save()(而不是C.save())。这个机制对于简单情况工作良好，但是对于更复杂的多重继承关系，暴露出的问题就比较明显了。其中问题之一是关于”菱形继承”时的方法查找顺序。例如：

    class A:
      def save(self): pass

    class B(A): pass
                           
    class C(A):
      def save(self): pass
                                                
    class D(B, C): pass

此处类D继承自类B和类C，类B和类C均继承自类A。应用传统的MRO，查找方法时在类中的搜索顺序是D, B, A, C, A。因此，语句x.save()将会如前一样调用A.save()。然而，这可能非你所需！既然B和C都从A继承，别人可以争辩说重新定义的方法C.save()可以被看做是比类A中的方法”更具体”(实际上，有可能C.save()会调用A.save())，所以C.save()才是你应该调用的。如果save方法是用于保持对象的状态，不调用C.save()将使C的状态被丢弃，进而程序出错。

虽然当时的Python代码很少存在这种多重继承代码，”新类”(new-style class)的出现则使之成为常见现象。这是由于所有的新类都继承自object这一基类。因此涉及到新类的多重继承总是会产生前面所述的菱形关系。例如：

    class B(object): pass

    class C(object):
      def __setattr__(self, name, value): pass

    class D(B, C): pass

而且，object定义了一些方法(例如__setattr__())可以由子类扩展，这时解析顺序更是至关重要。上面代码所属例子中，方法C.__setattr__应当被应用到类D的实例。

为了解决在Python2.2引入的新类所带来的方法解析顺序问题，我采取的方案是在类定义时就计算出它的MRO，并存储为该类对象的一个属性。官方文档中MRO的计算方法为：深度优先，从左到右遍历基类，这个与经典MRO一致，但是如果任何类在搜索中是重复的，只有最后一个出现的位置被保留，其余会从MRO list中删除。因此我们前面这个例子中，搜索顺序将会是D, B, C, A(经典类采用经典MRO，则会是D, B, A, C, A)。

实际上MRO的计算比文档所说的要更复杂。我发现某些情况下新的MRO算法结果不理想。因此还存在一个特例，用于处理两个基类在两个不同的派生类中顺序不同，而这两个派生类又被另外一个类继承的情况。如下代码所示：

    class A(object): pass
    class B(object): pass
    class X(A, B): pass
    class Y(B, A): pass
    class Z(X, Y): pass

利用文档中描述的新MRO算法，Z关于这些类的MRO为Z, X, Y, B, A, object。(这儿的object是通用基类。)。然而，我不希望结果中B出现在A之前。因此实际的MRO会交换其顺序，产生Z, X, Y, A, B, object。直观上说，算法尝试保持基类在搜索时过程中首次出现的顺序。例如，对于类Z，他们基类X应当被首先搜索到，因为在继承的list中排序最靠前。既然X继承自A和B，MRO算法会尝试保持其顺序。这是在我在Python2.2中实际实现的算法，但是文档只提到了前面的不包括特例处理的算法(我幼稚的认为这点小差别不需明言)。

然而，就在Python 2.2 引入新类不久，Samuele Pedroni就发现了文档中MRO算法与实际代码中观察到结果不一致的现象。而且，在上述特例之外也可能发生不一致情况。详细讨论的结果认为Python2.2采用的MRO是坏的，Python应当采用C3线性化算法，该算法详情见论文”A Monotonic Superclass Linearization for Dylan”(K. Barrett, et al, presented at OOPSLA’96)。

本质上，Python 2.2 MRO的主要问题在于不能保持单调性(monotonicity)。在一个复杂的多层次继承情况，每个继承关系都决定了一个直接的查找顺序，如果类A继承类B，则MRO明显应当先查找A后查找B。类似，如果类B多重继承类C和类D，则搜索顺序中类B应该在类C之前，且类C应该在类D之前。

在复杂的多层次继承情况，始终能满足这一规则就称为保持了单调性。也就是说，当你决定类A会在类B之前查找到，应当再也不会遇到类B需要在类A之前查找的情况(否则，结果是未定义的，应该拒绝这种情况下的多层次继承)。以前的MRO算法未能做到这一点，新的C3算法则在保证单调性方面发挥了效用。基本上，C3的目的在于让你依据复杂多层次继承中所有的继承关系进行排序，如果所有顺序关系都能满足，则排序结果就满足单调性。否则，无法得到确定的顺序，算法会报错，拒绝运行。

于是，在Python 2.3，摈弃了我手工作坊的 2.2 MRO算法，改为采用经过学术审核检验的C3算法。带来的结果之一就是当多层次继承中存在基类顺序不一致情况时，Python将会拒绝这种类继承。还是参考前面例子，对于类X和类Y就存在顺序上不一致，对于类X，规则判定类A应该在类B之前被检查。然而对于类Y，规则认为类B应该在类A之前被检查。单独情况下，这种不一致是可以接受的，但是如果X和Y共同作为另外一个类(例中定义的类Z)的基类，C3算法就会拒绝这种继承关系。这个也算是对应了Zen of Python的”errors should never pass silently”规则。

------

- 英文原文链接: [Here][1]
- 原文作者: Guido van Rossum
- 译者: [Jabari-Bi][2]

[1]: http://python-history.blogspot.com/2010/06/method-resolution-order.html
[2]: http://weibo.com/jiabai

