---
layout: post
title: Weird Behaviour Of Mocha
date: 2010-01-31 03:08
author: matejoseph
comments: true
categories: [Uncategorized]
---
In the markus code, there are a lot of places where we use `markus_config_<something>` instead of `MarkusConfigurator.markus_config_<something>`. We should start using the latter. He's why:

`ensure_config_helper.rb`

```ruby
def self.check_config()
  puts 'Checking #{markus_config_logging_logfile}'
  puts 'Checking #{markus_config_logging_errorlogfile}'
  puts 'Checking #{markus_config_repository_storage}'
  puts 'Checking #{markus_config_validate_file}'
  puts 'Checking #{MarkusConfigurator.markus_config_logging_logfile}'
  puts 'Checking #{MarkusConfigurator.markus_config_logging_errorlogfile}'
  puts 'Checking #{MarkusConfigurator.markus_config_repository_storage}'
  puts 'Checking #{MarkusConfigurator.markus_config_validate_file}'
  check_in_writable_dir(markus_config_logging_logfile, 'MARKUS_LOGGING_LOGFILE')
  check_in_writable_dir(markus_config_logging_errorlogfile, 'MARKUS_LOGGING_ERRORLOGFILE')
  check_writable(markus_config_repository_storage, 'REPOSITORY_STORAGE')
  check_readable(markus_config_repository_storage, 'REPOSITORY_STORAGE')
  if ! RUBY_PLATFORM =~ /(:?mswin|mingw)/ # should match for Windows only
    check_executable(markus_config_validate_file, 'VALIDATE_FILE')
  end
end
```

Output from running rake test:units When loading everything up:

```bash
Checking log/info_development.log
Checking log/error_development.log
Checking /home/jmate/everything/workspaces/repos
Checking /home/jmate/everything/workspaces/markus/config/dummy_validate.sh
Checking log/info_development.log
Checking log/error_development.log
Checking /home/jmate/everything/workspaces/repos
Checking /home/jmate/everything/workspaces/markus/config/dummy_validate.sh
```

At this point, `<blah> == MarkusConfigurator.<blah>`

When running the test cases:

```bash
/tmp/ensure_config_helper_test_777699315/log/log_info_file.log
Checking log/info_test.log
Checking log/error_test.log
Checking /home/jmate/everything/workspaces/repos
Checking /home/jmate/everything/workspaces/markus/config/dummy_validate.sh
Checking /tmp/ensure_config_helper_test_777699315/log/log_info_file.log
Checking /tmp/ensure_config_helper_test_777699315/log/log_error_file.log
Checking /tmp/ensure_config_helper_test_777699315/source_repo_dir
Checking /tmp/ensure_config_helper_test_777699315/validate_script.sh
/tmp/ensure_config_helper_test_595310852/log/log_info_file.log
...
...
...
Checking log/info_test.log
Checking log/error_test.log
Checking /home/jmate/everything/workspaces/repos
Checking /home/jmate/everything/workspaces/markus/config/dummy_validate.sh
Checking /tmp/ensure_config_helper_test_473533902/log/log_info_file.log
Checking /tmp/ensure_config_helper_test_473533902/log/log_error_file.log
Checking /tmp/ensure_config_helper_test_473533902/source_repo_dir
Checking /tmp/ensure_config_helper_test_473533902/validate_script.sh
```

Now you can see that `<blah> != MarkusConfigurator.<blah>`. So the namespace for the method you are trying to call cannot be ambiguous. In the source code above you must use `MarkusConfigurator.<blah>`.

<script src="https://utteranc.es/client.js"
        repo="josephmate/josephmate.github.io"
        issue-number="33"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>