---
layout: page
title: About
description: 梦境亦是美，醒来亦是空
keywords: jxnu-liguobin, 梦境迷离
comments: true
menu: 关于
permalink: /about/
---

1. Scala2 REPL和编译器Contributor
2. [scala/docs.scala-lang](https://github.com/scala/docs.scala-lang)中文审核人
3. [lightbend-labs/scala-logging](https://github.com/lightbend-labs/scala-logging) Maintainer
4. [Scala Cookbook2](https://github.com/bitlap/ScalaCookbook) 译者
5. [https://github.com/apache/incubator-pekko](https://github.com/apache/incubator-pekko) Committer  ~~~~~~~ 逃 hh

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
{% if site.url contains 'dreamylost.cn' %}
<li>
微信公众号：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ assets_base_url }}/assets/images/qrcode.jpg" alt="梦境迷离" />
</li>
{% endif %}
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
