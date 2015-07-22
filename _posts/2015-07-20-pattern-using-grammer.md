---
layout: post
title: 神奇的单例
categories: grammer
---
深深被下面这个单例折服了，这可能是静态内部类使用的正确方式。
静态内部类私有的成员也可以访问啊 ==!!!
{% highlight java %}
public class Singleton {

	private Singleton() {}

	private static class InstanceHolder{
		private static Singleton instance = new Singleton();
	}

	public static Singleton getInstance() {
		return InstanceHolder.instance;
	}
}
{% endhighlight %}


这个版本的单例更是让人表示压力很大啊
对class做同步，对静态成员做volatile，真是...==
{% highlight java %}
public class Singleton {  
    private static volatile Singleton instance;  
  
    private Singleton() {  
  
    }  
  
    public static Singleton getInstance() {  
        if (instance == null) {  
            synchronized (Singleton.class) {  
                instance = new Singleton();  
            }  
        }  
        return instance;  
    }  
}  
{% endhighlight %}