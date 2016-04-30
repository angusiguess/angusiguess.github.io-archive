---
layout: post
title: "The puts and takes of core.async, part !!!: You would be downright amazed at what I can do with just a pipeline."
date: 2016-04-30 10:53:00
---

The companion code to this post can be found [here](https://github.com/angusiguess/wikipedia-edit-tracker)

# Updates

Reddit user nefreat recommends [this post](http://elbenshira.com/blog/understanding-transducers/) for more info on transducers and comp ordering with transducers.

# The struggle

Initially my goal for this post was to write about error handling in pipelines.
I wrote some code and experimented with a bunch of scenarios and concluded that
such a post might be boring and that there wouldn't be much meat there. This is
because pipelines are, in my opinion, at the time of this writing, the single most
robust abstraction in the entire core.async quiver.

I am not even joking. Let's take a look at what pipelines manage to pack in to a
few small functions:

* Composability
* Error Handling
* Testability
* Easy mode parallelism (just change a number.)
* Ordering guarantees.

Let's talk about these.

# Transducers

In order to understand what pipelines give us for composition, first we have to
talk a little bit about transducers. I'm not going to attempt to write down 
[type signatures](http://conscientiousprogrammer.com/blog/2014/08/07/understanding-cloure-transducers-through-types/)
or even try to write any transducers from scratch right now. I highly encourage
you to take a look at the clojure.core code for ```map```, ```filter```,
and ```distinct```. To start building pipelines, though,
all we really need to care about is how to use the mapping and filtering transducers.

The tl;dr for why transducers are worth your time is this: *Transducers don't care
where data comes from, they only care about how to transform it.* It is for this
reason that transducers can operate over collections, operate lazily and eagerly,
and be packed up with a bunch of data and passed around for later evaluation (I've never
had to write an eduction in my working life but we have an informal competition at
work to find the use case and I've no doubt there is one).

What does this mean for us? Transducers are the natural way to operate on data coming
through channels.

Enough talk, here are some examples. The project that I started working on for this post
is a system for tracking wikipedia edits. There were two possible ways to do this, one
is through a socket.io websocket (I had very little luck finding a clojure client that
would readily consume it) and, somewhat amusingly, an IRC channel.

I mean it sounds funny but it's actually a great solution to the problem. A channel can
operate as a pub-sub system that bots can readily consume *and* all of the events are
human-readable as well, which made it pretty fun to figure out how to consume and groom
these events for our purposes.

Let's assume we have an IRC producer (I'll publish my code for it, I make no guarantees
about its stability as it's build on top of an [old Raynes library](https://github.com/Raynes/irclj)).
As clojurists, our first instinct is to turn this into structured data.

Here's what an event from the producer looks like:

{% highlight clojure %}

{:command "PRIVMSG"
 :params ["#en.wikipedia" "14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */"]
 :raw ":rc-pmtpa!~rc-pmtpa@special.user PRIVMSG #en.wikipedia :14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */",
 :nick "rc-pmtpa",
 :user "~rc-pmtpa",
 :host "special.user",
 :target "#en.wikipedia",
 :text "14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */"}

{% endhighlight %}

First things first, let's look at this map and see what we care about.

* Command is useful because we may get a number of message types from IRC, and
  we're interested in PRIVMSG.
  
* Nick is useful because only one user actually publishes these edit events. These
  channels aren't very noisy but if they were we would end up getting bad data.
  
* Target isn't useful to us now, but if we wanted to watch *all* of the edit channels
  for *all* of wikipedia, this would allow us to split that data when the time came.
  
* Text contains pretty much all of the metadata that we might want to do ~big data~ on.

So we want to map the above event to something that looks like this:

{% highlight clojure %}

{:command "PRIVMSG"
 :nick "rc-pmtpa",
 :target "#en.wikipedia",
 :text "14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */"}

{% endhighlight %}

That's a little less noisy, isn't it?

# Baby's first transducer.

If we wanted to write a function to do this, it would be pretty small, wouldn't it?
It's already in ```core```, in fact.

{% highlight clojure %}

(select-keys data [:command :nick :target :text])

{% endhighlight %}

Now if we had a list of this data that we wanted to transform eagerly, we'd ```map```
this function over that list. If we don't provide a collection to map, though, we get
a *mapping transducer*. All that means is that we can plug this into a number of functions
to perform this transformation.

When I was testing it in the REPL, I did just that, building my transducer and using ```into```
to pass my collection of input into a collection of output.

{% highlight clojure %}

(def tx-process-irc-data (map #(select-keys % [:command :nick :target :text])))

(into [] (tx-process-irc-data) [{:command "PRIVMSG"
                                          :params ["#en.wikipedia" "14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */"]
                                          :raw ":rc-pmtpa!~rc-pmtpa@special.user PRIVMSG #en.wikipedia :14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */",
                                          :nick "rc-pmtpa",
                                          :user "~rc-pmtpa",
                                          :host "special.user",
                                          :target "#en.wikipedia",
                                          :text "14[[07List of intelligence gathering disciplines14]]4 10 02https://en.wikipedia.org/w/index.php?diff=715127585&oldid=705758737 5* 03Dainomite 5* (+24) 10/* See also */"}])



{% endhighlight %}

# *Insight One: Transducers are testable as all get-out.*

Because transducers decouple our actual transformations from the data we pass through them,
we can easily supply mock data to a transducer and assert on what comes out the other side.
We can run tests on large samples of captured data, or if we're feeling sassy we can write
generators to generate all kinds of valid or invalid data.

# This is nice but we still have work to do.

Next up, let's filter out everything that doesn't come from the producer of these events. Their
irc nick is "rc-pmtpa".

This is another one-liner, all we need to do is pull the ```:nick``` key out of the map and
filter on it.

{% highlight clojure %}

(filter #(= (:nick %) "rc-pmtpa"))

{% endhighlight %}

How, pray tell, are we going to combine these two transducers?

# *Insight two: Transducers are composable*

Let's just alter that previous tx definition a bit.

{% highlight clojure %}

(def tx-process-irc-data
    (let [get-keys (map #(select-keys % [:command :nick :target :text]))
          filter-nick (filter #(= (:nick %)) "rc-pmtpa")]
          (comp get-keys filter-nick)))

{% endhighlight %}

That's it, we can compose these two functions and get a function that will:

 * filter out the nick we don't want
 * select only the keys we want.

# *Gotcha one*

```comp``` returns a function which applies the right-most of its arguments
first, and each other argument to the left-most. This would lead us to believe
that composed transducers work the same way, right?

Unfortunately, no. Here's a quick REPL session:

{% highlight clojure %}

user> ((comp + str) 8 8 8)
ClassCastException Cannot cast java.lang.String to java.lang.Number  java.lang.Class.cast (Class.java:3369)
user> ((comp str +) 8 8 8)
"24"
user> ;; We can't add strings, we need to do the addition first.
user> (def tx-one (comp (map +) (map str)))
#'user/tx-one
user> (def tx-two (comp (map str) (map +))
#'user/tx-two
user> (def tx-one (comp (map +) (map str)))
#'user/tx-one
user> (into [] tx-one [8 8 8])
["8" "8" "8"]
user> (into [] tx-two [8 8 8])
ClassCastException Cannot cast java.lang.String to java.lang.Number  java.lang.Class.cast (Class.java:3369)
user> ;; Oh what

{% endhighlight %}

I'd like to explain this but if I start tracing transducer function signatures right now I'll never get anything done.
If anybody would like to get at me with a nice explanation of this, I'll be happy to add it and credit you.

TL;DR, composition of transducers works effectively from left to right. This is nice because that's how we tend to think
of pipelines, but just be aware that there's a tiny inconsistency when we use comp.

**Transducers all night, apply from left to right**

**Functions mos def, apply from right to left**

You've got the idea though, we can stick as many of these together as we'd like. They're
small, testable, composable, and way more efficient than the alternative, which is to write
either one big go-loop that does more than we'd like it to, or write a bunch of small go-loops
that don't run in parallel and force us to hold intermediate copies of all of our data.

# Let's fast forward a bit.

Thanks for sticking with me so far, I know this is long but there's a lot to talk about. Let's skip
to our completed transducer.

{% highlight clojure %}

(defn parse-text
  "The text emitted by the bot has this format:
  [[Wikipedia Article Name]] <operation?> <Edit Diff URL> <Edit Comment>
  We want to transform this into a map with the format:
  {:title \"[[Wikipedia Article Name]]\"
   :operation <nil or operation>
   :diff-url <Edit Diff URL>
   :comment <Edit Comment>}"
  [s]
  (let [pattern #"(?x)(\[\[[^\]]*\]\])   #Match and capture the article name.
                  .* #Chomp
                  (https://[^\ ]+ ) #Match edit url
                  \s+(.*) # Match comment"]
    (-> (re-seq pattern s)
        first
        rest
        vec)))

(defn log-result [x]
  (println x)
  x)

(defn tx-process-irc-data []
  (let [get-keys (map #(select-keys % [:command :nick :target :text]))
        filter-nick (filter #(= (:nick %) "rc-pmtpa"))
        ;; Strip out mIRC colour codes. See http://www.mirc.com/colors.html
        strip-colours (map #(update %1 :text strip-colour-codes))
        parse-text (map #(update %1 :text parse-text))
        add-parsed-values (map (fn [event]
                                 (let [{:keys [text]} event
                                       [topic edit-url comment] text]
                                   (assoc event :topic topic
                                          :edit-url edit-url
                                          :comment comment))))
        log-result (map log-result)]
    (comp get-keys filter-nick strip-colours parse-text add-parsed-values log-result)))
    


{% endhighlight %}

Now we filter out keys, filter the nickname, strip mIRC colours out (those ugly caret C's spattered
all over our text), parse the text into some metadata and finally add that metadata to our map.

Oh, also we log the result out to console.

# Key insight #3: Transducers can have side-effects.

We want to be careful with this, because it's very easy to jam up a pipeline by depending on external
calls, connections, reads/writes to disk, etc. It also requires us to put more error handling in our
pipeline.

It is also potentially incredibly powerful. Here's an idea I'm playing with. I haven't used this in production,
use at your own risk, but I think it's interesting. Also it might not compile, consider it a sketch.

{% highlight clojure %}

(defn mark-time [label]
  (map (assoc % label (time/now))))
  
(defn tx-transform [env]
    (let [tx (case env :dev (comp (mark-time :start) do-other-stuff (mark-time :end))
                       :prod do-other-stuff)]
                       tx))

{% endhighlight %}

This allows us optional logging or metrics middleware that we can either accumulate in our data as we pass
it through or report to some external source. With stateful transducers there are some even fancier possible
measurements, like throughput or frequency counts.

# Parallelism

Parallelism, done right, is always just an optimization. Only some kinds of problems are parallel but if we can
phrase a problem as a parallel problem, a good abstraction should be able to do the rest. Luckily, we already have
some of these abstractions. ```map``` and ```filter``` are both parallelizable because they guarantee that *each
 piece of data will be independent from other pieces of data*. As a result, anything built on map and filter
 transducers should be something we can parallelize easily.
 
 We can see this in ```pmap```, which works exactly like a map but with multiple worker threads (I don't really like
 writing ```pmaps``` because I've never taken the time to learn the gotchas and tradeoffs).
 
 Pipelines handle this brilliantly, taking an argument for the number of workers and doing the rest.
 
# Some pipelines.
 
 Let's looks at a pipeline definition.
 
 {% highlight clojure %}
 
 (a/pipeline 
   ;; The number of workers to run. Generally twice the number of cores on your machine, but worth measuring.
   8
   ;; The channel we pipe our transducers to.
   to-file> 
   ;; Our transducer
   (irc-data/tx-process-irc-data) 
   ;; The channel our pipeline consumes from.
   publish< 
   ;; If true, close our to-file> channel when publish< closes. True by default.
   true 
   ;; Exception handler, takes the exception, places any non-nil results on the next channel.
   (fn [e] (log/error (.getMessage e))
       e))
       
  {% endhighlight %}

So there's a lot here, but some really nice properties. If we want to run the same code we're running
on a c3.8extralarge up in AWS, we need but change a number to take advantage of 32 cores.

We can choose whether to close channels or not. If we have multiple pipelines feeding the same
channel (I've never done this but it's certainly feasible), we can do that without a single pipeline taking
down the rest of the system.

The exception handler lets us do all sorts of interesting error handling. If we want to ```System/exit```,
we can do that. If we want to close a consumer channel or send a restart signal or any manner of things,
we can do that. We can just log the error and the consumer of the pipeline won't ever see the data we've
ignored.

# Abort, Retry, Ignore, Escalate

As one of my areas of interests is error handling in this sort of system, there's a small thread to pick up.
Generally, we can respond to error conditions one of four ways. We can bail on the entire computation, try again,
ignore the erroneous input, or decide that this is above our paygrade and send it to something with more context
to handle.

The last option is something I'm still wrestling with in ```core.async```. If our pipeline only contains pure
functions, retry isn't really an option without altering the input data (something I don't generally think we should
do). This gives us only two real options: We can kill the pipeline or we can ignore the input.

# Kill in Dev, Log in Production

While working on this IRC pipeline, I started it up and left it overnight, happy path only, to see if it would crash.
If this system had any need for stronger guarantees it would be a good idea to try to subject it to connection problems
and bad data as well, but failing early and often is likely the best way to find instances of errors.

Once our service is out in the world, there is no tool better to determine what went wrong than logs. Lots of logs.
Any event that happens in a system warrants at least a small log, and log levels are our friends. If an info log 
is too chatty in production (as I've recently discovered in a bug in one of our services), it's a good bet that the
service may be doing something much more often than is necessary.

# Finally, Ordering

One of the nicest properties of pipelines is that they preserve event ordering. This allows us to safely implement a 
number of concurrent systems that we can easily reason about. This leads us to one of the weaknesses of a pipeline,
however.

### *Pipelines work best when their work is roughly uniform in size*

Luckily for our wikipedia edits example, this holds true. Consider a reporting pipeline that pulls a wide variety
of events. Some of these are small maps, some of these are large data blobs.

Let's say that a small map M takes roughly 3ms to process and a large data blob B takes roughly 300ms to process.

Due to ordering constraints, a batch of events will be constrained to the slowest event.

Let's do a little bit of greasy arithmetic:

Suppose that B represents 10% of our input events and M represents 90% of our input events.

We have a number of workers, W, that process pipeline events.

The maximum throughput of our pipeline will be, roughly,

{% highlight clojure %}

(defn throughput 
    "Gives the number of events per ms given that:
     w is the number of workers.
     b is the time taken (in ms) to process a blob
     pb is the probability that a blob will occurred
     m is the time taken (in ms) to process a small event
     pm is the probability that an m will occur."
    [w b pb m pm]
    (let [expected-value-of-b (* b pb)
      expected-value-of-m (* m pm)
      total-expected-value (+ expected-value-of-b expected-value-of-m)]
    ;; Gives the number of events per ms.
    (/ w total-expected-value)))
    
    ;; For our example, we get 0.14035087719298245 events per ms.

{% endhighlight %}

# Exercises left to the reader.

This post is much longer than I meant it to be, but I hope it's helpful. Here are some
things to think about:

How do ```pipeline```, ```pipeline-blocking```, and ```pipeline-async``` differ? In what
situations might we use each?

How would we generalize that throughput calculation for any number of events of any size?

How would we check our throughput model to see if it's true?

How would we write a ```pool``` function that works exactly as pipeline does but relaxes ordering?
