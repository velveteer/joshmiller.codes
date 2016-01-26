+++
date = "2014-04-13T15:43:24-06:00"
draft = false
title = "Understanding co"

+++

Or perhaps more suitably titled "How I learned to stop worrying and love the thunk". The generator and the coroutine are not new concepts, but they are now a part of ECMAScript 6, and this allows us to play with cooperative multi-tasking in Node unstable via the `--harmony` flag. 

Why are generators worth implementing into the standard? The reasons are numerous, and anyone familiar with call/cc in Scheme, async/await in C#, or yield in Python will understand that re-entrant functions are a powerful low-level construct. Low-level because we don't manage the implicit state created by suspended functions, this is something the interpreter will handle. We are only concerned with how to pause, resume, and send values to generators. 

Further reading:  
[http://blog.alexmaccaw.com/how-yield-will-transform-node](http://blog.alexmaccaw.com/how-yield-will-transform-node)  
[http://devsmash.com/blog/whats-the-big-deal-with-generators](http://devsmash.com/blog/whats-the-big-deal-with-generators)  

Generators by themselves aren't going to provide much for the average Javascript programmer beyond being factories for iterators. But the ability of generators to send *and* receive values is what enables cooperative multi-tasking.
A few really smart people figured out how to use generators to handle flow-control of asynchronous functions, similar to how others use futures and promises. I like how co does it, but to each their own. The basic idea is that a generator can suspend its execution and yield a partially applied async function (also known as a thunk) or a promise to the caller. Exactly how this works varies between different libraries, so for a broader perspective I recommend looking at:

* [async](https://github.com/caolan/async)
* [Q](https://github.com/kriskowal/q)
* [co](https://github.com/visionmedia/co)
* [suspend](https://github.com/jmar777/suspend)
* [galaxy](https://github.com/bjouhier/galaxy)
* [genny](https://github.com/spion/genny)
* [gen-run](https://github.com/creationix/gen-run)


What most of them have in common is a way for the caller to handle a yielded thunk. Thunks are important in co because we want the *caller* of the generator to execute the async function and only resume the generator once the async function calls back. This works because the caller executes the thunk while passing it a callback, and this callback function will resume the generator with a value or throw an error inside of it. Most callback functions in Node should be using the cb(error, response) signature. Co assumes this convention, whereas suspend provides `suspend.resumeRaw()` for non-conventional callback signatures and galaxy can handle them [for the most part](https://github.com/bjouhier/galaxy#odd-callbacks). 

Further reading:  
[http://fredkschott.com/post/2014/03/understanding-error-first-callbacks-in-node-js/](http://fredkschott.com/post/2014/03/understanding-error-first-callbacks-in-node-js/)  


Regular callback:

    fs.readFile(path, 'utf8', function(err, res) {
        //handle error
        //parse response
    });

Thunkified:

    function readFile(path) {
        return function(callback) {
          fs.readFile(path, 'utf8', callback);
        };
    }
    // Called via readFile(path)(callback)
    // Convention dictates a callback in the form cb(error, result)


My explanation is a bit rough, but there are plenty of better examples of how this can work. One of my favorite tours of this functionality is written by TJ Holowaychuk (the author of the co library): [https://medium.com/code-adventures/174f1fe66127](https://medium.com/code-adventures/174f1fe66127)

He gives a very basic example of how flow control can be accomplished with generators:

    var fs = require('fs');

    function thread(fn) {
      var gen = fn();
      function next(err, res) {
        var ret = gen.next(res);
        if (ret.done) return;
        ret.value(next);
      }
      
      next();
    }

    thread(function *() {
      var a = yield read('Readme.md');
      var b = yield read('package.json');
      console.log(a);
      console.log(b);
    });

    function read(path) {
      return function(cb) {
        fs.readFile(path, 'utf8', cb);
      }
    }


What I want to do here is step through each part of this basic example to show how it works. This is pretty much a simplication of what co does under the hood.

Before we dive into this, I recommend reading Zef Hemel's amazing article on co here: [http://zef.me/6096/callback-free-harmonious-node-js](http://zef.me/6096/callback-free-harmonious-node-js). However, keep in mind that this method is not "callback-free" -- we are definitely still using callbacks, but they are being abstracted from typical Node CPS.

First let's look at what we're doing in the generator:


    thread(function *() {

        // Suspend execution and send the partial function to the
        // caller (the thread wrapper function)
        var a = yield read('Readme.md');
        ...
    });


Following the partially applied function is now our task. Where did it go? Remember that we yielded something that looks like:


    // Thunky fresh
    function(cb) { fs.readFile('Readme.md', 'utf8', cb) }; 



We yielded this to the thread function, so the current execution context looks like this:


    // fn == the generator function *() from above

    function thread(fn) {

        // Initializes the generator object
        var gen = fn();

        // Looks familiar? It is the signature of a callback function
        function next(err, res) {

          // next() is used to resume the generator from where it left
          // off, and it can also take values to send to the generator
          
          // On the first call to next() res is null, so the
          // generator yields the thunk.
          // We will pass in our own cb that calls next() with 
          // the result of the async operation
          var ret = gen.next(res);
          
          // ret == { value: thunk, done: false }
          // If the generator has nothing left to yield it returns done: true
          if (ret.done) return;
          
          // Execute the thunk, passing in next(err, res) as the callback
          ret.value(next);
        }

        next();
    }


So the generator is still suspended, and we are running the async function with a callback that sends the result back to the generator:


    fs.readFile('Readme.md', 'utf8', next(err, res) {

        // Will resume generator and send res
        // "var a" in the generator will be assigned to the value of res
        gen.next(res);
        ...
    );


When the callback is executed, we resume the generator and send it the result of the async operation. The generator will then continue until another yield or until it returns (run-to-completion).

Ultimately each generator is blocked while waiting for the callback to resume it, and this can benefit flow control in a way similar to `promise.then()` or `future.wait()`. They can each kick off their own async operations, and they will only continue once those operations have completed or an error is thrown. With co, you can even nest generators and perform parallel async operations by yielding an array. I think that is pretty cool. An example can be seen here: [https://github.com/visionmedia/co/blob/master/examples/generator-array.js](https://github.com/visionmedia/co/blob/master/examples/generator-array.js)

Using generators wrapped by co definitely cleans up our syntax and gives us synchronous-looking code that supports try/catch blocks. The downside is that every function that would accept a callback now needs to be wrapped into a thunk, and nested generators can be difficult to debug. I do not think that thunks are a bad practice, but they do change the signature of async functions. The `suspend.resume()` function from the [suspend library](https://github.com/jmar777/suspend#suspendresume) offers to let us keep using a traditional signature where the last argument of an async operation is assumed to be a callback.

I have been using co and various other thunkified libraries like co-monk, co-redis, and koa. I really think this approach to async flow control will help make Node a better place, but browser support for generators is still in flux. I recommend checking out co because it supports promises on top of thunks, and there are already a ton of thunkified libraries that integrate with it: [https://github.com/visionmedia/co/wiki](https://github.com/visionmedia/co/wiki)