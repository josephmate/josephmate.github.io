---
layout: post
title: Javascript to Redirect a Drop Down List (select)
date: 2010-03-05 00:05
author: matejoseph
comments: true
categories: [drop down list, html select, javascript, Javascript, redirect]
---
You can redirect using an html select without having to place it in a form, or have the user press a button. The user only needs to change the value.

Check out the following code that you can copy and paste into an html file and take for a test drive:
```html
<html>
	<head>
		<title>Drop Down List Redirect</title>
	</head>

	<body>
		<select onchange='top.location.href = 'http://www.google.com/search?q='
				+ this.options[ this.selectedIndex ].value' >
			<option value=''>None</option>
			<option value='cute+dogs'>Cute Dogs</option>
			<option value='lasers+beams'>Laser Beams</option>
			<option value='kitty+cat'>Kitty Cat</option>
		</select>
	</body>
</html>
```

Breaking down the javascript code into more understandable chunks gives:
```javascript
// 'this' points to the select object after change the item in the drop down list.
var drop_down = this;

// drop_down.selectedIndex contains the position of the item that was selected from the drop down
var selected_index = drop_down.selectedIndex;

// drop_down.options contains all of html option elements inside the html select
// we need to go to .value to get the 'value='something'' written in the HTML
var selected_value = drop_down.options[ selected_index ].value;

// changing top.location.href redirects ( unless you only append #blah )
top.location.href = 'http://www.google.com/search?q=' + selected_value
```

I hope you guys find this helpful!

Cheers,
Joseph
