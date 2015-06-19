# elazar/phantestic

A small PHP testing framework that aims to be simple, fast, modern, and flexible.

Currently in a very alpha state. Feel free to mess around with it, but expect things to break.

## Installation

Use [Composer](https://getcomposer.org).

```json
{
    "require-dev": {
        "elazar/phantestic": "^0.1"
    }
}
```

## Components

A **test loader** loads the test to be run. It can be anything that implements [`\Traversable`](http://php.net/manual/en/class.traversable.php) (i.e. an instance of a class that implements [`\IteratorAggregate`](http://php.net/manual/en/class.iteratoraggregate.php) or [`\Iterator`](http://php.net/manual/en/class.iterator.php) , such as [`\Generator`](http://php.net/manual/en/class.generator.php)) to allow loaded tests to be iterable.

The **test runner** uses the test loader to load tests, run them, and in the process emit multiple events that **test handlers** can intercept and act upon.

As an example, [`LocalTestRunner`](https://github.com/elazar/phantestic/blob/master/src/TestRunner/LocalTestRunner.php) runs tests in the same PHP process as the test runner. Its constructor accepts two arguments: the test loader to use and an array of test handler objects that implement [`TestHandlerInterface`](https://github.com/elazar/phantestic/blob/master/src/TestHandler/TestHandlerInterface.php).

When its `run()` method is called, [`LocalTestRunner`](https://github.com/elazar/phantestic/blob/master/src/TestRunner/LocalTestRunner.php) handles injecting an [event emitter](https://github.com/igorw/evenement/blob/master/src/Evenement/EventEmitterInterface.php) into the test handler objects, which enables those objects to register callbacks with the emitter for any events that may be relevant to them.

An example of a test handler is [`CliOutputTestHandler`](https://github.com/elazar/phantestic/blob/master/src/TestHandler/CliOutputTestHandler.php), which outputs the results of executing tests to `stdout` as they are received and a failure summary once all tests have been run.

## Configuring a Runner

There is no stock test runner; one must be configured based on the needs of your project.

Here's a sample runner configuration.

```php
$classmap_path = '../vendor/composer/autoload_classmap.php';
$loader = new \Phantestic\TestLoader\ClassmapFileObjectTestLoader($classmap_path);
$handlers = [ new \Phantestic\TestHandler\CliOutputTestHandler ];
$runner = new \Phantestic\TestRunner\LocalTestRunner($loader, $handlers);
$runner->run();
```

[`ClassmapFileObjectTestLoader`](https://github.com/elazar/phantestic/blob/master/src/TestLoader/ClassmapFileObjectTestLoader.php) locates tests based on the contents of a classmap file, such as the one generated by by `-o` flag of several [Composer](https://getcomposer.org) commands. By default, it looks for class files with names ending in `Test.php`, instantiates the classes, and invokes methods with names prefixed with `test`. The regular expressions used to filter file and method names can be changed using the constructor parameters of [`ClassmapFileObjectTestLoader`](https://github.com/elazar/phantestic/blob/master/src/TestLoader/ClassmapFileObjectTestLoader.php).

## Writing Tests

Theoretically, tests can be anything [callable](http://php.net/manual/en/language.types.callable.php). The test loader may restrict this to specific types of callables (e.g. [`ClassmapFileObjectTestLoader`](https://github.com/elazar/phantestic/blob/master/src/TestLoader/ClassmapFileObjectTestLoader.php) only supports instance methods). The test loader wraps test callbacks in an instance of a class implementing [`TestCaseInterface`](https://github.com/elazar/phantestic/blob/master/src/TestCase/TestCaseInterface.php), such as the default [`TestCase`](https://github.com/elazar/phantestic/blob/master/src/TestCase/TestCase.php) implementation.

Failures can be indicated by throwing an [exception](http://php.net/manual/en/language.exceptions.php). Other statuses can be indicating by throwing an instance of a subclass of [`TestResult`](https://github.com/elazar/phantestic/blob/master/src/TestResult/TestResult.php). [`TestCase`](https://github.com/elazar/phantestic/blob/master/src/TestCase/TestCase.php) [converts errors to exceptions](http://php.net/manual/en/class.errorexception.php#errorexception.examples) and considers any uncaught exception to indicate failure. Likewise, no exception being thrown indicates success.

```php
// src/Adder.php
class Adder
{
    public function add($x, $y)
    {
        return $x + $y;
    }
}

// tests/AdderTest.php
class AdderTest
{
    public function testAdd()
    {
        $adder = new Adder;
        $result = $adder->add(2, 2);
        if ($result != 4) {
            throw new \RangeException('2 + 2 should equal 4');
        }
    }
}
```

## Writing Test Handlers

Test handlers implement [`TestHandlerInterface`](https://github.com/elazar/phantestic/blob/master/src/TestHandler/TestHandlerInterface.php), which has a single method: `setEventEmitter()`. This method receives an instance of [`EventEmitterInterface`](https://github.com/igorw/evenement/blob/master/src/Evenement/EventEmitterInterface.php) as its only argument. Within its implementation of `setEventEmitter()`, a test handler can use this argument to register event callbacks. An example of this is below, taken from [`CliOutputTestHandler`](https://github.com/elazar/phantestic/blob/master/src/TestHandler/CliOutputTestHandler.php), which registers its own methods as callbacks for several events.

```php
public function setEventEmitter(EventEmitterInterface $emitter)
{
    $emitter->on('phantestic.test.failresult', [$this, 'handleFail']);
    $emitter->on('phantestic.test.passresult', [$this, 'handlePass']);
    $emitter->on('phantestic.tests.after', [$this, 'printSummary']);
}
```

Supported events may vary depending on the test loader and runner in use.

### [`LocalTestRunner`](https://github.com/elazar/phantestic/blob/master/src/TestRunner/LocalTestRunner.php)

* `phantestic.tests.before`: Before any tests are run, with the test runner as an argument
* `phantestic.tests.after`: After all tests are run, with the test runner as an argument
* `phantestic.test.before`: Before each test, with the test case and runner as arguments
* `phantestic.test.after`: After each test, with the test case and runner as arguments
* `phantestic.test.failresult`: When a test fails, with the test case and runner as arguments
* `phantestic.test.passresult`: When a test passes, with the test case and runner as arguments
* `phantestic.test.RESULT`: When a test has a result other than passing or failing where `RESULT` is the short name of the class extending [`TestResult`](https://github.com/elazar/phantestic/blob/master/src/TestResult/TestResult.php), with the test case and runner as arguments

### [`ClassmapFileObjectTestLoader`](https://github.com/elazar/phantestic/blob/master/src/TestLoader/ClassmapFileObjectTestLoader.php)

* `phantestic.loader.loaded`: When a test case is loaded, with the test case and associated test class instance and name and test method name as arguments
