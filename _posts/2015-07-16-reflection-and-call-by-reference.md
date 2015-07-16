---
layout: post
title: Reflection and Call-by-Reference
excerpt: "How to work-around problems using ReflectionMethod::invoke() with call-by-reference parameters."
tags: [nibralab, reflection, reflectionmethod, invoke, call-by-reference, php, testing]
comments: true
share: true
---

When refactoring the unit tests for the Joomla CMS router, I stumbled upon several cases, where the Subject under Test
was mocked or decorated (wrapped) in order to access protected methods. While it might be ok to test non-public members on legacy code
(deriving classes may depend on it - you get this mess, if you don't do TDD), it is *never* acceptable to run tests
on a mock or a wrapper. *Those mocks or wrappers are not your Subject under Test!*

## ReflectionMethod::invoke()

So I started to refactor these mocks and decorators using the Reflection API, which allows to make private and protected
members accessible. So I did something like this (simplified) in my test:

{% highlight php %}
public function testCallOfNonPublicMethod()
{
    $object = new SomeClass();
    $param  = $this->getMock('ParamClass');
    
    $method = new ReflectionMethod('SomeClass', 'someMethod');
    $method->setAccessible(true);
    
    $this->assertEquals($expected, $method->invoke($object, $param);
}
{% endhighlight %}

The advantage is, that the code tells you exactly what happens to the Subject under Test.
The method is set to be visible and then invoked on the object. But at one point I ran into an unexpected warning:

{% highlight bash %}
Warning: Parameter 1 to SomeClass::someMethod() expected to be a reference, value given in SomeTest.php on line 42
{% endhighlight %}

That was weird, especially, as the parameter was an object, which *always* is passed by reference.

Some debugging later, I found PHP4 relicts, in this case, a method like

{% highlight php %}
class SomeClass {
    protected function someMethod(ParamClass &$param) {
        ...
    }
}
{% endhighlight %}

The problem is, that PHPUnit internally at one point clones the arguments (or otherwise turn them into values;
I didn't investigate that further), so the reference gets lost. If the method is called directly, the reference gets
transported, but that's not possible, because method is protected.

On PHP4, the call could have been changed to
 
{% highlight php %}
$this->assertEquals($expected, $method->invoke($object, &$param);
{% endhighlight %}
 
to solve the problem. This Call-by-Reference has though been deprecated in PHP5, and removed in PHP 5.4:

{% highlight bash %}
Fatal error: Call-time pass-by-reference has been removed in SomeTest.php on line 42
{% endhighlight %}

So that is no option as well.

Ok, now I understood, why the Subject under Test was wrapped.
But still, it is *never* acceptable to run tests on a mock or a wrapper. So how can this be solved?
 
## ReflectionMethod::invokeArgs()

The trick is to get the reference one level down, so PHPUnit's cloning (or whatever) does not destroy the reference any more.
Just like the `call_user_func()` and `call_user_func_array()` methods, ReflectionMethod provides a way to pass
the method arguments using a single array. For methods with call-by-reference parameters, the test code now looks
like this:

{% highlight php %}
public function testCallOfNonPublicMethod()
{
    $object = new SomeClass();
    $param  = $this->getMock('ParamClass');
    
    $method = new ReflectionMethod('SomeClass', 'someMethod');
    $method->setAccessible(true);
    
    $this->assertEquals($expected, $method->invokeArgs($object, array(&$param));
}
{% endhighlight %}

As you can see, the reference operator can be explicitly specified without any violations. 

## GreenCape/reflector

I wrote a small class, that makes using Reflection like this a bit easier. The above code is reduced to

{% highlight php %}
use GreenCape\Reflection\Reflector;
...
public function testCallOfNonPublicMethod()
{
    $object = new SomeClass();
    $param  = $this->getMock('ParamClass');

    $this->assertEquals($expected, Reflector::invokeArray($object, 'someMethod', array(&$param));
}
{% endhighlight %}

You can get Reflector from the GreenCape repository at [GitHub](https://github.com/GreenCape/reflector)
or at [Packagist](https://packagist.org/packages/greencape/reflector).
It does a lot more than described here, e.g. access to non-public members of the complete inheritance chain of an object.

Feedback is of course always welcome!
