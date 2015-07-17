---
layout: post
title: Isolating PHPUnit
excerpt: "How to work-around the Notice: Undefined index: SCRIPT_NAME in /.../phpunit/src/Util/Filter.php on line 69."
tags: [nibralab, phpunit, notice, undefined index, php, composer, docker, testing]
comments: true
share: true
---

Shortly, I wanted to make a contribution to the Joomla CMS. Of course I did unit testing, with a customized `phpunit.xml`,
just running the tests, that were relevant for the area, I was working on, with code coverage turned off. That is
necessary, because, when doing Test Driven Development, you don't want to wait several minutes for the whole test suite
to finish. Also, coverage isn't of any interest at this stage.

As expected, everything went fine.

When my contribution was ready to be committed to my Joomla fork in order to create a pull request, I started the
complete test suite - and got hundreds of notices:

{% highlight bash %}
Notice: Undefined index: SCRIPT_NAME in /.../libraries/vendor/phpunit/phpunit/src/Util/Filter.php on line 69
{% endhighlight %}

### The Problem

Well, we know, that some Joomla tests have some side effects, which they should not have. Because of that, it is
possible, that tests work in isolation, but not together with others. But, these are known (and being worked on),
and the side effects do not produce that particular notice.

Searching the web for this notice reveals, that the issue has been known for years, without a viable solution.

### The Cause

In the past, the test suite worked fine on my machine. I've lately worked on a couple of projects, with each having
its own requirements, also regarding version of PHPUnit. Joomla for example requires version 4.1.x, because some
tests use a feature, that was removed in 4.2. Globally, I currently have version 4.7.7 installed. But if I use the
global PHPUnit instead of that in Joomla's vendor directory, *it identifies itself as version 4.1.6*, which definitly
should not be the case. My conclusion is, that the many PHPUnit versions, and all the Composer autoloaders, somehow got
messed up.

### The Solution

"It used to work on my machine" gives a hint into the right direction. It has to be ensured, that the environment is not
polluted with stuff interferring with what we want to do. And that's exactly, what Docker is made for.

You need a Docker image, that provides PHP at the commandline. Its configuration should be easy to change - often
the `memory_limit` must be set up, for example. I have such an image in my repo, so I used that.

Just start an interactive bash session in the container and prepare the environment.
For the Joomla CMS it looks like this::

{% highlight bash %}
$ cd path/to/joomla-cms
$ docker run -it --rm -v "$PWD:/sut" greencape/nginx-php /bin/bash
[root@ddfa1bae4f0b /] cd /sut
[root@ddfa1bae4f0b /sut] echo 'memory_limit=-1' > /etc/php.d/memory.ini
{% endhighlight %}

Keep the console window open. Now you can run the tests whenever you want, by typing

{% highlight bash %}
[root@ddfa1bae4f0b /sut] ./libraries/vendor/bin/phpunit
{% endhighlight %}

For other projects than Joomla, the paths will of course be different. The container configuration will also be 
different for other images. But I'm sure, you get the idea, how to isolate and thus stabilize your testing.
