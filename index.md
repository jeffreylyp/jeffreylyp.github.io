---
layout: home
---
<div class="index-content blog">
    <div class="section">
        <ul class="artical-list">
        {% for post in site.posts %}
            <li>
                {{ page.date|date:"%Y-%m-%d" }} >> <a href="{{ post.url }}">{{ post.title }}</a>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
    </div>
</div>
