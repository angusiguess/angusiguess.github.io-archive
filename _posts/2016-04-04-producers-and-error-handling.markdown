---
layout: post
title: "The puts and takes of core.async, part !!: error handling for producers."
date: 2016-04-04 22:00:00
---

### Update: April 7

Reddit user halgari (who wrote the core.async go macros and has an *excellent* series of videos on how they work,
which can be found [here](https://www.youtube.com/watch?v=R3PZMIwXN_g)) 
pointed out that using a ```>!!``` in a function that would be
placed in a go-loop can lead to some nasty bugs and block the thread pool. I've
updated my code to reflect this.

@priyatam on the clojurians slack pointed out some issues running the code, I've
resolved these as well.

Working with ```core.async``` over time, we start to see some idioms and patterns
emerge. We'll start with probably the simplest one, the ```producer```. If anyone
has a better name for this pattern I'd be happy to hear it, but for now ```producer```
should be okay.

A producer is applied when we need to:

1. Generate a stream of values or fetch them from an outside source.
2. Place them on a channel.

That's it! They usually function as some kind of i/o for an application, here are
some examples:

* Consuming values from a message queue, database, or log file.
* Handling key, mouse, and touch events from the browser.

They're simple and powerful, let's take one apart:

Suppose we have a web store of some sort, and we want to track clicks.
These clicks are published to an append-only log, like kafka.

Our kafka client is pretty simple: It gives us a message based on the ```offset```
we provide to it.

So ```(take! connection 0)``` will give us the first message, and so on.

This append-only log doesn't track a whole lot of state, so it's up to our
producer to track the offset and increment it after we've consumed a message.

{% highlight clojure %}

(defn start []
  (let [connection (client/simple-consumer 10)
        ;; This is our data source.
        publish> (a/chan)
        ;; This is the channel we'll return to connect to the rest of our system.
        ]
    (a/go-loop [offset 0] ;; This producer is a pretend kafka stream, so we track the offset while we're reading.
      (let [click (client/take! connection offset)]
       ;; A click is just a map that looks like this:
       ;; {:user-id <some uuid>
       ;;  :click <one of #{:add-to-cart :shipping :payment}>}
        (a/>! publish> click)
        (recur (inc offset))))
    publish>))
    
{% endhighlight %}

That's it! The producer will keep pulling values off of our log and putting them
on the ```publish>``` channel.

It respects backpressure, so it will only read values so long as the rest of the
system downstream can process them, and otherwise should perform nicely.

But we talk to our log over the network. And networks are, well...

![Unreliable](http://cdn.makeagif.com/media/2-18-2015/EGRce3.gif)

Unreliable.

# Error Handling

Our fake client looks like this:

{% highlight clojure %}

(defprotocol LogConsumer
  (take! [this offset]))

;; A *very* simple model of a consumer for an append-only log. Initialized with
;; a vector of values and provides them to a client by 'offset' which in this case
;; is just the index of that value.

;; This client throws two exceptions, one if an invalid offset is specified
;; (we've read every message on the log or given a negative offset)
;; And one simulated session timeout. There's a 1/1000 chance that when
;; we try to take! from the client it'll throw an exception.

(defrecord SimpleConsumer
    [events]
  LogConsumer
  (take! [this offset]
    (if (= 0 (rand-int 1000)) (throw (kafka-exception offset))
        (try (nth events offset)
             (catch Exception e
               (throw (invalid-offset-exception offset)))))))

(defn simple-consumer [n]
  (let [events (into [] (gen/sample gen-click-data n))]
    (SimpleConsumer. events)))


{% endhighlight %}

I'm going to crib liberally from Joe Armstrong's wonderful dissertation
[Making Reliable Systems in the Presence of Software Errors](http://erlang.org/download/armstrong_thesis_2003.pdf),
because I believe that ```core.async``` shares some of the same goals as
Erlang does. We'll discuss this more later, but there are three important features
that the two share:

1. Isolation of state.
2. Message passing.
3. Isolation of errors.

1 and 2 are very closely connected. Keeping state local to a go block or an actor
means that we only need to worry about one piece of code modifying our data.

We *can* share state in core.async (atoms, records with mutable fields), but I would
argue that it's generally a bad idea and I don't think I've ever had cause to do it.

Message passing is just a consequence of this. If we can't share state we need for
go blocks to be able to interact somehow. Values over channels function as messages
because we assume nothing about the receiving process.

Error isolation is interesting. For a long time I thought the way ```core.async```
silently swallowed exceptions was a flaw or a necessary evil of its design. Reading
Armstrong's thesis turned me around a bit.

### For a ```core.async``` system to continue to function in the presence of errors, each piece of it should be able to fail independently without affecting the others.

Unfortunately, this producer doesn't satisfy that. It will chug along happily until
one of our session exceptions is thrown and then it will stop. It won't log anything,
the channel will just stop producing values. This will result in anything downstream
of our producer halting as well, since there are no more values to take.

## _We can do better_

Let's pull everything that would operate in the go-loop into its own function:

{% highlight clojure %}


(defn publish-click
  ([connection offset publish>]
   ;; Catch any errors thrown in this function.
   (a/go (try (let [click (client/take! connection offset)]
                (a/>! publish> click)
                ;; We tag any of our return values with a message type.
                ;; For something as simple as this producer it isn't immediately useful
                ;; But this will come in handy for more complex patterns.
                [:read (inc offset)])
              (catch Throwable e
                [:error {:exception e
                         :offset offset
                         :operation :publish-click}])))))
                   
{% endhighlight %}

And modify our go-loop to call the function instead.

{% highlight clojure %}

(defn start
  ([publish> offset]
   (let [connection (client/simple-consumer 10000)]
     (a/go-loop [offset offset]
       ;; We use the message tag to determine if an error occurred. If it did, we return
       ;; the message state.
       (let [[msg-type state :as msg] (a/<! (publish-click connection offset publish>))]
         (if (= msg-type :error)
           state
           (recur state))))))
  ([publish>]
  ;; If no argument is provided, assume we should start reading at the beginning.
   (start publish> 0)))

{% endhighlight %}

Now, our go-loop has virtually no logic in it aside from message dispatch. It also returns a value
if an error occurs. This will allow us to introduce our next pattern:

## _A very simple supervisor_

Suppose we caught this problem in one of our production systems, and while we want to solve this problem
better, in the meantime we're going to handle the connection error. This will probably get us in trouble
later but luckily our workplace is a work of fiction so I can make a point.

{% highlight clojure %}

(defn start-supervisor []
  (let [publish> (a/chan 1024)]
    (a/go-loop [producer (start publish>)]
      (println "Starting producer.")
      (let [{:keys [offset] :as error} (a/<! producer)]
        (println "An error has occurred: " error)
        (println "Restarting producer.")
        (recur (start publish> offset))))
    publish>))

{% endhighlight %}

This will keep our producer going and restart it in the event of an error. And, dear reader,
there _will_ be errors.

But there are some immediate benefits as well! The first is that we've achieved isolation for
our producer. Now if it fails we can apply whatever recovery logic we like and continue producing
to the same channel without making any changes to the rest of our pipeline. A small step, to be sure,
but our system is the better for it.

Next time we'll see if we can't work all of these great ad-hoc patterns into something a little
more reusable.

The code for all of these examples can be found [here](https://github.com/angusiguess/async-blog-code)
