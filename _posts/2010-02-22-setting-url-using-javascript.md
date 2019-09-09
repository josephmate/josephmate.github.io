---
layout: post
title: Setting URL using Javascript
date: 2010-02-22 02:55
author: matejoseph
comments: true
categories: [Uncategorized]
---
Severin just pointed out a much easier way to update the URL using javascript:
```html
<script language='javascript' type='text/javascript'>
   window.location.href = '/main/some_controller/some_action#some_anchor';
</script>
```

Thanks Severin!
I have extended this code into an example you can copy and paste into an html file and play around with:
```html
<html>
<head><title>URL Updating</title></head>
<body>

<input    type='button' 
             name='cool'
             value='cool'
             onclick='window.location.href = window.location.href + '#value=param';' />

</body>
</html>
```

With Markus, this leaves lots of places for us to place the code. For a link_to_remote, we can place set the window.location.href in the :before or :complete variables. Example:
markus/app/views/ajax_paginate/_initial_paginate_links_alpha.html.erb
```ruby
<%= link_to_remote '<< ' + t('pagination.first'), :url => {
      :action => action,
      :id => assignment.id,
      :filter => filter,
      :page => 1,
      :per_page => per_page,
      :sort_by => sort_by,
      :alpha_category => alpha_pagination_options[0],
      :update_alpha_pagination_options => 'false'
    },
    :before => 'ap_thinking_start('#{table_name}');',
    :complete => 'ap_thinking_stop(); window.location.href = window.location.href + '#value=param';' %>
```

Notice that I stuck it right after the :complete => "ap_thinking_stop(); Additionally, notice that we are no longer limited to a's. We can not apply this to any html objects.

Cheers,
Joseph

