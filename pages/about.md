---
layout: page
title: About
description: 歇斯底里的本源
keywords: Lingxiang Tai
comments: true
menu: 关于
permalink: /about/
---

哈哈哈


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
