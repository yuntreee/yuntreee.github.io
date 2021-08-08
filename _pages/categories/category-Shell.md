---
title: "Shell Script"
layout: archive
permalink: categories/Shell
author_profile: true
sidebar:
    nav: "docs"
---


{% assign posts = site.categories.Shell %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}