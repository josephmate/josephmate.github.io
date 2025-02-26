---
layout: post
title: Nested Contexts
date: 2010-01-31 02:08
author: matejoseph
comments: true
categories: [Uncategorized]
---
I learned a new trick with shoulda. Credit goes to: http://www.viget.com/extend/reusing-contexts-in-shoulda-with-context-macros/

You can nest contexts inorder to resuse them.

Here is a quick example I prepared:

```ruby
context 'we have a temp dir' do
    setup
    do
        @temp_dir = '/tmp/temp_#{rand(1073741824)}'
        @log_dir = '#{@temp_dir}/log'
        @file = '#{@log_dir}/file'
        FileUtils.mkdir( @temp_dir )
    end
    teardown
    do
        FileUtils.rm_r( @temp_dir )
    end
    should 'be able to test with the temp dir'
    do
    end

    context 'we have a log dir'
    do
        setup
        do
            FileUtils.mkdir( @log_dir )
        end
        should 'be able to test with the temp and log dir'
        do
        end

        context 'we have a file inside the log dir inside the temp dir'
        do
            setup
            do
                FileUtils.touch( [@file] )
            end
            should 'be able to test with temp, log and file'
            do
            end
        end
    end
end
```



Cheers,
Joseph

