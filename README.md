AMP: Async Multi-threading in PHP (5.4+)
----------------------------------------

Amp parallelizes synchronous PHP function calls to worker thread pools in non-blocking applications.
The library dispatches blocking calls to worker threads where they can execute in parallel and
returns results asynchronously upon completion.

**Problem Domain**

PHP has a vast catalog of synchronous libraries and extensions. However, it's generally difficult to
find libs for use inside non-blocking event loops. Beyond this limitation there are common tasks
(like filesystem IO) which don't play nice with the non-blocking paradigm. Amp exposes threaded
concurrency in a non-blocking way to execute tasks in worker threads.

> **NOTE:** Amp isn't really intended for use in a PHP web SAPI environment. It doesn't make sense
to fire up a new thread pool that internally uses sockets for inter-thread communication on each
request in a web environment. It's designed for use in CLI environments.

### Project Goals

* Expose threaded multiprocessing inside event-driven non-blocking applications;
* Build all components using [SOLID][solid], readable and unit-tested code.


### Requirements

* [PHP 5.4+][php-net] You'll need PHP.
* [ext/pthreads][pthreads] The pthreads extension ([windows .DLLs here][win-pthreads-dlls])
* [rdlowrey/alert][alert] Alert IO/events
* [rdlowrey/after][after] Simple concurrency primitives for use with Alert reactors

[php-net]: http://php.net "php.net"
[pthreads]: http://pecl.php.net/package/pthreads "pthreads"
[solid]: http://en.wikipedia.org/wiki/SOLID_(object-oriented_design) "S.O.L.I.D."
[alert]: https://github.com/rdlowrey/Alert "Alert IO/events"
[after]: https://github.com/rdlowrey/After "After concurrency primitives"
[win-pthreads-dlls]: http://windows.php.net/downloads/pecl/releases/pthreads/ "pthreads Windows DLLs"

### Installation

```bash
$ git clone git@github.com:rdlowrey/Amp.git
$ cd Amp
$ composer install
```



## The Guide

**Intro**

