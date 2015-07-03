---
layout: post
title: 函数参数不被修改的语法元素
categories: grammer
---
首先，因为函数内部会创建形式参数，所以若参数为基本类型，即传递方式是所谓的按值传递，则函数不会改变任何参数。

所以若函数会改变参数值，针对的也只是对象。

##in C++

在函数参数表中，加入限定常量的const关键字会使参数所指向的对象不能在函数体内被改变。情况有三种
{% highlight c %}
    void foo(Type argu1, Type argu2) const;
    void foo(const Type argu);
    void foo(Type const argu);
{% endhighlight %}


第一种方式就是不能改变对象内容。而第二种和第三种，根据const的不同位置，实际差别还是很大的。目前看来，第三种的写法才合适。

第三种是表示该参数内的对象不可被修改，而第二种表示的是：传递来的参数是一个const常量。

虽然直接限定一个量为常量确实不能让它在函数内被改变，但和本意相悖。因为常量无论在函数内还是函数外，哪都不能被修改，所以过于严格了。
在下最初老是将const写在参数类型前，所以编译器总提示不能将非const转化为const，尽管我印象里非const转const有什么错呢？

另一种常见情况就是函数体内还有另外一个函数，而内层函数可能修改参数。这时需要限定内层函数。
{% highlight c %}
    void foo(Type const argu) {
      f(argu)；
    }
{% endhighlight %}
##in Java

在Java里，和const对应的有一个final关键字，他们都可以限制一个量为常量。但在参数传递的时候，final和const可完全不是一回事。

final不能表示一个对象在函数内不可修改。Java里就没有机制去保护参数对象不被修改。
但是确实有这种写法，而且这种写法在某些情况下是必须的：
{% highlight java %}
    public void foo(final Type argu) {
      ...？
    }
{% endhighlight %}
这种情况是函数内有匿名对象，且匿名对象使用了这个参数。

在函数内使用匿名对象，匿名对象若要使用一个对象成员，则该成员得是final。原因如下，该匿名对象可能在函数所属对象消亡时仍然存在，因此被使用的变量也得在对象消亡时也存在，也就是只能为常量。这个问题很常见了。
{% highlight java %}
    public final Type var;
    public void foo() {
      return new Abstract(var,v);
    }
{% endhighlight %}
而被匿名对象调用的可不仅仅是类成员，也可以是局部变量，甚至形式参数，但是出于同样的原因，这些变量也只能是final。

因此用Java时，在参数中使用final的情况不再是对象参数变不变的问题，而是匿名内部对象的问题，这完全不是同一个问题。所以只有在函数体内使用匿名内部对象时（且该对象调用该形参）才会在参数表里用final限定参数，完整的情况应该类似于：
{% highlight java %}
    public void foo(final Type argu) {
      ...
      return new Abstract(argu);
    }
{% endhighlight %}