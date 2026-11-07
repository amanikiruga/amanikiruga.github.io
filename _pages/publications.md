---
layout: page
permalink: /publications/
title: publications
description: my academic papers.
nav: false
nav_order: 2
---
<!-- _pages/publications.md -->
<div class="publications">

<h2 class="year">selected</h2>
{% bibliography -f {{ site.scholar.bibliography }} -q @*[selected=true]* --group_by none %}

<h2 class="year">all</h2>
{% bibliography -f {{ site.scholar.bibliography }} %}

</div>
