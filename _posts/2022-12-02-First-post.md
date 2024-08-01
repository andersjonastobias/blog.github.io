---
layout: post
title: "First post"
date: 2022-12-02
---

### This is the first test  

{% capture summary %}SUMMARY{% endcapture %}  
{% capture details %}  
DETAILS  
{% endcapture %}{% include details.html %} 

<details markdown=block>
<summary markdown=span>A *Summary*</summary>
These are the **details** for this item.

test.
</details>