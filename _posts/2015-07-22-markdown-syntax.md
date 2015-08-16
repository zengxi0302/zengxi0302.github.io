---
layout: post
title:  "Markdown常用语法标记"
date:   2015-07-22 07:30:00
categories: 写作
---

本文的markdown标签针对ruby实现的kramdown解释器

#ruby代码段
{% highlight ruby %}
int main()
{
	printf("hello github.\n");
}
#=> prints 'hello github.' to STDOUT. 
{% endhighlight %}

#c代码段
{% highlight c %}
    #include <stdio.h>
    int main(){
    printf("hello");
}
{% endhighlight %}

#shell片段
{% highlight bash %}
 	ls | while read line; done echo $line; done
{% endhighlight %}

#标题示例
------

#H1 `#` 

##H2 `##`

###H3 `###`

####H4 `####`

#####H5 `#####`

######H6 `######`

#强调
------

*强调1-1* //用`*`表示斜体

_强调1-2_ //用`_`也可以

**强调2-1** //用`**`表示粗体

__强调2-2__ //用`__`也可以

***强调3-1*** //用`***`表示斜粗体

___强调3-2___ //用`___`也可以

****这是什么**** //四个`*`会被重新翻译，就不用了

#列表
------

颜色

* 红

	a. 朱红

	b. 酒红

* 黄
	1. 深黄
	
	2. 淡黄 

#引用
------

> 这是一段引用,用`>`表示

#字体颜色
------

*红色*{:style="color:red"}. `*红色*{:style="color:red"}`

*绿色*{:style="color:green"}.

*黄色*{:style="color:yellow"}.

#链接
------
[www.google.com](https://www.google.com)  `[www.google.com](https://www.google.com)`

#图片
------
![github](/images/github.png "github")  `![github](/images/github.png "github")`

#表格
------

| head1 | head2 | head3 |
| ----- |:----- |:----- |
| a     | b     | c     |


###sample2

Markdown | Less | Pretty
--- | --- | ---
*Still* | `renders` | **nicely**
1 | 2 | 3
