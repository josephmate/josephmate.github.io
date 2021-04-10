---
layout: post
title: Mocking Modules with Mocha
date: 2010-01-31 02:57
author: matejoseph
comments: true
categories: [Uncategorized]
---
I could not find any examples on mocking modules with mocha. It's probably because it's so easy! It's just like mocking an instance of a class.

the_module.rb :

```ruby
module TheModule
    def the_module_function
        return 'the real value'
    end
end
```

test.rb :

```ruby
require 'test/unit'
require 'mocha'
require 'the_module'
include TheModule

class TheModuleTest &lt; Test::Unit::TestCase
    # replace 'the real value' with 'the mocked value''
    def test_it
        assert TheModule.the_module_function == 'the real value'

        TheModule.stubs(:the_module_function).returns('the mocked value')

        assert TheModule.the_module_function == 'the mocked value'
    end
end
```

Running the tests:

```bash
jmate@CalculatorJozef:~/everything/workspaces$ ruby test.rb
Loaded suite test
Started
.
Finished in 0.00096 seconds.

1 tests, 2 assertions, 0 failures, 0 errors
```


Cheers,
Joseph

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="36"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>