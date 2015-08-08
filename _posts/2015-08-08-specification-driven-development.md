---
layout: post
title: Specification Driven Development
excerpt: "Specification Driven Development (SDD) is an extension to Test Driven Development (TDD)."
tags: [nibralab, development cycle, test driven design, specifications, requirements, testdox]
comments: true
share: true
---

Specification Driven Development (SDD) is an extension to Test Driven Development (TDD).
 
Test Driven Development is a proven way to enhance the quality of software.
The concept says, that you write the test first, run it to see it fail, and then write the code until the test passes.
With this paradigm you get just the code you need, not more, not less.
You can evolve and refactor your code without the risk of breaking anything, because every feature is guarded by a test.

The problem with TDD, however, is to define which test to write.
Usually, you have a piece of code in mind, for which you write the test.
But having the code in your head and then writing the test is not really 'test first' - 
the code is already there, you just don't have it written down yet.

So, where is the right point to start?

## Derive Tests from the Specification

Of course, it can only be the requirements, ideally a specification. The workflow will be

  - for each section of the specification
      - create a test class
      - for each sentence of the section
          - identify the requirement(s) 
          - for each requirement
              - write a test 
              - run the test, see it fail
              - write the code, until the test passes

Refactor the (test) code, whenever you can improve maintainability and comprehension. 

