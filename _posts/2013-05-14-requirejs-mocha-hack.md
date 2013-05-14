---
layout : post
category : programming
tags : [javascript, node, mocha, testing]
---
{% include JB/setup %}
[RequireJS](http://requirejs.org/docs/node.html) is great on the client side.  It handles the asynchronous nature of the web fairly well, allowing you to have control over the order in which your javascript files are loaded.  Tools like require.js make it possible to build complicated client side apps in a sane manor.

Node is great for many reasons, but one that almost always gets mentioned is that it lets you share code between your server and client side.  The re-use of code is a pretty tantalizing lure, but unfortunately, the promise of it isn't as straight forward as it seems.  Node's built in require system (at least the way its implemented out of the box) isn't particularly compatible with the web.

    {% highlight javascript %}
    //node, built-in
    var Widget = require('./widget');

    var widget = new Widget();
    {% endhighlight %}

That format inherently blocks.  The script simply can't keep going with other stuff until Widget gets declared.  If widget.js is off on a server somewhere, that's a delay.  If you're loading a bunch of files, it can quickly add up to an unacceptably long delay.

RequireJS gets around this by wrapping all its loading inside of callbacks.  In the example below, this code relies on the sound.js file getting loaded.  Because the code that relies on sound.js is inside of a callback, requirejs can busy itself loading other things before running it.

    {% highlight javascript %}
    //require.js, client side
    define(['sound'], function(Sound) {
      var public = {}
        , now = function() { return Sound.getCtx().currentTime; }
        ;

      ...

      return public;
    })
    {% endhighlight %}

While node's built in module system is incompatible with the web, there's nothing incompatible about RequireJS's system and node's.  The two can exist fairly nicely side by side.  In fact, they kind of need to, as one of the first steps that's necessary to load files through RequireJS is to load RequireJS through node's built in module system.

    {% highlight javascript %}
    //require.js, server side
    var requirejs = require('requirejs');

    requirejs(['./lib/deletion-array.js'],
      function(DeletionArray) {
      ...
    })
    {% endhighlight %}

I should note, that if you are compiling all of your client side code into one big javascript file, these concerns about the blocking nature of node's require syntax basically go away.  And there is a tool to let you do just that.  But if you don't want your development process to involve a compilation step, and you also want to reuse code without having different versions for client and server side, you need to go the requirejs route.

But this decision will lead to some annoyances.  The biggest of which (for me) was that the switch completely broke my server-side [Mocha](http://visionmedia.github.io/mocha/) tests.  I would run them, and mocha would fail to find any tests to run.  The reason for this is that when mocha first scans through the files, none of the tests declared within the requirejs callback have actually been added to mocha's queue of tests yet.  That callback won't get called until deletion-array.js gets loaded, which in turn won't happen until after mocha has scanned through the test file and seen that it has no tests left to run and should terminate.

    {% highlight javascript %}
    requirejs(['./lib/deletion-array.js'],
      function(DeletionArray) {

      describe('DeletionArray', function() {
        it('allow logarithmic speed deletions from arbitrary points in the array', function() {
        });
        ...
      });
      ...
    });
    {% endhighlight %}

There is a pretty simple, if totally hacky way, to get around this though.  You just have to make sure that there are tests running for long enough for requirejs's callback to get called.  How long is that?  In my fiddling, this test was sufficient:

    {% highlight javascript %}
    describe('Some hacky nonsense', function() {
      it('enables the tests to run with requirejs', function(done) {
        setTimeout(done, 10);
      });
    });
    {% endhighlight %}

In fact, setting the timeout to 0 milliseconds, or even removing it entirely, was sufficient to get the tests to run.  But out of an abundance of caution, I'm leaving the 10 millisecond delay in my code.  Since I don't want this hack to dirty my other tests, I've moved this code to its own file, clearly labeling it as a hack.  Hopefully, by the time you read this, this unfortunate hack will be unnecessary.  Until that time, I hope it helps.

