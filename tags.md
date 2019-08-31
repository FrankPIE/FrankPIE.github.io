---
layout: page
title: Tags 

---

<div class="page-content wc-container">
	<div class="post">
		<h1>标签</h1>  
		<ul>
			{% for tag in site.tags %}					
				<h3>{{ tag[0] }}</h3>
				<ul>
					{% for post in site.posts %}
        			{% for tag_value in post.tags %}
          			{% if tag_value == tag[0] %}
          			<li class="archive_list">
            			<time>{{ post.date | date_to_string }}</time>
            			<a class="archive_list_article_link" href='{{ post.url | relative_url }}'>{{post.title}}</a>
            			<p>{{post.description}}</p>
          			</li>
          			{% endif %}
        			{% endfor %}
      				{% endfor %}
				</ul>
			{% endfor %}
		</ul>
	</div>
</div>