---
title: "Python"
layout: archive
permalink: categories/Python
author_profile: true
sidebar:
    nav: "docs"
---


{% assign posts = site.categories.Python %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}