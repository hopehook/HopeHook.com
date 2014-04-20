---
date: 2014-01-23
layout: post
title: 第一次提交我的博客代码到github用到的命令
categories: 教程
tags: github
---

From：[Click me](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)
---

{% highlight cpp %}
    f:
{% endhighlight %}

---

{% highlight cpp %}
    cd github\Blog
{% endhighlight %}

---

{% highlight cpp %}
    git init
{% endhighlight cpp %}
---

{% highlight cpp %}
    git checkout --orphan gh-pages
{% endhighlight %}
---

{% highlight cpp %}
    git add .
{% endhighlight %}
---

{% highlight cpp %}
    git commit -m "first post"
{% endhighlight %}
---

{% highlight cpp %}
    git remote add origin https://github.com/hopehook/Blog.git
{% endhighlight %}
---

{% highlight cpp %}
    git push origin gh-pages
{% endhighlight %}
---

 
