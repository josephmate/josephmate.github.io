---
layout: post
title: Unintuitive Scoping and Javascript Closures
date: 2012-03-04 08:53
author: matejoseph
comments: true
categories: [closure, javascript, Javascript, scope, scoping]
---
I had this problem at work, and have simplified it and removed all context related to the project so this article can be easily consumed. The problem arises when you try to create a closure onto a local variable initialized in a for loop.

I will begin with an example that matches my scoping intuition, which does not use a closure:

```javascript
function setupArray() {
  var array = new Array();
  var i=0;
  for (i=0;i<=4;i++) {
    var tmp;
    tmp = i + 1;
    array[i] = tmp;
  }
  return array;
}

function outputArray() {
  var array = setupArray();
  for (i=0;i<=4;i++) {
    document.write(array[i] + '<br/>');
  }
}

outputArray();
```

With this example, we get the output that matches our intuition:

```javascript
1
2
3
4
5
```

Now when we modify the code to use a closure instead and we run into problems.

```javascript
function setupArray() {
  var array = new Array();
  var i=0;
  for (i=0;i<=4;i++) {
    var tmp;
    tmp = i + 1;
    array[i] = function() {
      return tmp;
    }
  }
  return array;
}

function outputArray() {
  var array = setupArray();
  for (i=0;i<=5;i++) {
    document.write(array[i]() + '<br/>');
  }
}
outputArray();
```

Here is the output that I was not expecting.

```javascript
5
5
5
5
5
```

All the anonymous functions bind to the same variable reference. It is not creating a new variable for each iteration of the loop. My solution was to just place the contents of the loop into a function.

```javascript
function setupArrayEntry(i) {
  var tmp;
  tmp = i + 1;
  return function() {
    return tmp;
  }
}

function setupArray() {
  var array = new Array();
  var i=0;
  for (i=0;i<=4;i++) {
    array[i] = setupArrayEntry(i);
  }
  return array;
}
```

This fixes the problem.

```javascript
1
2
3
4
5
```

After posting this I did some additional research and discovered that this seems to be a common problem that developers encounter. For example, <a href="http://robertnyman.com/2008/10/09/explaining-javascript-scope-and-closures/">this</a> post extremely valuable in understanding what is going on. The author also provides a similar solution. Check it out!
