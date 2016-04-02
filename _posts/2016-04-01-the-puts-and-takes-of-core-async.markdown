---
layout: post
title: "The puts and takes of core.async"
date: 2016-04-01 22:00:00
---

I've worked on a *whole* bunch of [core.async](https://clojure.github.io/core.async/) code
over the last couple of years and on the whole it's a pleasure to work with. After some
practice it feels like a very natural way to model concurrent programs. But everything has
tradeoffs, doesn't it? Let's talk about some of these tradeoffs and gotchas and what we
can do to solve them.

# _Queues rule everything around me._

This is probably the most often mentioned design decision, but it takes awhile for the implications
to sink in. Core.async channels function a little bit like queues, and they've been wisely designed
to disallow unbounded queues. A channel can have length 0, which will cause any put to block until
that value has been taken off the channel. If a channel is created with a length, then it can continue
to take values up to the channel size, at which point it will block. Core.async comes with sliding and
dropping buffers out of the box so you can configure the channel to drop the first or last message if
the buffer is full as well.

What *isn't* immediately obvious is that our queues have queues. A put to a core.async channel is 
put in a queue which is bounded at 1024 per channel, and if this queue is exceeded we get an exception.
This is consistent with the ideas of core.async, which nudge us to think about the size of our queues
and tune them so our systems run well without anything running out of memory, but there are a few mistakes
I've seen once or twice.

For example, let's say we have a go-loop consuming from some data stream (doesn't matter what it is),
and publishing it to a channel for the system or some data pipeline to consume. Let's write a little
bit of code to do that.

{% highlight clojure %}

(defn stream-consumer [some-stream out>]
  (a/go-loop []
    (when-let [stream-value (take some-stream)]
      ;;Let's assume that we can just pass on the raw value.
      (a/put! out>)
      (recur))))

{% endhighlight %}

Seems innocent enough, doesn't it? Unfortunately in this case we've messed up a little bit. Because we're
using ```put!```, the asynchronous put, we're going to recur immediately and get the next value from our
stream. If we have enough data available to take (I've seen this issue with a reporting service so, uh, 
lots of data), we'll eventually exhaust our buffer and clojure will let us know.

```
java.lang.AssertionError: 
Assert failed: 
No more than 1024 pending puts are allowed on a single channel.
Consider using a windowed buffer.
```

This is a really fun error because it does a few things I plan to talk about when it's thrown inside a 
```go-loop```.

1. It, like any other error value, will kill our go-loop, and by default this error won't even show up anywhere.

2. It's not even an Exception, it's a throwable, so our best friend ```(catch Exception e ...)``` will be of no help to us.

Okay so we made a little mistake, in this case it's not even that hard to fix.

{% highlight clojure %}

(defn stream-consumer [some-stream out>]
  (a/go-loop []
    (when-let [stream-value (take some-stream)]
      ;;Let's assume that we can just pass on the raw value.
      (a/>! out> stream-value)
      (recur))))

{% endhighlight %}

That's it, we just make that put a parking put and all is well again.

We hope.

# _All is not well again_

A lot of really smart people tell us that when we design asynchronous libraries,
we should abstract away that code and provide interfaces that seem synchronous
in some way. This is great because it lets us treat those libraries as black boxes,
this is terrible because it lets us treat those libraries as black boxes.

Let's look at an instance of this error in the core.async code.

Mixes are pretty cool! They let us blend a bunch of different channels together and
even turn them on and off at will. I haven't used them much but when I have they've
proven to be incredibly powerful. Let's say we're watching a filesystem and we want
to bind a *whole bunch* of callbacks to send change events for a given file. Each
callback creates a channel with the type of event and some data about the state change
for a file. We want to be able to add or ignore files or directories at runtime, so
passing a channel around is a little difficult to do. Mix to the rescue!

{% highlight clojure %}

(defn watch-file [filename mix]
  (let [out> (a/chan 32768)]
    ;; Handwavey use of clojure-watch
    (watch/start-watch [{:path filename
                         :event-types [:create :modify :delete]
                         :bootstrap (fn [filename]
                                      (a/admix mix out>))
                         :callback (fn [event filename]
                                     ;; This is just a put! in disguise, don't be fooled!
                                     ;; But our channel is huge so it should be okay.
                                     (a/go (a/>! out> [event filename])))}])))

{% endhighlight %}

This looks pretty good. Unless a given file is changed tens of thousands of times in very
short order everything should be fine.

After running the system for awhile and watching a whole bunch of file, our ```AssertionError``` comes back.

![but why](http://i.imgur.com/TnQRX6v.gif)

Well, it turns out that ```mix``` [has the issue I've described](https://github.com/clojure/core.async/blob/master/src/main/clojure/clojure/core/async.clj#L735-L739).

The ```changed``` channel only passes in a ```true``` to notify the mix that channels have been added to or removed from
the mix so it can update its state accordingly. It doesn't matter how many changes have happened, only that there's been
a change. ```changed``` is also bounded at 0, so if 1024 channels are added to the mix in very short order, the mix dies
with an assertion error.

Luckily, it's an easy fix. Since we don't care about the contents of changed, and multiple changes are idempotent, we can
just build that channel with a dropping or sliding buffer.

For those who are interested, the issue and the fix is [here](http://dev.clojure.org/jira/browse/ASYNC-145), it's a one-liner.

If you use mix at all and expect to have large additions and subtractions of channel, I'd suggest applying this fix to your
own code.

It's worth noting that ```onto-chan``` is also not blocking, so using it in a ```go-loop``` can be a Bad Idea.

# _There's Plenty More To Talk About_

I hope this has been at least a little helpful if you work with ```core.async``` or are planning to. I'm going to talk a lot
more about the challenges of building and running a ```core.async``` system. The next topic should keep us busy for quite awhile.
Buckle up, it's going to be about error handling.
