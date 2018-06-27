---
layout: page
title: About
permalink: /about/
published: true
---

<div class="page" markdown="1">

{% capture page_subtitle %}
<img
    class="me"
    alt="{{ author.name }}"
    src="{{ site.author.photo | relative_url }}"
    srcset="{{ site.author.photo2x | relative_url }} 2x"
/>
{% endcapture %}

{% include page/title.html subtitle=page_subtitle %}

90后小码农，VR行业从业者，会点C++，懂点图形学，搞过几年引擎，经常干些杂活，在这里写点东西，交点朋友。
</div>
