---
layout: post
title: Rails, AJAX, Back Buttons, and Bookmarks
date: 2010-02-20 01:04
author: matejoseph
comments: true
categories: [Uncategorized]
---
Markus's submission page uses AJAX to grab the next or previous page of the table of submissions. However, this breaks bookmarks and the back button. So the goal is to do something similar to gmail. They update everything after the anchor (#) in the URL. Here is what we can do with Markus with Ruby on Rails, to do the exact same thing.

My examples are only going to cover the implementation of the Next > button. However, this can apply to any AJAX link.

The part of code that generates the Next link is in:
_initial_paginate_links.html.erb:
```ruby
   <%= link_to_remote 'Next >', :url => {
         :action => 's_table_paginate',
         :id => assignment.id,
         :filter => filter,
         :page => current_page + 1,
         :per_page => per_page,
         :sort_by => sort_by
       },
       :before => 'ap_thinking_start();',
       :complete => 'ap_thinking_stop();' %>
```

When the page loads it generates:
```html
<a onclick='ap_thinking_start();; new
Ajax.Request('/main/submissions/s_table_paginate/1?filter=none&amp;amp;page=3&amp;amp;per_page=30&amp;amp;sort_by=group_name',
{asynchronous:true, evalScripts:true,
onComplete:function(request){ap_thinking_stop();},
parameters:'authenticity_token=' +
encodeURIComponent('hjLTZo+xwhfYVOjA6E6Cbt3mm0SoaJw3t+nQG9UF/iA=')});
return false;'
href='#'>Next ></a>
```

However, we want the URL to update, so we need to get rid of the " return false; " at the end of onclick. I accomplished that by using link_to and the remote_function helpers that Rails provides.

_initial_paginate_links.html.erb:
```ruby
   <%= link_to 'Next >', '#value=param',
         : onclick =>
           remote_function(
             :url => {
               :action => 's_table_paginate',
               :id => assignment.id,
               :filter => filter,
               :page => current_page + 1,
               :per_page => per_page,
               :sort_by => sort_by
             },
             :before => 'ap_thinking_start();',
             :complete => 'ap_thinking_stop();'
           )
 %>
```

Which generates:
```html
<a onclick='ap_thinking_start();; new
Ajax.Request('/main/submissions/s_table_paginate/1?filter=none&amp;amp;page=3&amp;amp;per_page=30&amp;amp;sort_by=group_name',
{asynchronous:true, evalScripts:true,
onComplete:function(request){ap_thinking_stop();},
parameters:'authenticity_token=' +
encodeURIComponent('hjLTZo+xwhfYVOjA6E6Cbt3mm0SoaJw3t+nQG9UF/iA=')});'
href='#value=param'>Next ></a>
```

Notice that the return false is no longer there! If you try to click the Next > button, then it will place #value=param at the end of the url without reloading the page.

Now all that remains is figuring out how to do this with an html form. For example drop downlists, with an onchange method that updates the page using AJAX but still appends #value=param to the end of the URL.

Cheers,
Joseph

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="30"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>