* [Event Loop Basics](#event-loop-basics)

**Basic Usage**

* [Basic Calls](#basic-calls)
* [Userland Functions](#userland-functions)
* [Static Methods](#static-methods)
* [Magic Calls](#magic-calls)
* [Error Handling](#error-handling)
* [Task Timeouts](#task-timeouts)
* [Pool Size](#pool-size)
* [Thread Execution Limits](#thread-execution-limits)
* [Pthreads Context Flags](#pthreads-context-flags)

**Advanced Usage**

* [Stackable Tasks](#stackable-tasks)
* [Magic Tasks](#magic-tasks)
* [Class Autoloading](#class-autoloading)
* [Naive Wait Parallelization](#naive-wait-parallelization)
* [Parallelization Combinators](#parallelization-combinators)



### Intro

#### Event Loop Basics

> **NOTE:** Because Amp uses `After` concurrency primitives it's possible to execute tasks in
> parallel with zero knowledge of event loops. You can find out more in the
> [Naive Wait Parallelization](#naive-wait-parallelization) and
> [Parallelization Combinators](#parallelization-combinators) sections.

Executing code inside an event loop allows us to use non-blocking libraries to perform multiple IO
operations at the same time. Instead of waiting for each individual operation to complete the event
loop assumes program flow and informs us when our tasks finish or actionable events occur. This
paradigm allows programs to execute other instructions when they would otherwise waste cycle waiting
for slow IO operations to complete. The non-blocking approach is particularly useful for scalable
network applications and servers where the naive thread-per-connection approach is untenable.

Unfortunately, robust applications generally require synchronous functionality and/or filesystem
operations that can't behave in a non-blocking manner. Amp was created to provide non-blocking
applications access to the full range of synchronous PHP functionality without blocking the main
event loop.

> **NOTE:** It's critical that any non-blocking libs in your application use the *same* event loop
> scheduler. The Amp dispatcher uses the [`Alert`][alert] event reactor for scheduling.

Because Amp executes inside an event loop, you'll see all of the following examples create a new
event reactor instance to kick things off. Once the event reactor is started it assumes program
control and *will not* return control until your application calls `Reactor::stop()`.

> **Learn more about the [Alert event reactor](https://github.com/rdlowrey/Alert).**



### Basic Usage

#### Basic Calls

The simplest way to use Amp is to dispatch calls to global functions:

```php
<?php
// Everything happens inside an event reactor loop
(new Alert\NativeReactor)->run(function($reactor) {

    // Create our pthreads task dispatcher
    $dispatcher = new Amp\Dispatcher($reactor);

    // Invoke strlen('zanzibar') in a worker thread and
    // notify our callback when the result comes back

    $promise = $dispatcher->call('strlen', 'zanzibar!');
    $promise->when(function($error, $result) use ($reactor) {

        // Output the results of the function call executed in
        // a worker thread.
        printf("Woot! strlen('zanzibar') === %d", $result);

        // Stop the event loop so we don't sit around forever
        $reactor->stop();
    });

});
```

The above example outputs the following to our console:

```
Woot! strlen('zanzibar') === 8
```

Obviously, the `strlen` call here is a spurious use of threaded concurrency; remember that it only
ever makes sense to dispatch work to a thread if the processing overhead of sending the call and
receiving the result is outweighed by the time that would otherwise be spent waiting for the result.
A more useful example demonstrates retrieving the contents of a filesystem resource:

```php
<?php
(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher->call('file_get_contents', '/path/to/file');
    $promise->when(function($error, $result) use ($reactor) {
        if ($error) {
            printf("Something went wrong: %s", $error->getMessage());
        } else {
            var_dump($result);
        }
        $reactor->stop();
    });
});
```

The above code retrieves the contents of the file at `/path/to/file` in a worker thread and dumps
the result in our main thread upon completion.


#### Userland Functions

We aren't limited to native functions. The `Amp\Dispatcher` can also dispatch calls to userland
functions ...

```php
<?php
function myMultiply($x, $y) {
    return $x * $y;
}

(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher->call('myMultiply', 6, 7);
    $promise->when(function($error, $result) use ($reactor) {
        if ($error) {
            printf("Something went wrong: %s", $error->getMessage());
        } else {
            var_dump($result);
        }
        $reactor->stop();
    });
});
```

The above code results in the following output:

```
int(42)
```


#### Static Methods

The `Dispatcher::call()` method can accept any callable string, so we aren't limited to function
names. We can also dispatch calls to static class methods:

```php
<?php
class MyMultiplier {
    public static function multiply($x, $y) {
        return $x * $y;
    }
}

(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher->call('MyMultiplier::multiply', 6, 7);
    $promise->when(function($error, $result) use ($reactor) {
        if ($error) {
            printf("Something went wrong: %s", $error->getMessage());
        } else {
            var_dump($result);
        }
        $reactor->stop();
    });
});
```

The above code results in the following output:

```
int(42)
```

> **IMPORTANT:** In this example we've hardcoded the `MyMultiplier` class definition in the code.
> There is *no* class autoloading employed. There is no way for `ext/pthreads` to inherit globally
> registered autoloaders from the main thread. If you require autoloading in your worker threads
> you *MUST* dispatch a stackable task to define autoloader function(s) in your workers as
> demonstrated in the [Class Autoloading](#class-autoloading) section of this guide.


#### Magic Calls

Dispatchers take advantage of the magic `__call()` method to simplify calls to functions in the global
namespace. Consider:

```php
<?php
(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher->fopen('/path/to/file', 'r');
    $promise->when(function($error, $result) use ($reactor) {
        if ($error) {
            printf("Something went wrong: %s", $error->getMessage());
        } else {
            var_dump($result);
        }
        $reactor->stop();
    });
});
```

The above code opens a read-only file handle to the specified file and returns the result in
our main thread upon completion.


#### Error Handling

You may have noticed that our examples to this point have not returned results directly. Instead,
they return an instance of `After\Promise`. These monadic placeholder objects allow us to distinguish
between successful results and errors. The most important thing to remember is this:

> All `After\Promise` instances will notify `when()` callbacks using the "error-first" form when
> tasks resolve.

This behavior makes it difficult to accidentlly ignore execution failures. Consider ...

**Uncaught Exception**

```php
<?php

function myThrowingFunction() {
    throw new \RuntimeException('oh noes!!!');
}

(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher->myThrowingFunction();
    $promise->when(function($error, $result) use ($reactor) {
        var_dump($error);
        $reactor->stop();
    });
});
```

**Fatal Error**

In the following example we purposefully do something that will generate a fatal error in our
worker thread. Pthreads (and Amp) recover from this condition automatically. There is no need to
restart the thread pool and our main thread seamlessly recovers:

```php
<?php

function myFatalFunction() {
    $nonexistentObject->nonexistentMethod(); // fatal
}

(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher->myFatalFunction();
    $promise->when(function($error, $result) use ($reactor) {
        echo $error; // view the error traceback
        $reactor->stop();
    });
});
```

#### Task Timeouts

> **NOTE:** Relying on timeouts is almost always a poor design decision. You're much better served
> to solve the underlying problem that causes slow execution in your dispatched calls/tasks.

Amp automatically times out tasks exceeding the (configurable) maximum allowed run-time. We can
customize this setting as shown in the following example:

```php
<?php
(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);

    // Only use one worker so our thread pool acts like a FIFO job queue
    $dispatcher->setOption(Amp\Dispatcher::OPT_POOL_SIZE_MAX, 1);

    // Limit per-call execution time to 2 seconds
    $dispatcher->setOption(Amp\Dispatcher::OPT_TASK_TIMEOUT, 2);

    // This task will timeout after two seconds
    $promise = $dispatcher->sleep(9999);
    $promise->when(function($error, $result) {
        var_dump($error instanceof Amp\TimeoutException); // bool(true)
    });

    // Queue up another task behind the sleep() call
    $promise = $dispatcher->multiply(6, 7);
    $promise->when(function($error, $result) use ($reactor) {
        var_dump($result); // int(42)
        $reactor->stop();
    });
});
```

#### Pool Size

You may have noticed that in the above timeout example we explicity set a max pool size option.
The effect of this setting should be obvious: it controls how many worker threads we spawn to handle
task dispatches. An example:

```php
<?php

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->setOption(Amp\Dispatcher::OPT_POOL_SIZE_MAX, 16);
```

By default the `Amp\Dispatcher` will only spawn a single worker thread. Each time a call is
dispatched a new thread will be spawned if all existing workers in the pool are busy (subject to the
configured max size). The default `OPT_POOL_SIZE_MAX` setting is 8. If no workers are available and
the pool size is maxed calls are queued and dispatched as workers become available.

> **NOTE::** Idle worker threads are periodically unloaded to avoid holding open threads
> unnecessarily.

Dispatchers keep a minimum number of worker threads open at all times (even when idle). By default
the minimum number of threads kept open is 1. This value may be changed as follows:

```php
<?php

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->setOption(Amp\Dispatcher::OPT_POOL_SIZE_MIN, 4);
```


#### Thread Execution Limits

In theory we shouldn't have to worry about sloppy code or extensions playing fast and loose with
memory resources. However in the real world this may not always be an option. Amp makes provision
for these scenarios by exposing a configurable limit setting to control how many dispatches a
worker thread will accept before being respawned to clean up any outstanding garbage. If you wish
to modify this setting simply set the relevant option:

```php
<?php

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->setOption(Amp\Dispatcher::OPT_EXEC_LIMIT, 1024); // 1024 is the default
```

Users who wish to remove the execution limit you may set the value to `-1` as shown here:

```php
<?php

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->setOption(Amp\Dispatcher::OPT_EXEC_LIMIT, -1);
```


#### Pthreads Context Flags

Users can control the context inheritance mask used to start worker threads by setting thread start
flags as shown here:

```php
<?php

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->setOption(Amp\Dispatcher::OPT_THREAD_FLAGS, PTHREADS_INHERIT_NONE);
```

The full list of available flags can be found in the relevant [pthreads documentation page][pthreads-flags].

[pthreads-flags]: php.net/manual/en/pthreads.constants.php "pthreads flags"


### Advanced Usage

#### Stackable Tasks

While Amp abstracts much of the underlying pthreads functionality there are times when low-level
access is useful. For these scenarios Amp allows the specification of "tasks" extending pthreads
[`Stackable`][pthreads-stackables]. Stackables allow users to specify arbitrary code in the main
thread and use it for execution in worker threads.

> **NOTE:** All `Stackable` classes MUST (per pthreads) specify the abstract `Stackable::run()` method

Instances of your custom `Stackable` may then be passed to the `Dispatcher::execute()` method
for processing.

```php
<?php
use Alert\ReactorFactory, After\Future, Amp\Dispatcher, Amp\Thread;

MyTask extends \Stackable {
    public function run() {
        $result = strlen('zanzibar');

        // Stackables must register their results using either of the
        // Amp\Thread::SUCCESS or Amp\Thread::FAILURE codes:
        $this->worker->resolve(Thread::SUCCESS, $result);
    }
}

(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);

    $promise = $dispatcher->call('strlen', 'zanzibar');
    $promise->when(function($error, $result) {
        assert($error === null);
        assert($result === 8);
    });

    // Using execute() to dispatch our custom strlen task
    $promise = $dispatcher->execute(new MyTask); // task object declared above
    $promise->when(function($error, $result) use ($reactor) {
        assert($error === null);
        assert($result === 8);
        $reactor->stop();
    });
});
```

[pthreads-stackables]: http://us1.php.net/manual/en/class.stackable.php "pthreads Stackable"



#### Magic Task Dispatch

`Dispatcher` implementations delegate the magic `__invoke` function to the
`Dispatcher::execute()` method. This provides a simple shortcut method for `execute()` calls:

```php
<?php

class MyTask extends \Stackable {
    public function run() {
        // do something here
    }
}

(new Alert\NativeReactor)->run(function($reactor) {
    $dispatcher = new Amp\Dispatcher($reactor);
    $promise = $dispatcher(new MyTask);
    $promise->when(function($error, $result) use ($reactor) {
        assert($error === null);
        assert($result === 8);
        $reactor->stop();
    });
});
```


#### Class Autoloading

There is no way for pthreads workers to inherit global autoload settings. As a result, if calls
or task executions require class autoloading users must make provisions to register autoload
functions in workers prior to dispatching tasks. This presents the problem of re-registering these
settings each time a worker thread is respawned. Amp resolves this issue by allowing applications to
register `Threaded` tasks to send workers whenever they're spawned.

Consider the following example in which we define our own autoload task and register it for
inclusion when workers are spawned:

```php
<?php

class MyAutoloadTask extends \Stackable {
    public function run() {
        spl_autoload_register(function($class) {
            if (0 === strpos($class, 'MyNamespace\\')) {
                $class = str_replace('\\', '/', $class);
                $file = __DIR__ . "/src/$class.php";
                if (file_exists($file)) {
                    require $file;
                }
            }
        });
    }
}

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->addStartTask(new MyAutoloadTask);
```

Now all our worker threads register class autoloaders prior to receiving tasks or calls. Note that
"start tasks" are stored in an `SplObjectStorage` instance, so repeatedly adding the same instance
will have no effect. After adding a start task you may also remove it in the future as shown here:

```php
$myStartTask = new MyAutoloadTask;

$reactor = new Alert\NativeReactor;
$dispatcher = new Amp\Dispatcher($reactor);
$dispatcher->addStartTask($myStartTask);

// ... //

$dispatcher->removeStartTask($myStartTask);
```


#### Naive Wait Parallelization

Because Amp uses the `After` concurrency primitives library, users don't actually need any understanding
of the underlying non-blocking event loop to execute Amp tasks in parallel. By calling `wait()` on
any promise we can block code execution indefinitely until the promised value resolves:

```php
<?php

try {
    $dispatcher = new Amp\Dispatcher;

    // Dispatch a threaded task
    $promise = $dispatcher->call('strlen', 'zanzibar');

    // Synchronously Block until the promise resolves
    $result = $promise->wait();

    var_dump($result); // int(8)

} catch (Exception $e) {
    printf("Something went wrong:\n\n%s\n", $e->getMessage());
}
```

#### Parallelization Combinators

We can parallelize mutliple threaded operations by using After combinators:

```php
<?php

try {
    $dispatcher = new Amp\Dispatcher;

    $a = $dispatcher->call('sleep', 1);
    $b = $dispatcher->call('sleep', 1);
    $c = $dispatcher->call('sleep', 1);

    // Combine these three promises into a single promise that
    // resolves when all of the individual operations complete
    $comboPromise = After\all([$a, $b, $c]);

    // Our three sleep() operations will complete in one second
    // because they all run at the same time!
    $comboPromise->wait();

} catch (Exception $e) {
    printf("Something went wrong:\n\n%s\n", $e->getMessage());
}
```

Combinator return values are also easily accessible. Consider the following example where we list
the individual results from our parallel calls:

```php
<?php

function add($x, $y) {
    return $x + $y;
}

try {
    $dispatcher = new Amp\Dispatcher;

    $a = $dispatcher->call('add', 1, 2);
    $b = $dispatcher->call('add', 10, 32);
    $c = $dispatcher->call('add', 5, 7);

    // Combine these three promises into a single promise that
    // resolves when all of the individual operations complete
    $comboPromise = After\all([$a, $b, $c]);

    // Wait for the three parallel operations to complete
    list($a, $b, $c) = $comboPromise->wait();
    var_dump($a, $b, $c);

    /*
    int(3)
    int(42)
    int(12)
    */

} catch (Exception $e) {
    printf("Something went wrong:\n\n%s\n", $e->getMessage());
}
```