Let's have a look at a concrete example.
We want to write a dependency injection container following the 
[upcoming PSR-11 standard](https://github.com/container-interop/fig-standards/blob/master/proposed/container.md),
which interface is identical to the Interop Container (despite the namespace).
Because the PSR interface is not yet awailable, we use the PSR-11 specification, replacing 'PSR' with 'Interop'.
Here is the specification:

- - -

### The Interop Container Specification

#### 1.1 Basics

The `Interop\Container\ContainerInterface` exposes two methods: `get` and `has`.

`get` takes one mandatory parameter: an entry identifier.
It MUST be a string.
A call to `get` can return anything (a mixed value),
or throws an exception if the identifier is not known to the container.
Two successive calls to `get` with the same identifier SHOULD return the same value.
However, depending on the implementor design and/or user configuration, different values might be returned,
so user SHOULD NOT rely on getting the same value on 2 successive calls.
While ContainerInterface only defines one mandatory parameter in `get()`, implementations MAY accept additional optional parameters.

`has` takes one unique parameter: an entry identifier.
It MUST return `true` if an entry identifier is known to the container and `false` if it is not.

#### 1.2 Exceptions

Exceptions directly thrown by the container MUST implement the `Interop\Container\Exception\ContainerExceptionInterface`.

A call to the `get` method with a non-existing id SHOULD throw a `Interop\Container\Exception\NotFoundExceptionInterface`.

#### 1.3 Additional feature: Delegate lookup

This section describes an additional feature that MAY be added to a container.
Containers are not required to implement the delegate lookup to respect the `ContainerInterface`.

The goal of the delegate lookup feature is to allow several containers to share entries.
Containers implementing this feature can perform dependency lookups in other containers.

Containers implementing this feature will offer a greater lever of interoperability with other containers.
Implementation of this feature is therefore RECOMMENDED.

A container implementing this feature:

  - MUST implement the `ContainerInterface`
  - MUST provide a way to register a delegate container (using a constructor parameter, or a setter, or any possible way).
    The delegate container MUST implement the `ContainerInterface`.

When a container is configured to use a delegate container for dependencies:

  - Calls to the `get` method should only return an entry if the entry is part of the container.
    If the entry is not part of the container, an exception should be thrown (as requested by the `ContainerInterface`).
  - Calls to the `has` method should only return `true` if the entry is part of the container.
    If the entry is not part of the container, `false` should be returned.
  - If the fetched entry has dependencies, instead of performing the dependency lookup in the container,
    the lookup is performed on the delegate container.

*Important!* By default, the lookup SHOULD be performed on the delegate container only, not on the container itself.

It is however allowed for containers to provide exception cases for special entries,
and a way to lookup into the same container (or another container) instead of the delegate container.

- - -

### For each Section of the Specification ...

As you can see, the specification is made up from three sections.
We start with the first section, *Basic*.

#### Create a Test Class

Since we might implement more than one specification in the same project,
it makes sense to organize our test structure accordingly.
All tests live in the `tests` directory,
and we decide to introduce another level for the specs, in this case `Interop`.
We create a corresponding test file `BasicTest.php` in the `tests/Interop` directory.

```
<?php

namespace MyContainer\Tests\Interop;

class BasicTest extends \PHPUnit_Framework_TestCase
{
}
```

#### For each Sentence of the Section

The first sentence of the section says

> The `Interop\Container\ContainerInterface` exposes two methods: `get` and `has`.

##### Identify the Requirement(s) 

Although this sentence describes a property of the `Interop\Container\ContainerInterface`, it implicitely contains
three requirements for our implementation:

  - The container implementation MUST implement the `Interop\Container\ContainerInterface`
  - The container MUST provide a `get` method
  - The container MUST provide a `has` method

##### For each Requirement

> The container implementation MUST implement the `Interop\Container\ContainerInterface`

This directly leeds to the first test.

###### Write a Test 

```
use MyContainer\Container;

class BasicTest extends \PHPUnit_Framework_TestCase
{
    public function testImplementsInterface()
    {
        $container = new Container();
        
        $this->assertInstanceOf('Interop\\Container\\ContainerInterface', $container);
    }
}
```

###### Run the Test, See it Fail

If you run PHPUnit, you'll get this result:

```
$ phpunit
...
PHP Fatal error:  Class 'MyContainer\Container' not found in .../Tests/Interop/BasicTest.php on line 11
 ```

Well, we knew that would happen.

###### Write the Code, until the Test Passes

The next step is to implement the feature:

```
<?php

namespace MyContainer;

class Container implements \Interop\Container\ContainerInterface
{
}
```

> Of course the test class needs access to the source, either through an `include` or an autoloader.

Test:

```
$ phpunit
...
PHP Fatal error:  Class MyContainer\Container contains 2 abstract methods and must therefore be declared abstract or implement the remaining methods (Interop\Container\ContainerInterface::get, Interop\Container\ContainerInterface::has) in ...
 ```

Your IDE may have told you that already. In many IDEs, the remedy is just a keystroke away.
After adding the method stubs, the code has become this:

```
<?php
namespace MyContainer;

use Interop\Container\Exception\ContainerException;
use Interop\Container\Exception\NotFoundException;

class Container implements \Interop\Container\ContainerInterface
{
    /**
     * Finds an entry of the container by its identifier and returns it.
     *
     * @param string $id Identifier of the entry to look for.
     *
     * @throws NotFoundException  No entry was found for this identifier.
     * @throws ContainerException Error while retrieving the entry.
     *
     * @return mixed Entry.
     */
    public function get($id)
    {
        // TODO: Implement get() method.
    }

    /**
     * Returns true if the container can return an entry for the given identifier.
     * Returns false otherwise.
     *
     * @param string $id Identifier of the entry to look for.
     *
     * @return boolean
     */
    public function has($id)
    {
        // TODO: Implement has() method.
    }
}
```
Test:

```
$ phpunit
...
OK (1 tests, 1 assertions)
```

##### Refactor the (Test) Code

While the test result tells us, how many tests and assertions were successfully executed,
it does not say, what exactly was tested. There is no visible relation to the specification.

PHPUnit offers the possibility to get a report in `testdox` format.
It looks like a list of features with checkboxes.
To make really use of that feature, naming of the tests is important.
With the tests as they are defined now, you'll get this result:

```
$ phpunit --testdox
...
MyContainer\Tests\Interop\Basic
 [x] Implements interface
 ```
The feature text is derived from the name of the test method.
Even if it looks sufficient at first sight, it is not:
We'd rather have a feature description close to the requirement wording.
One way would be to rename the test methods, but that does not always allow us to phrase things like we want,
since no special characters are allowed, and all uppercase letters are converted to space-lowercase.
The other option is to use the `@testdox` annotation.

```
    /**
     * @testdox The container implements the `Interop\Container\ContainerInterface`
     */
    public function testImplementsInterface()
    {
        $container = new Container();
        
        $this->assertInstanceOf('Interop\\Container\\ContainerInterface', $container);
    }
```

Now, the test result looks better:

```
$ phpunit --testdox
...
MyContainer\Tests\Interop\Basic
 [x] The container implements the `Interop\Container\ContainerInterface`
 
###### Write a Test 

Satisfying the first test forced us to implement the two methods, that make up the next requirements.
That explains and justifies, why they were part of just one requirement - they can't be separated.
So we hurry to add the remaining two tests to get even again.

```
    /**
     * @testdox The container provides a `get` method
     */
    public function testMethodGetExists()
    {
        $container = new Container();
        
        $this->assertTrue(method_exists($container, 'get'));
    }

    /**
     * @testdox The container provides a `has` method
     */
    public function testMethodHasExists()
    {
        $container = new Container();
        
        $this->assertTrue(method_exists($container, 'has'));
    }
```

###### Run the Test, See it Fail

```
$ phpunit --testdox
...
MyContainer\Tests\Interop\Basic
 [x] The container implements the `Interop\Container\ContainerInterface`
 [x] The container provides a `get` method
 [x] The container provides a `has` method
 ```

The tests do not fail, which we expected.
In this case, it is not a problem to deviate from the pure 'test-first' paradigm.

In order to be closer to the original text of the specification,  

> The `Interop\Container\ContainerInterface` exposes two methods: `get` and `has`.

you can put the *same* text to all three `@testdox` annotations:

```
class BasicTest extends \PHPUnit_Framework_TestCase
{
    /**
     * @testdox The Container implements the `Interop\Container\ContainerInterface` and exposes the methods: `get` and `has`
     */
    public function testImplementsInterface()
    {
        $container = new Container();
        
        $this->assertInstanceOf('Interop\\Container\\ContainerInterface', $container);
    }

    /**
     * @testdox The Container implements the `Interop\Container\ContainerInterface` and exposes the methods: `get` and `has`
     */
    public function testMethodGetExists()
    {
        $container = new Container();
        
        $this->assertTrue(method_exists($container, 'get'));
    }

    /**
     * @testdox The Container implements the `Interop\Container\ContainerInterface` and exposes the methods: `get` and `has`
     */
    public function testMethodHasExists()
    {
        $container = new Container();
        
        $this->assertTrue(method_exists($container, 'has'));
    }
}
```

The resulting report is much shorter and more comprehensive:

```
$ phpunit --testdox
...
MyContainer\Tests\Interop\Basic
 [x] The Container implements the `Interop\Container\ContainerInterface` and exposes the methods: `get` and `has`
```

> The single entry composes the results of all three tests. The checkbox is checked only, if *all* three tests pass.
> To identify the problem if a test fails, just run PHPUnit without the `--testdox` option,
> and you'll get a detailed description including a backtrace dump.

The real value gets obvious, if you see the specification and the test result together:

Spec: The `Interop\Container\ContainerInterface` exposes two methods: `get` and `has`.
Test: [x] The Container implements the `Interop\Container\ContainerInterface` and exposes the methods: `get` and `has` 

There's no doubt about that the specification is met.

#### Continue with the next sentence ...

The rest is left to you, the reader, as an exercise.
You should end up with a result similar to this:

```
MyContainer\Tests\Interop\Basic
 [x] The Container implements the `Interop\Container\ContainerInterface` and exposes the methods: `get` and `has`
 [x] `get` takes one mandatory parameter: an entry identifier. It MUST be a string
 [x] A call to `get` can return anything (a mixed value)
 [x] `get` throws an exception if the identifier is not known to the container
 [x] Two successive calls to `get` with the same identifier return the same value
 [x] `has` takes one unique parameter: an entry identifier
 [x] `has` returns `true` if an entry identifier is known to the container and `false` if it is not

MyContainer\Tests\Interop\Exceptions
 [x] Exceptions directly thrown by the container implement the `Interop\Container\Exception\ContainerExceptionInterface`
 [x] A call to the `get` method with a non-existing id throws an `Interop\Container\Exception\NotFoundExceptionInterface`

MyContainer\Tests\Interop\DelegateLookup
 [x] The Container implements the `Interop\Container\ContainerInterface`
 [x] The Container provides the `registerDelegate` method to register a delegate container
 [x] The Container accepts delegate containers implementing the `Interop\Container\ContainerInterface`
 [x] Calls to the `get` method only return an entry if the entry is part of the container
 [x] If the entry is not part of the container, `get` throws an `Interop\Container\Exception\NotFoundExceptionInterface`
 [x] Calls to the `has` method only return `true` if the entry is part of the container
 [x] If the entry is not part of the container, `has` returns `false`
 [x] If the fetched entry has dependencies, the lookup is performed on the delegate container
```

## Conclusion

Starting development from the specification via tests to production code provides you with the straightest way to a
good result.

 - You'll never again miss a iota from the specification, because any sentence is investigated for requirements.
 - You can at any time prove that your code respects the specification, because you have a test for every requirement.
 - You don't have untested code, because you only wrote code to satisfy your tests.

