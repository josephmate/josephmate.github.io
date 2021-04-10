---
layout: post
title: will_paginate and markus
date: 2010-02-15 21:51
author: matejoseph
comments: true
categories: [Uncategorized]
---
Problem Experienced:
The page number is not saved in the URL when flipping through pages.

At first we thought it was something to do with will_paginate. However, if you take a look at: http://railscasts.com/episodes/51-will-paginate . You will notice that will_paginate successfully puts the parameters into the URL. It's not will_paginate; it's the way we're using will_paginate.

So now I am looking at where the Rail's params[] are used. Markus does something differently with params[] compared to the simple website from the railscast. Additionally, I'm looking through all the AJAX calls we make on the submissions page. Finally, I am trying to understand how s_table_paginate is participating in pagination.

On a side note:
Mike gave me a nice console script to populate the submissions page for markers.
Just run script/console and place paste the following code ( ctrl shift v to paste into a terminal ):
```ruby
a = Assignment.first
Student.all.each do |student|
    student.create_group_for_working_alone_student(a.id)
end
```

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="31"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>