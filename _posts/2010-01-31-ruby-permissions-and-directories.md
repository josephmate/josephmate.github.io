---
layout: post
title: Ruby, Permissions and Directories
date: 2010-01-31 00:27
author: matejoseph
comments: true
categories: [Uncategorized]
---
I recently learned that if you remove the execute bit on a directory and try to rm -r the parent directory then you will be unable to remove the executable directory. However, this works in a terminal.

Example in ruby:

```ruby
#!/usr/bin/env ruby
require 'fileutils'
include FileUtils

@parent_dir = '/tmp/parent_#{rand(1073741824)}'
@test_dir = '#{@parent_dir}/test'
@file = '#{@test_dir}/file'

FileUtils.mkdir( @parent_dir )
FileUtils.mkdir( @test_dir )
FileUtils.touch([ @file ])
FileUtils.chmod( 0600, [ @test_dir ])
FileUtils.rm_r( @parent_dir )
```

Output from script:

```bash
jmate@CalculatorJozef:~/everything$ ./test.rb
/usr/lib/ruby/1.8/fileutils.rb:1297:in `unlink': Permission denied - /tmp/parent_674314876/test/file (Errno::EACCES)
 from /usr/lib/ruby/1.8/fileutils.rb:1297:in `remove_file'
 from /usr/lib/ruby/1.8/fileutils.rb:1302:in `platform_support'
 from /usr/lib/ruby/1.8/fileutils.rb:1296:in `remove_file'
 from /usr/lib/ruby/1.8/fileutils.rb:1285:in `remove'
 from /usr/lib/ruby/1.8/fileutils.rb:756:in `remove_entry'
 from /usr/lib/ruby/1.8/fileutils.rb:1335:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:1335:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:1339:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:1334:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:1333:in `each'
 from /usr/lib/ruby/1.8/fileutils.rb:1333:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:1334:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:1333:in `each'
 from /usr/lib/ruby/1.8/fileutils.rb:1333:in `postorder_traverse'
 from /usr/lib/ruby/1.8/fileutils.rb:754:in `remove_entry'
 from /usr/lib/ruby/1.8/fileutils.rb:612:in `rm_r'
 from /usr/lib/ruby/1.8/fileutils.rb:608:in `each'
 from /usr/lib/ruby/1.8/fileutils.rb:608:in `rm_r'
 from ./test.rb:14
```

Example in terminal:

```bash
jmate@CalculatorJozef:/tmp$ mkdir parent
jmate@CalculatorJozef:/tmp$ mkdir parent/test
jmate@CalculatorJozef:/tmp$ touch parent/test/file
jmate@CalculatorJozef:/tmp$ rm -r parent
```

As you can see from the output above. I had no trouble doing this in the terminal.

It might have something to do with not being able to cd into a directory that does not have the execute bit set.

Cheers,

Joseph

