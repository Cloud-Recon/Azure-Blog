---
layout: default
title: "Cloud-Recon — Home"
---

<!-- Hero -->
<div align="center">
  <h1>👋 Welcome to Cloud-Recon</h1>
  <p>Real‑world Azure networking from the field — designs, gotchas, and lessons learned.</p>
  <img src="{{ '/assets/images/Cloud-Recon.webp' | relative_url }}" alt="Wayne and Dogs" style="max-width: 760px; border-radius: 12px; margin: 10px 0 6px;" />
  <p><a href="{{ '/about/' | relative_url }}">About Wayne</a></p>
</div>

---

## 📚 Latest Posts

<!--
This section updates itself.
Each time you add a Markdown file into _posts (YYYY-MM-DD-title.md),
it will appear here automatically with its date, title, and excerpt.
-->
<ul style="list-style: none; padding-left: 0;">
{% assign posts = site.posts | sort: 'date' | reverse %}
{% for post in posts %}
  <li style="margin: 0 0 1.1rem;">
    <h3 style="margin: 0 0 .25rem;">
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </h3>
    <small>{{ post.date | date: "%d %b %Y" }}</small>
    {% if post.excerpt %}
      <p style="margin:.35rem 0 0;">{{ post.excerpt | strip_html | truncate: 180 }}</p>
    {% endif %}
  </li>
{% endfor %}
</ul>

---

## 🔭 What’s coming
- Azure Route Server deep dive (publishes Fri 15th Aug)
- Azure Virtual WAN
- Azure Firewall


