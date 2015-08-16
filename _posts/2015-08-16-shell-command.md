---
layout: post
title:  "常用的shell命令记录"
date:   2015-08-16 20:54:00
categories: linux
---

##find命令

{% highlight bash %}
find / -name file		//根据name进行查找
find / -iname file		//无关大小写
find / -name "*.sh"			//通配符，正则表达式
find / -name "[A-Z]*"		//以A-Z为开头的文件

find / -size +5M				//根据size进行查找
find / -size -5M
find / -size +5M -a size -10M		//查找大于5M小于10M的文件, 条件and
find / -size +5M -o -name file	//大于5M，或名称是file，条件or

find / -atime -1				//查找1天没访问过的文件
find / -atime +30				//查找30内都天没访问过的文件
find / -amin -5 				//查找5分钟内访问过的文件
							//其它的还有ctime, mtime, cmin, mmin，
							//表示创建和修改时间，单位是天和分钟

find / -user root				//查找属主是root的文件

find / -type f 					//查找所有的文件 - (f 文件; d 目录; p 管道;

#find常与xargs配合使用
find / -size +100M | xargs rm	//查找大于100M的文件，并删除

#也后跟exec跟命令执行
find / -type f -exec mv {} tmp/ \;	//将匹配的文件移动到tmp目录
{% endhighlight %}

##cut命令
{% highlight bash %}
cut -c -5 file			//取文件前5个字符
cut -c 5- file			//取文件第五个字符后的所有字符
cut -c 10-15 file		//取10-15个字符

cut -d " " -f 1			//以" "为分割符，取field 1
cut -d : -f 1,3-5,7		//以:为分割符，取1，3-5，第7个字段
{% endhighlight %}

##sed命令
{% highlight bash %}
sed -i "s/a/b/g" file	//将file中所有的a替换成b
{% endhighlight %}

##awk命令
{% highlight bash %}
echo "1 2 3 4 5 " | awk '{print $1}'	//取第一个字段

{% endhighlight %}
