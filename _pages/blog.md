---
permalink: /blog
layout: page
title: Blog
---

<h2>My Blog Posts</h2>
<ul>
  {% for post in site.posts %}
    <li>
      <a href=".{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

<h2>In ChartMogul</h2>
<ul>
  <li>
    <a href="https://chartmogul.com/blog/how-we-reduced-snowflake-costs-by-70-a-practical-optimization-guide/">How we reduced Snowflake costs by 70%: a practical optimization guide</a>
  </li>
  <li>
    <a href="https://chartmogul.com/blog/building-a-data-engineering-stack-that-boosts-scalability/">Building a data engineering stack that boosts scalability</a>
  </li>
  <li>
    <a href="https://chartmogul.com/blog/how-migrating-our-database-eliminated-data-processing-incidents/">How Migrating Our Database Eliminated Data Processing Incidents</a>
  </li>
</ul>