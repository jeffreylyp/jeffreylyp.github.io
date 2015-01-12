---
layout: home
---
<div class="index-content blog">
    <div class="section">
        <ul>
        {% for post in site.posts %}
            <li>
                <span>{{ post.date | date_to_string }}</span> >> <a href="{{ post.url }}">{{ post.title }}</a>
                <div class="title-desc">{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
    </div>
</div>
