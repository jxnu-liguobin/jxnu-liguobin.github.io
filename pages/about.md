---
layout: page
title: About
description: 梦境亦是美，醒来亦是空
keywords: jxnu-liguobin, 梦境迷离
comments: true
menu: 关于
permalink: /about/
---

1. Scala2 Contributor 和 Reviewer
2. [scala/docs.scala-lang](https://github.com/scala/docs.scala-lang) Reviewer
3. [Scala Cookbook2](https://github.com/bitlap/ScalaCookbook) 译者

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
</ul>

## 技术栈

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
