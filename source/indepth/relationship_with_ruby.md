---
title: Passenger and its relationship with Ruby
section: misc
sidebar: sidebar
---
# Passenger and its relationship with Ruby

Passenger has a lightweight dependency on Ruby. This page explains what we mean by "lightweight" and how exactly Ruby is used and. For non-Ruby users, we explain how we ensure this Ruby dependency doesn't impact resource usage. For Ruby users, we explain what it means if you try to install Passenger on a system with multiple Ruby interpreters.

## How Ruby is used

Passenger's core is written in C++ for performance and memory efficiency. It supports web applications written in any language. Passenger requires Ruby, its usage of Ruby is minimal in order to maximize performance and to minimize memory usage.

 * Passenger's installer, build system and command line administration tools are written in Ruby.
 * Certain internally used tools, such as the crash handler (which generates a backtrace in case Passenger crash) and the prespawn script <!-- TODO: fix link --> are written in Ruby as well. These tools are only invoked in a one-off manner, and do not run persistently in the background.
 * Ruby web application support is implemented in Ruby.
 * If you use [Flying Passenger](https://www.phusionpassenger.com/documentation/Users%20guide%20Nginx.html#flying_passenger), then the Flying Passenger daemon is written in Ruby. The daemon is a small (less than 500 lines of code) and offloads most tasks to the C++ core.
 * If you use [Passenger Standalone](https://www.phusionpassenger.com/documentation/Users%20guide%20Standalone.html), then the frontend (the `passenger` command) is written in Ruby. The frontend is small (less than 1500 lines of code) and offloads most tasks to the C++ core.

Other than the aforementioned aspects, Passenger does not use Ruby during normal operation. For example, if you run Node.js web applications on Passenger, then there will be (almost) no Ruby code running on the system.

## When the system has multiple Ruby interpreters

Passenger may be installed with any Ruby interpreter. Once installed, you can run Passenger's Ruby parts under any Ruby interpreter you want, even if that Ruby interpreter was not the one you originally installed Passenger with.

The reason for this is that Passenger does not dynamically link to Ruby: Passenger uses Ruby entirely out-of-process. Thus you can switch to any Ruby interpreter you want during runtime, without recompiling Passenger, and without worrying about what Ruby you used to install Passenger.

Passenger is also capable of running Ruby web applications under any Ruby interpreter you want. So it is not important which Ruby you use to install Passenger: it will work regardless. Please refer to the documentation for the `passenger_ruby` <!-- TODO: fix link --> directive to learn how run different web applications under different Ruby interpreters.

### Caveat: RVM and RVM gemsets

There is however one caveat if you happen to be using RVM or RVM gemsets. When you `gem install` Passenger using RVM, then RVM will install Passenger into the *currently active RVM Ruby and gemset*. This means that Passenger commands - such as `passenger`, `passenger-install-xxx-module` and `passenger-status` - are available in that same RVM Ruby and gemset only. When you switch Ruby interpreter, or when you switch gemset, the Passenger commands will no longer be available, and you will get a `command not found` error. Here's an example which demonstrates the problem.

<pre class="highlight"><span class="c">## Install Phusion Passenger (open source edition) using Ruby 1.9.3
## and the 'business' gemset</span>
$ rvm use 1.9.3
<span class="output">Using /home/phusion/.rvm/gems/ruby-1.9.3-p429</span>
$ rvm gemset create business
$ rvm gemset use business
<span class="output">Using ruby-1.9.3-p429 with gemset business</span>
$ curl -O https://s3.amazonaws.com/phusion-passenger/releases/gem_bootstrap.sh
$ eval "`sh gem_bootstrap.sh`"
$ gem install passenger

<span class="c">## Verify that passenger works</span>
$ passenger --version
<span class="output">Phusion Passenger version 4.0.14</span>

<span class="c">## Switch to a different RVM gemset. You will get a `command not found`</span>
$ rvm gemset use default
<span class="output">Using ruby-1.9.3-p429 with gemset default</span>
$ passenger --version
<span class="output">bash: passenger: command not found</span>

<span class="c">## Switch to a different Ruby interpreter. You will also get
## a `command not found`</span>
$ rvm use 2.0.0
<span class="output">Using /home/phusion/.rvm/gems/ruby-2.0.0-p195</span>
$ passenger --version
<span class="output">bash: passenger: command not found</span>

<span class="c">## Switch back to the Ruby and gemset that you installed
## Passenger with, and verify that it works again</span>
$ rvm use 1.9.3
<span class="output">Using /home/phusion/.rvm/gems/ruby-2.0.0-p195</span>
$ rvm gemset use business
<span class="output">Using ruby-1.9.3-p429 with gemset business</span>
$ passenger --version
<span class="output">Phusion Passenger version 4.0.14</span></pre>

There are several ways to solve this problem:

 1. Permanently add Phusion Passenger's command directory to your PATH, so that your shell can always find them even when you switch RVM Ruby or gemset. If you don't know what PATH means, please read [About environment variables](https://www.phusionpassenger.com/documentation/Users%20guide%20Nginx.html#about_environment_variables) first.

    The drawback is that you have to redo this every time you upgrade Phusion Passenger, because the Phusion Passenger directory filename is dependent on the version number.

    First, identify the location of the Phusion Passenger command directory, like this:

    <pre class="highlight"><span class="prompt">$</span> echo `passenger-config --root`/bin
<span class="output">/home/phusion/.rvm/gems/ruby-1.9.3-p429/gems/passenger-4.0.15/bin</span></pre>

    Next, add the directory that you've found to your current shell's PATH:

        export PATH=/home/phusion/.rvm/gems/ruby-1.9.3-p429/gems/passenger-4.0.15/bin:$PATH

    Finally, make the change permanent by appending the above command to your bash startup file:

        echo 'export PATH=/home/phusion/.rvm/gems/ruby-1.9.3-p429/gems/passenger-4.0.15/bin:$PATH' \
        >> ~/.bashrc

 2. Switch back to the RVM Ruby and gemset that you installed Phusion Passenger with, before running any Phusion Passenger command.
 3. Prepend any Phusion Passenger command with `rvm-exec RUBY_NAME@GEMSET_NAME ruby -S`. If the relevant Phusion Passenger command also needs root privileges, then prepend `rvmsudo` before that. For example:

        rvm-exec 1.9.3@business ruby -S passenger --version
        rvmsudo rvm-exec 1.9.3@business ruby -S passenger-install-apache2-module