---
layout: home
---

<div class="index-content project">
    <div class="section">
        <ul class="artical-cate">
            <li><a href="/"><span>生活滴水</span></a></li>
            <li class="on"><a href="/project"><span>技术击石</span></a></li>
        </ul>

        <div class="cate-bar"><span id="cateBar"></span></div>

        <ul class="artical-list">
        {% for post in site.categories.project %}
            <li>
                <h2>
                    <b>>></b><a href="{{ post.url }}">{{ post.title }}</a>
                </h2>
                <div class="title-desc"><h3>文章简介：</h3><b>{{ post.date|date:"%Y-%m-%d" }}</b> :&nbsp;{{ post.description }}</div>
            </li>
        {% endfor %}
        </ul>
    </div>
    <div class="aside">
    </div>
</div>
