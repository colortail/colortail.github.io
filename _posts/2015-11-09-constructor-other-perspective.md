---
layout: post
title: ���캯������һ���ӽ�
categories: grammer
---

{% highlight c %}
class Strange{
public:
	Strange() {}
	Strange(const vector<int> arg) {}
};

Strange foo1(){
	Strange s(vector<int>());
	return s;
}

Strange foo2(){
	Strange s();
	return s;
}
Strange foo3(){
	Strange s;
	return s;
}

Strange foo4(){
	return Strange(vector<int>());
}

Strange foo5(){
	Strange s(vector<int>(1,1));
	return s;
}
{% endhighlight %}

�������ֶԶ���Ĺ��췽����ǰ���ֶ��ǲ��Եģ