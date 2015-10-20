---
layout: post
title:  "常用Shell技能"
date:   2015-10-20 15:15:54
categories: Linux
excerpt: 只记录平时开发中会长用的，太全太详细的shell命令没有必要。
---

* content
{:toc}

本文只记录一下平时开发经常会用到的Shell命令，一些细节记录下来方便以后查找。

---

##重定向

**期望每个程序的输出都是其他程序的输入，即使是未知的程序。**这是Unix哲学里让我感受最深的一条。在[从LinkedIn,Apache Kafka到Unix哲学](http://www.jointforce.com/jfperiodical/article/1036?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)一文中，对此有更加鲜明的说明。
这里我们将主要关注两个符合: **>**和**|**。

---

###重定向符: **>**

ls命令默认将结果输出到屏幕，使用“>”重定向符可以将所得结果输出到指定文件中。
<pre><code># ls -l /usr/bin > out.txt
</code></pre>

重复运行上面的命令，发现out.txt中的文件一直不变，这是因为当我们使用">"来重定向输出结果时，目标文件总是从开头被重写。如果想把重定向结果追加到文件内容后面，就使用">>"重定向符。
<pre><code># ls -l /usr/bin >> out.txt
</code></pre>

---

###管道符: **|**

---
