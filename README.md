# ko-process #

[![Build Status](https://travis-ci.org/misterion/ko-process.png?branch=master)](https://travis-ci.org/misterion/ko-process)
[![Latest Stable Version](https://poser.pugx.org/misterion/ko-process/v/stable.png)](https://packagist.org/packages/misterion/ko-process)
[![License](https://poser.pugx.org/misterion/ko-process/license.png)](https://packagist.org/packages/misterion/ko-process)

Ko-Process allows for easy callable forking. It is object-oriented wrapper arount fork part of
[`PCNTL`](http://php.net/manual/ru/book.pcntl.php) PHP's extension. Background process, detaching process from the
controlling terminal, signals and exit codes and simple IPC.

# Installation #

### Requirements ###

    PHP >= 5.4
    pcntl extension installed
    posix extension installed


### Via Composer ###

The recommended way to install library is [composer](http://getcomposer.org).
You can see [package information on Packagist](https://packagist.org/packages/misterion/ko-process).

```JSON
{
	"require": {
		"misterion/ko-process": "*"
	}
}
```

### Do not use composer? ###

Just clone the repository and care about autoload for namespace `Ko`.

# Usage #

Basic usage looks like this:

```php
$manager = new Ko\ProcessManager();
$process = $manager->fork(function(Ko\Process $p) {
    echo 'Hello from ' . $p->getPid();
})->onSuccess(function() {
    echo 'Success finish!';
})->wait();
```

If should wait for all forked process
```php
$manager = new Ko\ProcessManager();
for ($i = 0; $i < 10; $i++) {
    $manager->fork(function(Ko\Process $p) {
        echo 'Hello from ' . $p->getPid();
        sleep(1);
    });
}
$manager->wait();
```
### Process title? ###

Yes, both `ProcessManager` and `Process` can change process title with `setProcessTitle` function. Or you may use trait
Ko\Mixin\ProcessTitle to add this to any class you want. Run sample with code

```php
$manager = new Ko\ProcessManager();
$manager->setProcessTitle('I_am_a_master!');

echo 'Press ctrl+c to exit';
while(true) {
    sleep(1);
}
```

and `ps aux|grep I_am_a_master` or `top` to see you process title in linux process list.

### Spawn ###

Making master - child process pattern application you should care about child process be alive. The `spawn` function
will help you with that - once `spawn` will keep forked process alive after he exit with some error code.

```php
$manager = new Ko\ProcessManager();
for ($i = 0; $i < 10; $i++) {
    $manager->spawn(function(Ko\Process $p) {
        echo 'Hello from ' . $p->getPid();
        sleep(1);
        exit(1); //exit with non 0 exit code
    });
}
$manager->wait(); //we have auto respawn for 10 forks
```

Let`s explain you are writing something like queue worker based on AMPQ. So yoy can write something like this
```php

use PhpAmqpLib\Connection\AMQPConnection;
use PhpAmqpLib\Message\AMQPMessage;

$manager = new Ko\ProcessManager();
$manager->setProcessTitle('Master:working...');

$fork = $manager->spawn(function(Ko\Process $p) {
    $connection = new AMQPConnection('localhost', 5672, 'guest', 'guest');
    $channel = $connection->channel();

    $channel->queue_declare('hello', false, true, false, false);

    $callback = function($msg) use (&$p) {
        $p->setProcessTitle('Worker:processJob ' . $msg->body);

        //will execute our job in separate process
        $m = new Ko\ProcessManager();
        $m->fork(function(Ko\Process $jobProcess) use ($msg) {
            $jobProcess->setProcessTitle('Job:processing ' . $msg->body);

            echo " [x] Received ", $msg->body, "\n";
            sleep(2);
            echo " [x] Done", "\n";
        })->onSuccess(function() use ($msg){
            //Ack on success
            $msg->delivery_info['channel']
                ->basic_ack($msg->delivery_info['delivery_tag']);
        })->wait();

        $p->setProcessTitle('Worker:waiting for job... ');

        pcntl_signal_dispatch();
        if ($p->isShouldShutdown()) {
            exit();
        }
    };

    $channel->basic_qos(null, 1, null);
    $channel->basic_consume('hello', '', false, false, false, false, $callback);

    while(count($channel->callbacks)) {
        $channel->wait();
    }

    $channel->close();
    $connection->close();
});
```


### Shared memory and Semaphore ###

The `Ko\SharedMemory` used `Semaphore` for internal locks so can be safely used for inter process communications.
SharedMemory implements `\ArrayAccess` and `\Countable` interface so accessible like an array:

```php
$sm = new SharedMemory(5000); //allocate 5000 bytes
$sm['key1'] = 'value';

echo 'Total keys is' . count($sm) . PHP_EOL;
echo 'The key with name `key1` exists: ' . isset($sm['key1'] . PHP_EOL;
echo 'The value of key1 is ' . $sm['key1'] . PHP_EOL;

unset($sm['key1']);
echo 'The key with name `key1` after unset exists: ' . isset($sm['key1'] . PHP_EOL;
```

You can use `Semaphore` for inter process locking:
```php

$s = new Semaphore();
$s->acquire();
//do some job
$s->release();

//or
$s->tryExecute(function() {
    //do some job
});
```