---
title: "Ansible"
layout: archive
permalink: categories/Ansible
author_profile: true
sidebar:
    nav: "docs"
---


{% assign posts = site.categories.Ansible %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}