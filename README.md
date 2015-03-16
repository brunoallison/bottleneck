bottleneck
==========
[npm-downloads]: https://img.shields.io/npm/dm/bottleneck.svg?style=flat
[![Join the chat at https://gitter.im/SGrondin/bottleneck](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/SGrondin/bottleneck?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
![Downloads][npm-downloads]

Bottleneck is a tiny and efficient Asynchronous Rate Limiter for Node.JS and the browser. When dealing with services with limited resources, it's important to ensure that they don't become overloaded.

Bottleneck is the easiest solution as it doesn't add any complexity to the code.

It's battle-hardened, reliable and production-ready. [Unblock.us.org](http://unblock.us.org) uses it to serve millions of queries per day.


#Install

__Node__
```
npm install bottleneck
```
__Browser__
```
bower install bottleneck
```
or
```html
<script type="text/javascript" src="bottleneck.min.js"></script>
```

#Example

Most APIs have a rate limit. For example, the Reddit.com API limits programs to 1 request every 2 seconds.

```javascript
var Bottleneck = require("bottleneck"); //Node only

// Never more than 1 request running at a time.
// Wait at least 2000ms between each request.
var limiter = new Bottleneck(1, 2000);
```

Instead of doing
```javascript
someAsyncCall(arg1, arg2, argN, callback);
```
You do
```javascript
limiter.submit(someAsyncCall, arg1, arg2, argN, callback);
```
And now you can be assured that someAsyncCall will abide by your rate guidelines!

Bottleneck builds a queue of requests and executes them as soon as possible. All the requests will be executed *in order*.

This is sufficient for the vast majority of applications. **Read the [Gotchas](https://github.com/SGrondin/bottleneck#gotchas) section** and you're good to go. Or keep reading to learn about all the fine tuning available for the more complex cases.


#Docs

###Constructor
```javascript
var limiter = new Bottleneck(maxConcurrent, minTime, highWater, strategy);
```

* `maxConcurrent` : How many requests can be running at the same time. *Default: `0` (unlimited)*
* `minTime` : How long to wait after launching a request before launching another one. *Default: `0`ms*
* `highWater` : How long can the queue get? *Default: `0` (unlimited)*
* `strategy` : Which strategy to use if the queue gets longer than the high water mark. *Default: `Bottleneck.strategy.LEAK`.*


###submit()

Adds a request to the queue.

```javascript
limiter.submit(someAsyncCall, arg1, arg2, argN, callback);
```

It returns `true` if the strategy was executed.

####Gotchas

* If a callback isn't necessary, you must pass `null` or an empty function instead. It will not work if you forget to do this.

* Make sure that all the requests will eventually complete by calling their callback! Again, even if you submitted your request with a `null` callback , it still needs to call its callback. This is very important if you are using a `maxConcurrent` value that isn't `0` (unlimited), otherwise those uncompleted requests will be clogging up the limiter and no new requests will be getting through. It's safe to call the callback more than once, subsequent calls are ignored.

* If you want to rate limit a synchronous function (console.log(), for example), you must wrap it in a closure to make it asynchronous. See [this](https://github.com/SGrondin/bottleneck#rate-limiting-synchronous-functions) example.

###strategies

A strategy is a simple algorithm that is executed every time `submit` would cause the queue to exceed `highWater`.

#####Bottleneck.strategy.LEAK
When submitting a new request, if the queue length reaches `highWater`, drop the oldest request in the queue. This is useful when requests that have been waiting for too long are not important anymore.

#####Bottleneck.strategy.OVERFLOW
When submitting a new request, if the queue length reaches `highWater`, do not add the new request.

#####Bottleneck.strategy.BLOCK
When submitting a new request, if the queue length reaches `highWater`, the limiter falls into "blocked mode". All queued requests are dropped and no new requests will be accepted into the queue until the limiter unblocks. It will unblock after `penalty` milliseconds have passed without receiving a new request. `penalty` is equal to `15 * minTime` (or `5000` if `minTime` is `0`) by default and can be changed by calling `changePenalty()`. This strategy is ideal when bruteforce attacks are to be expected.


###check()
```javascript
limiter.check();
```
If a request was submitted right now, would it be run immediately? Returns a boolean.


###stopAll()
```javascript
limiter.stopAll(interrupt);
```
Cancels all *queued up* requests and prevents additonal requests from being submitted.

* `interrupt` : If true, prevent the requests currently running from calling their callback when they're done. *Default: `false`*


###changeSettings()
```javascript
limiter.changeSettings(maxConcurrent, minTime, highWater, strategy);
```
Same parameters as the constructor, pass ```null``` to skip a parameter and keep it to its current value.

**Note:** Changing `maxConcurrent` and `minTime` will not affect requests that have already been scheduled for execution.

For example, imagine that 3 60-second requests are submitted at time T+0 with `maxConcurrent = 0` and `minTime = 2000`. The requests will be launched at T+0 seconds, T+2 seconds and T+4 seconds respectively. If right after adding the requests to Bottleneck, you were to call `limiter.changeSettings(1);`, it won't change the fact that there will be 3 requests running at the same time for roughly 60 seconds. Once again, `changeSettings` only affects requests that have not yet been *submitted*.

This is by design, as Bottleneck made a promise to execute those requests according to the settings valid at the time. Changing settings afterwards should not break previous assumptions, as that would make code very error-prone and Bottleneck a tool that cannot be relied upon.


###changePenalty()
```javascript
limiter.changePenalty(penalty);
```
This changes the `penalty` value used by the `BLOCK` strategy.


###changeReservoir(), incrementReservoir()
```javascript
limiter.changeReservoir(reservoir);

limiter.incrementReservoir(incrementBy);
```
* `reservoir` : How many requests can be executed before the limiter stops executing requests. *Default: `null` (unlimited)*

If `reservoir` reaches `0`, no new requests will be executed until it is no more `0`

###chain()

* `limiter` : If another limiter is passed, tasks that are ready to be executed will be submitted to that other limiter. *Default: `null` (none)*

Suppose you have 2 types of tasks, A and B. They both have their own limiter with their own settings, but both must also follow a global limiter C:
```javascript
var limiterA = new Bottleneck(...some settings...);
var limiterB = new Bottleneck(...some different settings...);
var limiterC = new Bottleneck(...some global settings...);
limiterA.chain(limiterC);
limiterB.chain(limiterC);
// Requests submitted to limiterA must follow the A and C rate limits.
// Requests submitted to limiterB must follow the B and C rate limits.
// Requests submitted to limiterC must follow the C rate limits.
```

##Execution guarantee

Bottleneck will execute every submitted request in order. They will **all** *eventually* be executed as long as:

* `highWater` is set to `0` (default), which prevents the strategy from ever being run.
* `maxConcurrent` is set to `0` (default) **OR** all requests call the callback *eventually*.
* `reservoir` is `null` (default).


# Cluster

The main design goal for Bottleneck is to be extremely small and transparent to use. It's meant to add the least possible complexity to the code.

Let's take a DNS server as an example of how Bottleneck can be used. It's a service that sees a lot of abuse. Bottleneck is so tiny, it's not unreasonable to create one limiter of it for each origin IP, even if it means creating thousands of limiters. The `Cluster` mode is perfect for this use case.

The `Cluster` feature of Bottleneck manages limiters automatically for you. It is created exactly like a limiter:

```javascript
var cluster = Bottleneck.Cluster(maxConcurrent, minTime, highWater, strategy);
```

Those arguments are exactly the same as for a basic limiter. The cluster is then used with the `.key(str)` method:

```javascript
cluster.key("somestring").submit(someAsyncCall, arg1, arg2, cb);
```

###key()

* `str` : The key to use. All calls submitted with the same key will use the same limiter. *Default: `""`*

The return value of `.key(str)` is a limiter. If it doesn't already exist, it is created on the fly. Limiters that have been idle for a long time are deleted to avoid memory leaks.

###all()

* `cb` : A function to be executed on every limiter in the cluster.

For example, this will call `stopAll()` on every limiter in the cluster:

```javasript
cluster.all(function(limiter){
	limiter.stopAll();
});
```

###keys()

Returns an array containing all the keys in the cluster.


# Rate-limiting synchronous functions

Most of the time, using Bottleneck is as simple as the first example above. However, when Bottleneck is used on a synchronous call, it (obviously) becomes asynchronous, so the returned value of that call can't be used directly. The following example should make it clear why.

This is the original code that we want to rate-limit:
```javascript
var req = http.request(options, function(res){
	//do stuff with res
});
req.write("some string", "utf8");
req.end();
```

The following code snippet will **NOT** work, because `http.request` is not executed synchronously therefore `req` doesn't contain the expected request object.
```javascript
// DOES NOT WORK
var req = limiter.submit(http.request, options, function(res){
	//do stuff with res
});
req.write("some string", "utf8");
req.end();
```

This is the right way to do it:
```javascript
limiter.submit(function(cb){
	var req = http.request(options, function(res){
		//do stuff with res
		cb();
	});
	req.write("some string", "utf8");
	req.end();
}, null);
```

# Contributing

This README file is always in need of better explanations and examples. If things can be clearer and simpler, please consider forking this repo and submitting a Pull Request.

Suggestions and bug reports are also welcome.
