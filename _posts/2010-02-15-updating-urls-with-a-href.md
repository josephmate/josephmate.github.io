---
layout: post
title: Updating URLs with a href
date: 2010-02-15 22:43
author: matejoseph
comments: true
categories: [Uncategorized]
---
You can put parameters into the URL with javascript. Here is an example html document:
```html
<html>
<body>
	<a href='#test=test'>testing</a>
</body>
</html>
```

So with Markus's submission browse page,  instead of having:
```html
<span class='ap_next_link'>
  <a href='#' onclick='ajax stuff'>Next </a>
</span>
```
We can have something like:
```html
<span class='ap_next_link'>
  <a href='#<PARAMS GO HERE>' onclick='ajax stuff'>Next </a>
</span>
```

However, that means with every AJAX call that flips the page on the table, will will also have to update all the links on the page as well.

