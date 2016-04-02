---
layout: post
comments: true
title:  "React Test utils"
date:   2016-2-28
---

# A journeyperson's guide to testing React components

## When to use the Shallow Renderer

Wat is the Shallow Renderer?
<li>**Experimental** - Beware!</li>
<li>Renders your component one layer deep (doesn't render child components)</li>
<li>Does NOT render into a DOM element</li>
<li>yo dawg</li>

In this particular case we are using mocha and should, but using other testing frameworks are similar.

Here is the component we will be testing.

<!-- <div class="markdown-body"> -->
{% include 2016-2-28-react-test-utils/SimpleComponent.md %}
<!-- </div> -->
