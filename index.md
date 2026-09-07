---
layout: default
---

# Running more than one agent

Three posts on the layer that appears once an organisation has several agentic
harnesses and no shared definition of done: a contract, a coordination plane,
and an execution runtime.

<ol class="index">
{% assign ordered = site.posts | sort: "series_order" %}
{% for p in ordered %}
  <li>
    <a href="{{ p.url | relative_url }}">{{ p.title }}</a>
    <p>{{ p.excerpt | strip_html | truncatewords: 28 }}</p>
  </li>
{% endfor %}
</ol>
