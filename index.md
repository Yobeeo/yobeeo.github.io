---
layout: page
title: "Yobeeo's Blog"
---

欢迎来到我的博客！这里记录技术学习与生活点滴。

---

## 📝 最新文章

<ul class="post-list">
  {% for post in site.posts limit:5 %}
    <li>
      <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      {% if post.categories %}
        <span class="post-categories">
          {% for cat in post.categories %}
            <code>{{ cat }}</code>
          {% endfor %}
        </span>
      {% endif %}
    </li>
  {% endfor %}
</ul>

{% if site.posts.size > 5 %}
> 更多文章请查看[归档页面]()。
{% endif %}

---

## 🔗 链接

- [GitHub](https://github.com/Yobeeo)
- [关于我]({{ '/about/' | relative_url }}){:target="_blank"}

---

<style>
.post-list {
  list-style: none;
  padding: 0;
}
.post-list li {
  padding: 8px 0;
  border-bottom: 1px dashed #ddd;
}
.post-date {
  color: #888;
  margin-right: 12px;
  font-size: 0.9em;
}
.post-categories code {
  font-size: 0.8em;
  margin-left: 8px;
  background: #f0f0f0;
  padding: 1px 6px;
  border-radius: 3px;
}
</style>
